---
issue: # TBD – file in praxis repo and update
status: proposed
repos:
  - praxis
authors:
  - alexsnaps
graduation_criteria:
  - Service trait shape (health / stop) agreed by stakeholders
  - Startup deadline scope (per-service vs. global default) agreed
  - Degraded-state surface to filter callers decided (`ServiceHandle::health()` accessor, domain-method errors, or none)
  - Downcast mechanism from `Arc<dyn ManagedService>` to `Arc<dyn DomainTrait>` agreed
  - GC mechanism and timing decided (ref-count drop vs. scheduled sweep after reload)
  - Relation to ENH-1042 (filter-task-supervisor) and ENH-099 (state management) documented
  - `ServiceFactory` registration API agreed, including whether the bootstrap runtime outlives startup (service task runtime) or is torn down after all `create` calls finish
experimental_exempt: true
experimental_exempt_reason: "Core infrastructure — touches filter construction, pipeline build, and server bootstrap paths"
related:
  - 01042
  - 00099
stakeholders:
  - shaneutt
  - leseb
  - rikatz
---

# Service Registry

## What?

Filters that need long-lived, shared resources — database
connection pools, HTTP client pools, external SDK handles —
have no supported way to hold them today. Owning the resource
directly inside the filter struct means a hot-reload destroys
and recreates it; using a global static means no lifecycle and
no operator visibility.

This proposal introduces a `Service` trait for long-lived,
shared resources and a `ServiceRegistry` that manages their
lifecycle independently from any individual filter chain.

A `Service`:

- Is declared by name in a top-level `services:` config
  section, constructed once at server start, and not
  recreated on pipeline hot-reload.
- Survives filter-chain hot-reload as long as at least one
  active pipeline still holds a handle to it.
- Starts up inside `ServiceFactory::create`. The registry runs
  all `create` calls on a dedicated bootstrap runtime it owns
  (a private OS thread + `current_thread` Tokio runtime), so
  service authors can write async startup without managing their
  own runtime. The registry applies a configurable deadline;
  timeout or error is fatal. Filters never see a service that
  hasn't finished starting.
- Can enter a `Degraded` state after startup — the natural
  encapsulation point for a circuit breaker or a partial
  backend failure.
- Is stopped and its resources released when no active filter
  holds a handle to it (garbage collection).

The `ServiceRegistry` maps service names to live instances,
tracks which filter pipelines reference each, and drives the
start / stop lifecycle.

### Goals

- Provide a `Service` trait with a synchronous `health` query
  (`Ready` / `Degraded`) and an async `stop`.
- Provide a `ServiceRegistry` that is fully initialized from a
  top-level `services:` config section before any pipeline is
  built; each named service is started once and survives
  hot-reload as long as it remains in config.
- Expose `ServiceRegistry::get(name)` as the only filter-facing
  API: synchronous, no factory argument, no async — the registry
  is already running by the time any filter is constructed.
- Let a filter hold a `ServiceHandle<S>` so a hot reload does
  not destroy a running connection pool or break a circuit-breaker
  state machine that spans many requests.
- Enable circuit-breaker logic to live inside a `Service`
  implementation rather than inside a per-request filter, so
  the breaker accumulates history across requests and reloads.
- Give the admin surface enough information to report which
  services are running and whether they are `Ready` or `Degraded`.

### Non-Goals

- Replacing filter-task-supervisor (ENH-1042). A service may
  use pipeline-task infrastructure internally; that is
  composition. Services outlive individual pipelines.
- Replacing Pingora `BackgroundService` or `server.add_service`.
  Those are process-lifetime listeners and admin endpoints.
- Defining the storage backend API. ENH-099 explicitly reserved
  alignment with "the shared service definition once it lands";
  that reservation is this proposal. ENH-099 owns the storage
  abstraction; this proposal owns the lifecycle layer beneath it.
- Specifying any concrete connection pool or HTTP client
  implementation.
- Multi-tenant isolation or per-route service scoping.

## Why?

### Motivation

An `HttpFilter` that needs a database connection pool has two
bad options today:

**Own the pool inside the filter struct.** The pool is created
in `from_config`, lives inside the filter, and is dropped when
the pipeline is replaced on hot-reload. A config change that
touches one unrelated filter forces every other filter to
reconstruct — including destroying and recreating connection
pools. For a Redis-backed rate limiter or a Postgres-backed
session store that flush is expensive and visible to clients.

**Use a global static.** No lifecycle, no GC, no operator
visibility. The pool never stops even when the filter is
removed from every pipeline.

Neither option supports a startup phase. A filter that must
run a schema migration or warm a local cache before serving
traffic has nowhere to block pipeline readiness on completion.

Neither option supports a shared circuit breaker. A filter that
wraps an unreliable external call must reinvent open/half-open/
closed state per filter instance, losing all accumulated history
on every reload.

Neither option gives the operator visibility. There is no way
to ask "which external services is this proxy currently holding
connections to?" or "is the token-validation service degraded?"

The state-management proposal (ENH-099) explicitly deferred
`Backend connectivity fields aligned with the shared service
definition once it lands`. This is that definition.

### User Stories

- As a filter author building a database-backed rate limiter,
  I want a shared connection pool that survives hot-reload so
  that a config change does not flush open connections.
- As a filter author, I want to run schema migrations inside
  `ServiceFactory::create` and have the registry refuse to
  build pipelines until they complete, so the filter is never
  live against a stale schema.
- As a filter author, I want to encapsulate a circuit breaker
  inside my `Service` so that the filter can query health on
  the hot path without managing open/half-open/closed state
  itself.
- As an operator, I want the admin endpoint to report which
  services are running and whether they are `Ready` or
  `Degraded`, without needing to dig into filter-level metrics.
- As a platform engineer doing a hot reload, I want services
  that are no longer referenced by any pipeline to be stopped
  and their resources released without a process restart.

## How?

> **Note:** this is a **lightweight sketch** to anchor
> stakeholder discussion. Shapes are illustrative, not final.
> Open questions are listed at the end.

### Experimental Phase

`experimental_exempt` — this touches filter construction,
pipeline build, and the server bootstrap path, which are not
representable in the experimental repo without significant
forking.

### Design Sketch

#### Two traits: lifecycle vs. domain interface

The design splits the service API across two distinct boundaries:

- **`ManagedService`** — the registry-internal lifecycle trait.
  Only the registry calls these methods. Filter code never sees it.
- **The domain interface** — a user-defined trait (e.g. `trait Pool`,
  `trait TokenCache`) that the concrete service type implements.
  This is what the filter holds and calls through its handle.

```rust
use std::borrow::Cow;
use async_trait::async_trait;

pub enum ServiceHealth {
    /// Service is healthy.
    Ready,
    /// Service is available but impaired (e.g. circuit half-open).
    Degraded { reason: Cow<'static, str> },
}

pub type ServiceError = Box<dyn std::error::Error + Send + Sync>;

/// Registry-internal lifecycle trait.
///
/// Filter code never holds a reference to `dyn ManagedService`.
/// The registry uses this to monitor health and drive shutdown.
#[async_trait]
pub trait ManagedService: Send + Sync + 'static {
    /// Unique type name (e.g. `"pg_pool"`). Used for admin surfaces.
    fn service_type(&self) -> &'static str;

    /// Current health. Called by the registry for admin/observability.
    /// Must not block.
    fn health(&self) -> ServiceHealth;

    /// Release resources. Called when the last handle is dropped and
    /// GC runs. Filter code cannot trigger this.
    async fn stop(&self);
}
```

The concrete service type implements both `ManagedService`
(for the registry) and the user-defined domain trait (for
filters). The registry stores `Arc<dyn ManagedService>` for
lifecycle operations; `ServiceHandle<S>` holds only the domain
type `S` — `stop()`, `service_type()`, and `health()` are not
reachable through it.

#### `ServiceHandle<S>`

`S` is constrained to the service's domain interface — a
user-defined trait that contains no lifecycle methods. The
handle `Deref`s to `S`; nothing on the deref path touches
`ManagedService`.

```rust
use std::{ops::Deref, sync::Arc};

/// A filter-held reference to a running service's domain interface.
///
/// `S` is the concrete service type or a domain trait (e.g.
/// `dyn Pool`). It does not include `ManagedService` methods.
/// Cloning shares the reference; the registry's GC token is
/// internal and not accessible to the holder.
pub struct ServiceHandle<S: ?Sized> {
    inner: Arc<S>,
    // Internal GC token — invisible to the filter.
}

impl<S: ?Sized> Clone for ServiceHandle<S> { /* Arc::clone on inner */ }
impl<S: ?Sized> Deref for ServiceHandle<S> {
    type Target = S;
    fn deref(&self) -> &S { &self.inner }
}
```

#### `ServiceFactory` and registry bootstrap

Services are declared in a top-level `services:` config section.
Each named entry names a service type; the registry resolves the
type to a registered factory, calls it with the entry's config,
and starts the resulting service before any pipeline is built.

```rust
/// Constructs and starts a service instance from YAML config.
///
/// `create` is async so implementations can open connections, run
/// migrations, and retry internally without managing their own Tokio
/// runtime. The registry runs all `create` calls on a dedicated
/// bootstrap runtime it owns (OS thread + `current_thread` Tokio
/// runtime, mirroring the `spawn_on_dedicated_runtime` pattern
/// already used in the praxis server for health checks and
/// housekeeping). The registry wraps each call in
/// `tokio::time::timeout` from the per-service (or global)
/// `startup_deadline`; error or timeout is fatal.
///
/// There is no Tokio runtime in scope when the praxis server calls
/// `ServiceRegistry::build` — the Pingora runtime only starts after
/// all pipelines are built. Service authors must not assume an ambient
/// runtime; the registry provides one exclusively for `create`.
///
/// `Service` must implement `ManagedService` (for the registry) and
/// whatever domain trait filters will hold through `ServiceHandle`.
#[async_trait]
pub trait ServiceFactory: Send + Sync + 'static {
    type Service: ManagedService;
    async fn create(&self, config: &serde_yaml::Value) -> Result<Self::Service, ServiceError>;
}

/// Registry of running services. One instance per server.
pub struct ServiceRegistry { /* opaque */ }

impl ServiceRegistry {
    /// Build the registry from the `services:` config section.
    ///
    /// Spawns a dedicated OS thread with a `current_thread` Tokio
    /// runtime, runs each factory's `create` under the configured
    /// deadline, then joins the thread. Returns `Err` if any service
    /// fails to construct or times out.
    ///
    /// Called once during server bootstrap (or on reload when the
    /// services section changes), before any pipeline is built and
    /// before the Pingora runtime starts.
    pub fn build(
        factories: &ServiceFactoryRegistry,
        config: &serde_yaml::Value,
    ) -> Result<Self, ServiceError>;

    /// Look up a running service's domain interface by name.
    ///
    /// `S` is the domain type (concrete or trait object) — it must
    /// not be `ManagedService`. Returns `Err` if the name is unknown
    /// or the stored concrete type cannot be downcast to `S`.
    /// Synchronous; the registry is fully started before any filter
    /// is constructed.
    ///
    /// How the registry maps from `Arc<dyn ManagedService>` to
    /// `Arc<S>` is TBD (see open questions).
    pub fn get<S: 'static>(&self, name: &str) -> Result<ServiceHandle<S>, ServiceError>;

    /// Snapshot of {service name → filter/pipeline IDs holding a handle}.
    pub fn usage_snapshot(&self) -> HashMap<String, Vec<String>>;

    /// Stop and remove services with no live handles.
    ///
    /// Returns the names of services that were stopped.
    /// Called after a reload drops old pipelines.
    pub async fn gc(&self) -> Vec<String>;
}
```

#### Filter-side sketch

```rust
// The domain trait the filter cares about — no lifecycle methods.
trait Pool: Send + Sync {
    async fn acquire(&self) -> Result<PooledConnection, PoolError>;
}

// The concrete type implements both the registry lifecycle and the
// filter-visible domain trait.
struct PgPool { /* ... */ }
impl ManagedService for PgPool { /* service_type, health, stop */ }
impl Pool for PgPool { /* acquire */ }

// The filter holds only the domain handle.
struct DbRateLimitFilter {
    pool: ServiceHandle<dyn Pool>,
}

impl DbRateLimitFilter {
    fn from_config(
        config: &serde_yaml::Value,
        registry: &ServiceRegistry,
    ) -> Result<Self, FilterError> {
        // get::<dyn Pool> — no lifecycle methods visible through this handle.
        let pool = registry.get::<dyn Pool>("pg_pool:main")?;
        Ok(Self { pool })
    }
}

// On the hot path the filter calls domain methods only.
// It cannot call stop(), service_type(), or health() through the handle.
let conn = self.pool.acquire().await?;
```

#### Config sketch

```yaml
services:
  pg_pool:main:
    type: pg_pool
    startup_deadline: 30s   # optional; falls back to a global default
    host: db.internal
    port: 5432
    max_connections: 50

filters:
  - filter: db_rate_limit
    service: pg_pool:main
```

### Open Questions

1. **Startup deadline scope.** Should `startup_deadline` be
   per-service (in each entry's config, as sketched), a single
   global default, or both (per-service overrides a global
   default)? A global default avoids repeating the same timeout
   on every entry; per-service overrides are useful when one
   service (e.g. a slow migration) legitimately needs more time.

2. **Degraded-state surface.** `health()` is on `ManagedService`
   (registry-only). A filter that wants to fail-closed when its
   service degrades needs an alternative. See open question 5.

3. **Registry scope.** Is the registry a process-level singleton
   or per-server-instance? The extproc embedding model builds a
   Praxis filter pipeline inside a host process; it may need its
   own registry or may share the embedding server's. Also: does
   a service registered in `praxis-core` know which server scope
   it belongs to?

4. **Downcast mechanism.** The registry stores `Arc<dyn ManagedService>`.
   `get::<dyn Pool>(name)` must obtain `Arc<dyn Pool>` from it.
   Rust has no built-in `Arc<dyn A>` → `Arc<dyn B>` coercion even
   when the concrete type implements both. Options:
   - `ManagedService` requires `fn as_any(self: Arc<Self>) -> Arc<dyn Any + Send + Sync>`
     and the registry double-stores the erased `Arc` for downcast.
   - The factory registers both the `Arc<dyn ManagedService>` and
     a type-erased `Arc<dyn Any>` at construction time.
   - `get` is keyed on both name and `TypeId`; the factory registers
     multiple typed `Arc`s at construction.
   Whichever approach is chosen, a type mismatch at `get` call time
   is a config error that should be caught at pipeline build, not at
   request time.

5. **`health()` visibility to filters.** Currently `health()` is on
   `ManagedService` (registry-only). A filter that wants to fail-closed
   when its service is `Degraded` has no standard way to check health.
   Options: expose `health()` as a method on `ServiceHandle<S>`
   directly (independent of `S`); require service authors to include a
   health query on their domain trait; or leave it to the service's
   domain methods to return degraded errors inline.

6. **Relation to filter-task-supervisor (ENH-1042).** A service
   that drives a background health-check loop needs a Tokio task
   that outlives any individual pipeline. The bootstrap runtime
   is torn down after all `create` calls finish, so long-running
   tasks started there would die with it. Does the registry keep
   the bootstrap runtime alive as the service's task runtime for
   its lifetime, or does the service spawn its own thread + runtime
   inside `create` for its background loops? The answer affects
   shutdown ordering and whether services share or own their
   runtime.

7. **GC timing.** GC is triggered after a reload drops old
   pipelines. Is it the server's responsibility to call
   `registry.gc()` explicitly, or does the registry detect
   handle counts reaching zero and stop services automatically?
   Automatic GC on zero-handles risks stopping a service during
   a rolling reload that is about to re-reference it.

8. **Storage backend alignment (ENH-099).** Should a `KvStore`
   or `SqlStore` backend be registered as a `Service`? ENH-099
   reserved a hook for this. If yes, storage backends gain the
   same startup, health, and GC lifecycle as other services —
   at the cost of requiring a `services:` entry for every
   storage backend.
