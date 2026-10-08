---
issue: https://github.com/praxis-proxy/praxis/issues/1375
status: proposed
repos:
  - praxis
authors:
  - alexsnaps
graduation_criteria:
  - `ManagedService` trait shape (health / stop) agreed by stakeholders
  - `PipelineBuildContext` design agreed (unified replacement for `RegisteredFilterFactory` variants)
  - Service factory registration integrated into `ServerComposition`
  - Reload sequencing (service config change → filter chain rebuild) implemented and tested
  - Storage backend (ENH-099) uses `Service` lifecycle as its connectivity layer
  - Relation to ENH-1042 (background task runtime scope) documented
experimental_exempt: true
experimental_exempt_reason: "Core infrastructure — touches filter construction, pipeline build, server bootstrap, and ServerComposition"
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

This proposal introduces a `ManagedService` trait for long-lived,
shared resources and a `ServiceRegistry` that manages their
lifecycle independently from any individual filter chain.

A service:

- Is declared by name in a top-level `services:` config section,
  constructed once at server start and not recreated on
  filter-chain hot-reload.
- Survives filter-chain hot-reload as long as it remains in
  the `services:` config section unchanged.
- Starts up inside `ServiceFactory::create`, which is async.
  The registry runs all `create` calls on a dedicated bootstrap
  runtime (private OS thread + `current_thread` Tokio runtime)
  it owns and keeps alive. A configurable deadline applies per
  service; timeout or error is fatal. Filters never see a service
  that has not finished starting.
- Can enter a `Degraded` state after startup — the natural
  encapsulation point for a circuit breaker or a partial backend
  failure.
- Is stopped when its config changes on reload (see reload
  semantics below) or when it is removed from the `services:`
  section and no active pipeline holds a handle to it.

The `ServiceRegistry` maps service names to live instances,
tracks which filter pipelines reference each, and drives the
start / stop lifecycle.

### Goals

- Provide a `ManagedService` trait with synchronous `health`
  (`Ready` / `Degraded`) and async `stop`, used exclusively by
  the registry. Filter code never interacts with `ManagedService`
  directly.
- Provide a `ServiceRegistry` fully initialized from the
  `services:` config section before any pipeline is built; expose
  `ServiceRegistry::get::<ConcreteType>(name)` as the only
  filter-facing API — synchronous, no factory, no async.
- Give storage backends (ENH-099) the startup, health, and GC
  lifecycle they need; this is one of the primary motivations.
- Let a filter hold a `ServiceHandle<T>` so a hot-reload that
  does not change the `services:` section does not destroy a
  running connection pool or reset a circuit-breaker state
  machine.
- Enable circuit-breaker logic to live inside a service
  implementation rather than inside a per-request filter, so
  the breaker accumulates history across requests and reloads.
- Give the admin surface enough information to report which
  services are running and whether they are `Ready` or `Degraded`.
- Extend `ServerComposition` and replace the accumulating
  `RegisteredFilterFactory` variants with a unified
  `PipelineBuildContext` that carries `FilterRegistry`,
  `ServiceRegistry`, and future build-time dependencies.

### Non-Goals

- Replacing filter-task-supervisor (ENH-1042). A service may
  use the dedicated background runtime internally; that is
  composition, not replacement.
- Replacing Pingora `BackgroundService` or `server.add_service`.
  Those are process-lifetime listeners and admin endpoints.
- Defining the storage backend API. ENH-099 owns that; this
  proposal owns the lifecycle layer beneath it.
- Shipping any concrete service implementation (connection pool,
  HTTP client, etc.). Core provides the infrastructure; extension
  crates provide implementations.
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
traffic has nowhere to block pipeline readiness until that
work completes.

Neither option supports a shared circuit breaker. A filter that
wraps an unreliable external call must reinvent open/half-open/
closed state per filter instance, losing all accumulated history
on every reload.

Neither option gives the operator visibility into what external
connections the proxy is holding, or whether any of them are
impaired.

Storage backends are the canonical first consumer. ENH-099
explicitly deferred *"Backend connectivity fields aligned with
the shared service definition once it lands"*; this is that
definition. A Postgres-backed `SqlStore` that runs schema
migrations before the proxy accepts traffic, and surfaces a
`Degraded` signal when the database is unreachable, is exactly
the use case this proposal targets.

> [!NOTE]
> See Architecture Decision Record 8.

### User Stories

- As a filter author building a database-backed rate limiter,
  I want a shared connection pool that survives filter-chain
  hot-reload so that a config change does not flush open
  connections.
- As a filter author, I want to run schema migrations inside
  `ServiceFactory::create` and have the registry refuse to
  build pipelines until they complete, so the filter is never
  live against a stale schema.
- As a filter author, I want to encapsulate a circuit breaker
  inside my service so the breaker accumulates state across
  requests and reloads without each filter instance reinventing
  open/half-open/closed tracking.
- As an operator, I want the admin endpoint to report which
  services are running and whether they are `Ready` or
  `Degraded`, without needing to inspect filter-level metrics.
- As a platform engineer doing a hot reload, I want services
  that are no longer referenced by any pipeline to be stopped
  and their resources released without a process restart.

## How?

> **Note:** this is a **lightweight sketch** to anchor
> stakeholder discussion. Shapes are illustrative, not final.

### Experimental Phase

`experimental_exempt` — this touches filter construction,
pipeline build, `ServerComposition`, and the server bootstrap
path, which are not representable in the experimental repo
without significant forking.

### Design Sketch

#### Two-trait split: `ManagedService` vs. domain interface

The design separates two concerns:

- **`ManagedService`** — the registry-internal lifecycle trait.
  Only the registry calls `health()` and `stop()`. Filter code
  never holds `dyn ManagedService`.
- **The concrete service type** — what the filter receives via
  `ServiceHandle<T>`. The type implements both `ManagedService`
  (for the registry) and its own domain methods (for filters).
  The registry holds `Arc<dyn ManagedService>`; the handle holds
  `Arc<T>` obtained by downcasting `Arc<dyn Any + Send + Sync>`.

> [!IMPORTANT]
> Since `ServiceHandle<T>` derefs to `T`, and `T` implements
> `ManagedService`, lifecycle methods are technically reachable
> through the handle. Enforcement is by convention and
> documentation, not the type system. A future extension could
> use trait-object coercions (`Arc<dyn Pool>`) to eliminate this
> gap, but requires the factory to register coercions upfront and
> adds significant complexity; it is noted here as a future option
> (see Open Question 3).

```rust
use std::borrow::Cow;
use async_trait::async_trait;

pub enum ServiceHealth {
    Ready,
    Degraded { reason: Cow<'static, str> },
}

pub type ServiceError = Box<dyn std::error::Error + Send + Sync>;

/// Registry-internal lifecycle trait. Filter code never holds
/// a reference to `dyn ManagedService`.
#[async_trait]
pub trait ManagedService: Send + Sync + 'static {
    fn service_type(&self) -> &'static str;

    /// Current health. Called by the registry for admin/observability.
    /// Must not block.
    fn health(&self) -> ServiceHealth;

    /// Release resources. Called by the registry at GC time.
    async fn stop(&self);
}
```

> [!NOTE]
> See Architecture Decision Record 3 and 4.

#### `ServiceHandle<T>`

```rust
use std::{ops::Deref, sync::Arc};

/// A filter-held reference to a running service.
///
/// `T` is the concrete service type. `stop()`, `health()`, and
/// `service_type()` come from `ManagedService`, which `T` also
/// implements — but filter code should not call lifecycle methods
/// through this handle.
pub struct ServiceHandle<T> {
    inner: Arc<T>,
    // Internal GC token (atomic ref-count in the registry entry).
    // Not accessible to the holder.
}

impl<T> Clone for ServiceHandle<T> { /* Arc::clone on inner */ }
impl<T> Deref for ServiceHandle<T> {
    type Target = T;
    fn deref(&self) -> &T { &self.inner }
}
```

> [!NOTE]
> See Architecture Decision Record 3 and 4.

#### `ServiceFactory` and registry bootstrap

```rust
/// Constructs and starts a service instance from YAML config.
///
/// `create` is async; the registry runs it on a dedicated bootstrap
/// runtime (OS thread + `current_thread` Tokio runtime, mirroring
/// `spawn_on_dedicated_runtime` already used for health checks and
/// housekeeping). That runtime is kept alive after bootstrap to serve
/// as the task runtime for service background loops (health-check
/// ticks, circuit-breaker maintenance). The registry wraps each call
/// in `tokio::time::timeout` using the per-service `startup_deadline`
/// or the global default; error or timeout is fatal.
///
/// There is no Tokio runtime in scope when the praxis server calls
/// `ServiceRegistry::build` — the Pingora runtime only starts after
/// all pipelines are built.
#[async_trait]
pub trait ServiceFactory: Send + Sync + 'static {
    type Service: ManagedService;
    async fn create(&self, config: &serde_yaml::Value) -> Result<Self::Service, ServiceError>;
}

pub struct ServiceRegistry { /* opaque */ }

impl ServiceRegistry {
    /// Build from the `services:` config section.
    ///
    /// Spawns the dedicated bootstrap/task runtime, runs each factory's
    /// `create` under the configured deadline, then returns. Returns
    /// `Err` if any service fails to construct or times out.
    ///
    /// Called before any pipeline is built and before the Pingora
    /// runtime starts.
    pub fn build(
        factories: &ServiceFactoryRegistry,
        config: &serde_yaml::Value,
    ) -> Result<Self, ServiceError>;

    /// Look up a running service by name, downcasting to `T`.
    ///
    /// Synchronous. The registry is fully started before any filter
    /// is constructed. Returns `Err` if the name is unknown or the
    /// stored type is not `T`.
    pub fn get<T: ManagedService>(&self, name: &str) -> Result<ServiceHandle<T>, ServiceError>;

    /// Snapshot of {service name → filter/pipeline IDs with live handles}.
    pub fn usage_snapshot(&self) -> HashMap<String, Vec<String>>;

    /// Stop and remove services with no live handles.
    ///
    /// Called explicitly by the server after old pipelines are dropped
    /// at the end of a reload. Never triggered automatically by handle
    /// drop — see ADR-1.
    pub fn gc(&self) -> Vec<String>; // sync: stop() is async internally
}
```

> [!NOTE]
> See Architecture Decision Record 1 (explicit GC), 2 (bootstrap runtime), 3 (concrete-type `get`), and 5 (startup deadline).

#### Filter-side sketch

```rust
// The concrete type implements both lifecycle (for the registry)
// and domain logic (for filters).
struct PgPool { /* ... */ }

impl ManagedService for PgPool {
    fn service_type(&self) -> &'static str { "pg_pool" }
    fn health(&self) -> ServiceHealth { /* read internal circuit state */ }
    async fn stop(&self) { /* close connections */ }
}

impl PgPool {
    pub async fn acquire(&self) -> Result<PooledConnection, PoolError> { /* ... */ }
}

// The filter holds ServiceHandle<PgPool>. It calls domain methods only.
struct DbRateLimitFilter {
    pool: ServiceHandle<PgPool>,
}

impl DbRateLimitFilter {
    // from_config receives the registry via PipelineBuildContext.
    fn from_config(
        config: &serde_yaml::Value,
        ctx: &PipelineBuildContext<'_>,
    ) -> Result<Self, FilterError> {
        let pool = ctx.services().get::<PgPool>("pg_pool:main")?;
        Ok(Self { pool })
    }
}

// Hot path: domain call only.
let conn = self.pool.acquire().await?;

// Checking health (accessible but conventionally registry-only):
// self.pool.health() — callable but filter authors should not use it.
```

> [!NOTE]
> See Architecture Decision Record 3 (concrete-type get), 4 (health accessibility), and 9 (PipelineBuildContext).

#### Config sketch

```yaml
services:
  startup_deadline: 30s        # global per-service default
  instances:
    pg_pool:main:
      type: pg_pool
      startup_deadline: 60s    # per-service override (slow migration)
      host: db.internal
      port: 5432
      max_connections: 50

filters:
  - filter: db_rate_limit
    service: pg_pool:main
```

> [!NOTE]
> See Architecture Decision Record 5 (startup deadline).

#### Reload semantics

A service survives reload if and only if its entry in `services:`
is unchanged. When the `services:` section changes:

- **Entry unchanged**: service keeps running; new filter instances
  receive handles to the same `Arc<T>`.
- **Entry config changed**: the service is stopped and restarted.
  All filter chains that reference it are also rebuilt (they
  receive handles to the new instance). Old service is GC'd after
  old pipelines drop.
- **Entry removed**: service is GC'd after the last referencing
  pipeline drops its handles (via explicit `gc()` after reload).
- **Entry added**: service is constructed and started before
  pipelines that reference it are built.

Filter authors can assume the service's config is stable for the
lifetime of their handle.

> [!NOTE]
> See Architecture Decision Record 1 (explicit GC) and 7 (service config change triggers filter chain rebuild).

#### `ServerComposition` and `PipelineBuildContext`

Service factory registration is part of `ServerComposition`,
parallel to filter factory registration:

```rust
ServerComposition::new()
    .with_filter_registry(filter_registry)
    .with_service_factories(service_factory_registry)
```

The proliferating `RegisteredFilterFactory` variants (`Standard`,
`HttpWithRegistry`, `ChainBinding`, `Policy`, …) are replaced by
a single `PipelineBuildContext` passed to all context-aware
factories:

```rust
// One registration path for context-aware filters (replaces all variants):
registry.register_with_context("my_filter", |config, ctx| {
    let pool = ctx.services().get::<PgPool>("pg_pool:main")?;
    Ok(Box::new(MyFilter { pool }))
});
```

`PipelineBuildContext` carries `FilterRegistry`, `ServiceRegistry`,
and future build-time dependencies, so adding a new dependency
does not add a new registration variant.

> [!NOTE]
> See Architecture Decision Record 9 (unified PipelineBuildContext) and 10 (service factory registration via ServerComposition).

### Architecture Decision Record

> These decisions were reached in initial stakeholder review.
> They **can be revisited** before the graduation criteria are met.

1. **Explicit GC.** `gc()` is called explicitly by the server
   after old pipelines are dropped, not triggered by
   `ServiceHandle` drop. Arc counts can transiently hit zero
   during the window between old-pipeline drop and new-pipeline
   acquisition, making automatic GC unsafe.

2. **Bootstrap runtime kept alive.** The dedicated OS thread +
   `current_thread` Tokio runtime used for `create` calls is
   retained as the service task runtime for the registry's
   lifetime. Services can `tokio::spawn` background loops inside
   `create`; the registry cancels per-service tokens and calls
   `stop()` at GC time.

   > [!IMPORTANT]
   > A single dedicated thread is accepted for now; flagged for
   > potential revisit if contention or scheduling latency becomes
   > a concern.

3. **Concrete type for `get()`.** `registry.get::<PgPool>(name)`
   downcasts via `Arc<dyn Any + Send + Sync>`. Lifecycle method
   accessibility through `Deref` is accepted; enforcement is by
   convention.

   > [!IMPORTANT]
   > A future extension using trait-object coercions (`Arc<dyn Pool>`)
   > would enforce the boundary in the type system but requires
   > factories to register coercions upfront (see Open Question 3).

4. **`health()` accessible via handle.** Since `ServiceHandle<T>`
   derefs to `T` and `T: ManagedService`, `handle.health()` is
   callable from filter code. This is treated as acceptable for
   the circuit-breaker use case and not restricted.

5. **Startup deadline: global default + per-service override.**
   A global `startup_deadline` under `services:` sets the default
   for all services; per-instance entries can override.

   > [!IMPORTANT]
   > A total bootstrap deadline (hard cap on the entire `build()`
   > call across all services) is deferred from this proposal
   > (see Open Question 2).

6. **Per-server-instance registry.** `ServiceRegistry` is owned
   by `ServerState`, parallel to `FilterRegistry`. One per
   `try_run_server_with_composition` call; extproc embeddings get
   their own naturally.

7. **Service config change triggers filter chain rebuild.** If a
   service's config changes on reload, the service is stopped and
   restarted, and all filter chains that reference it are rebuilt.
   Filter authors can assume a stable service config for the
   lifetime of their handle.

8. **Storage backends are a primary motivation.** A `KvStore` or
   `SqlStore` backend (ENH-099) is a `Service`: startup runs
   migrations, `health()` surfaces database reachability, GC
   releases the pool when no filter needs it. ENH-099's backend
   connectivity fields must use this lifecycle layer.

9. **Unified `PipelineBuildContext`.** Replaces the accumulating
   `RegisteredFilterFactory` variants. All build-time dependencies
   (`FilterRegistry`, `ServiceRegistry`, future additions) are
   carried through a single context, with one "simple" and one
   "context-aware" filter registration path.

10. **Service factory registration via `ServerComposition`.** Core
    ships no concrete service types. Extension crates (e.g.
    praxis-ai) register their factories through the same
    composition API used for custom filters.

### Open Questions

> [!IMPORTANT]
> **Background task runtime scope (relation to ENH-1042).** The
> registry's dedicated runtime is kept alive for service background
> loops. ENH-1042's pipeline task supervisor is a separate,
> shorter-lived scope (pipeline lifetime). How do they compose when
> a service also wants to participate in pipeline events? This needs
> to be documented before ENH-1042 and this proposal are both
> implemented.

> [!IMPORTANT]
> **Total bootstrap deadline (deferred).** A hard cap on the entire
> `ServiceRegistry::build()` call (across all services) is out of
> scope for this proposal but is a natural follow-on — particularly
> useful for Kubernetes startup probes.

> [!IMPORTANT]
> **Trait-object coercion (future).** `ServiceHandle<dyn Pool>`
> would prevent filter code from reaching lifecycle methods through
> `Deref`, at the cost of requiring the factory to register
> coercions at construction time. Left as a future option if the
> convention-based boundary proves insufficient.
