---
issue: https://github.com/praxis-proxy/praxis/issues/99
discussion: https://github.com/praxis-proxy/praxis/issues/99#issuecomment-4411378263
status: proposed
repos:
  - praxis
  - ai
authors:
  - nerdalert
  - shaneutt
  - rikatz
graduation_criteria:
  - State class taxonomy agreed by stakeholders
  - State type hierarchy agreed by stakeholders
  - Scoping model agreed
  - Determine whether Ephemeral and Persistent storage really need to be separate and resolve
  - Storage trait API design reviewed by stakeholders
  - Reference schema for conversation/response storage
  - Resolve whether SQL is a first-class storage backend or a consumer-side concern
  - How? section with requirements and design
supersedes: 00412
related:
  - 00354
stakeholders:
  - shaneutt
  - nerdalert
  - twghu
  - rikatz
  - leseb
origin:
  repo: praxis
  issue: https://github.com/praxis-proxy/praxis/issues/99
  file: 00099_stateful-proxy-state-management.md
merged_from:
  - repo: ai
    issue: https://github.com/praxis-proxy/praxis/issues/412
    file: 00412_storage_layer.md
---

# Stateful Proxy State Management and Storage Layer

## Part 1: State Model

## What?

Praxis should define a project-wide state model before
stateful AI gateway, routing, quota, protocol, and
observability features each grow their own storage
patterns. The model should distinguish request-local
facts, local runtime state, shared hot-path state,
durable business state, configuration state, and
externalized decision state. It must provide optionality
for storage (e.g. local in-memory or disk, vs external kv-store, etc).

The spike for issue 99 found that Praxis already has
several local state patterns: request metadata, local
load-balancer counters, circuit breaker state, health
snapshots, hot-reloaded config snapshots, and filter-owned
maps. The per-IP rate limiter is the clearest existing
example: it uses a local `DashMap` with a `100_000` entry
soft cap and `200_000` entry hard cap to prevent unbounded
growth. These patterns are useful, but they are not enough
for features that must remain correct across multiple
Praxis replicas.

This proposal establishes the direction that stateful
features must use explicit storage classes and typed
domain APIs instead of exposing a raw global key-value API
as the primary filter-facing abstraction. Local in-memory
implementations should be available for tests, development,
demos, explicitly single-replica deployments, and some niche
HA scenarios. The default production guidance will be that
implementations must use an external (e.g. Valkey) storage
backend first, with strict timeouts, TTLs, key conventions,
failure-mode defaults, and bounded metrics labels.

### Goals

- Define state classes used consistently across Praxis
  features and docs.
- Keep request-derived facts in request context and
  `filter_metadata`, not in durable or shared stores.
- Make local runtime state explicitly bounded, observable,
  and documented as local-only unless proven otherwise.
- Add typed state APIs for concrete domains such as rate
  limits, token ledgers, protocol sessions, task ownership,
  policy decision caches, routing snapshots, cache indexes,
  and usage event export.
- Centralize shared backend configuration so filters do
  not create independent Redis clients, key formats,
  timeouts, or failure behavior.
- Require every shared hot-path state operation to define
  timeout behavior, fail-open or fail-closed semantics,
  key schema, TTL, cardinality bounds, and metrics.
- Keep durable billing, audit, certificate, object,
  vector, and control-plane state out of synchronous
  request processing unless a feature explicitly justifies
  that cost.
- Provide a complete experience for the storage and retrieval of metrics per-site, with
  optional and customizable storage solutions. This covers built-in metrics (core proxy,
  built-in filters, etc) but also extensions (custom filters).
- Provide an incremental implementation path that starts
  with documentation and bounded local primitives before
  adding shared backends.
- Enforce that no built-in filters outright fail without internal storage. Filters
  must tolerate lack of storage whenever possible. An explicit opt-out will be
  needed for any filter that is purely bound on external storage solutions.

### State Classes

| State class | Examples | First posture |
| --- | --- | --- |
| Request-local metadata | extracted model, JSON-RPC method, auth subject, tenant, selected route, token estimate, guardrail finding | Keep in request context and `filter_metadata`; use for later filters, logs, metrics, and headers. |
| Connection-local state | client address, TLS facts, client certificate fields, CONNECT tunnel state | Keep in connection/request context; promote only when a feature needs cross-request behavior. |
| Local runtime state | local rate buckets, circuit breakers, endpoint health, local caches, overload level | Bound with max entries, TTLs, eviction policy, reload behavior, and metrics. |
| Shared hot-path state | production quotas, token budgets, session ownership, task ownership, short-lived policy cache | Use Redis/Valkey or an external service with tight timeouts and explicit failure semantics. |
| Durable business state | billing records, usage history, subscriptions, audit logs | Export asynchronously to the owning system; do not make database writes part of normal request admission. |
| Configuration state | routes, clusters, model aliases, policies, endpoint resources, config generation | Treat as validated snapshots from files, xDS, Kubernetes, Gateway API, or another control plane. |
| Externalized decisions | authz, external processing, schedulers, guardrails, model routing providers | Use explicit timeout, retry, circuit breaker, and failure-mode rules at each call site. |

### Feature Drivers

| Area | State pressure | Direction |
| --- | --- | --- |
| Request rate limiting | Counters must remember previous requests and may need fleet-wide enforcement. | Local mode for dev/single replica; shared store for production multi-replica limits. |
| Token quotas and MaaS budgets | Budgets are ledgers over time and must be consistent across replicas. | Shared token ledger for enforcement plus async durable usage export. |
| MCP, A2A, and sticky sessions | Follow-up requests may need the backend that owns a session, task, or context. | Typed session and task stores with TTLs and config-generation awareness. |
| Intelligent routing and llm-d/GIE | Routing depends on fresh backend pressure, readiness, cost, and scheduler signals. | Local validated snapshots with freshness limits and safe fallback behavior. |
| Response and semantic caching | Cache entries, fill locks, vector refs, and purge state have cache-specific semantics. | Separate cache APIs; do not overload quota/session stores as a generic cache. |
| Auth, RBAC, guardrails, and policy | Security decisions may be expensive, cached, audited, or externally delegated. | Request-local decision facts, bounded caches, policy-version keys, and fail-closed defaults for enforcement. |
| Retry, hedging, and failover | Attempts and budgets must avoid loops, amplification, and confusing usage records. | Keep attempt state request-local first; add shared fleet budgets only when needed. |
| Observability | Metrics, logs, traces, and exporters aggregate state and can create cardinality risk. | Keep exporter state bounded and never label on raw request, user, session, task, or prompt values. |

### Non-Goals

- Do not make Praxis a database.
- Do not require every stateful feature to use Valkey.
- Do not expose a raw global mutable key-value API as the
  primary filter-facing abstraction.
- Do not put SQL, Kubernetes API, object-store, vector
  database, or other slow control-plane calls directly on
  every request path.
- Do not treat local maps as multi-replica correct.
- Do not store prompts, API keys, tokens, tool arguments,
  or PII in shared-state keys.
- Do not block MCP, A2A, MaaS, routing scorer, or cache
  work on implementing every state class at once.

## Why?

### Motivation

Praxis is an AI-native proxy framework, not just
a stateless reverse proxy. Planned features need decisions
based on facts produced before, during, and after a
request: parsed request bodies, streaming response usage,
tenant identity, token budgets, backend pressure, protocol
session IDs, task IDs, guardrail findings, retry attempts,
and dynamic configuration versions.

Without a shared state model, each feature will be tempted
to solve this locally. That creates predictable production
risks: unbounded memory maps, duplicated backend clients,
incompatible Redis key formats, hidden fail-open security
paths, hot-reload state loss, high-cardinality metrics, and
test coverage that passes in a single process but fails
across multiple replicas.

Hot reload is a concrete example of the problem. Today,
pipeline reload rebuilds stateful filter instances, so
local rate limiter and circuit breaker state resets while
the process continues serving traffic. That behavior is
acceptable for local protection, but not for
correctness-critical quota, session, or task state.

The spike also found that state should not be collapsed
into one storage layer. Request facts should stay local to
the request. Local runtime state should be fast,
bounded, and disposable. Shared hot-path state should be
reserved for correctness-sensitive counters, ledgers,
sessions, and ownership maps. Durable business records
should flow through async sinks or product systems.
Configuration should remain a validated snapshot from the
config or control plane.

Valkey is a good first shared hot-path backend
because it fits short-lived counters, TTL-backed session
maps, task ownership, policy decision caches, and
correlation maps. It is not the answer for long-term
billing history, large response bodies, vector search,
certificate management, or desired configuration state.

### User Stories

- As a proxy operator, I want local and shared state modes
  to be explicit so that I do not accidentally deploy
  single-replica counters as global production quotas.
- As an SRE, I want all hot-path state calls to have
  bounded timeouts and visible metrics so that a backend
  outage does not create unbounded request latency.
- As a security engineer, I want auth, policy, quota, and
  guardrail state failures to fail closed by default so
  that backend errors do not bypass enforcement.
- As a platform engineer, I want one Redis/Valkey backend
  configuration with shared connection, TLS, auth, timeout,
  and metric behavior so that every filter does not create
  a different operational surface.
- As a Praxis developer, I want typed state traits for
  rate limits, token ledgers, sessions, task ownership, and
  usage events so that filters do not hand-roll storage and
  key semantics.
- As an AI gateway operator, I want token counts and usage
  facts to flow from request and response processing into
  quota enforcement and billing export without turning
  request metadata into the ledger of record.
- As an SRE, I want storage-related degradation to be
  monitored with configurable performance guards so that
  Praxis can detect trends, alert on threshold breaches,
  and support graceful degradation with recovery rather
  than silent latency creep or hard failures.

> **Note:** Requirements and design are in the How?
> section below.

## Part 2: Unified State Interface and Storage Backends

This part defines the concrete interface beneath the state
model above: the traits filters use, the configuration
schema, scoping rules, and the pluggable backends. It
merges the Unified State Interface design (from ENH #99)
with the Storage Layer backend traits (from ENH #412),
which complement the state model with traits, configuration
schema, scoping rules, and backend implementations.

### What?

Create a standard interface and machinery for State
in Praxis core. State is the single top-level
abstraction. It has two categories:

- **Ephemeral state** covers backends like Valkey
  and in-memory stores. Data may be lost on restart.
  Suitable for caches, counters, and session
  affinity across requests and replicas.

- **Storage state** covers persistent backends like
  PostgreSQL, SQLite, and file stores. Data survives
  restarts. Suitable for conversation history,
  response records, and durable business data.

Filters and other consumers (such as probes)
declare their state needs in configuration. Praxis
manages backend lifecycle, configuration, and
scoped access. State backends are scoped to a
specific filter, a named chain, or made globally
available with explicit opt-in.

Beneath the typed domain APIs from Part 1, the interface
defines two pluggable storage backend traits (one for
key-value lookups and one for object/blob storage) along
with the contract each backend must satisfy (multi-tenancy,
TTL, encryption, size limits) and reference implementations
for common backends. This is internal storage for proxy
operation, not storage exposed to clients: Praxis does not
become a storage proxy or gateway to S3/GCS/Rados; it uses
these systems internally to persist its own operational
state.

The existing `KvBackend` trait in `praxis-core` is
a runtime cache (in-memory, non-durable, no TTL, no
tenancy). It remains unchanged. The traits introduced
here are separate abstractions for durable,
distributed storage with richer semantics.

### Goals

- A single interface that covers both ephemeral
  and persistent state under one abstraction.
- Config-driven backend lifecycle: operators
  declare backends in YAML, Praxis provisions
  them at startup.
- Scoped access by default: backends are bound to
  a filter or chain. Global access requires
  explicit configuration.
- Pluggable backends: core defines traits and ships
  common defaults (in-memory, Valkey, SQLite,
  OpenAI Files API). External crates provide
  additional backends (e.g. PostgreSQL) without
  modifying core.
- Accessible to filters and to probes (background
  processes), decoupled from the HTTP request
  lifecycle.
- Define two storage backend traits accessible to
  filters via `HttpFilterContext`:
  - **Key-value trait**: small values, low latency,
    keyed lookups. Backends: in-memory (demo/test),
    Valkey/Redis (production).
  - **Object store trait**: large payloads, higher
    latency, blob storage. Backends: S3, GCS, Rados,
    local filesystem (demo/test).
- Define the contract that all backend implementations
  must satisfy:
  - Multi-tenancy: tenant-scoped access, one tenant
    cannot read another tenant's data.
  - TTL: per-entry expiration with configurable
    defaults.
  - Encryption: data encrypted at rest (backend-native
    or proxy-managed).
  - Size limits: per-entry and per-tenant quotas.
  - Failure semantics: configurable fail-open or
    fail-closed per filter, with timeouts on every
    storage operation.
- Provide reference implementations:
  - In-memory (key-value and object): for development,
    testing, demos, and single-replica deployments.
  - At least one distributed backend per trait for
    production multi-replica deployments.
- Define a reference schema for conversation and
  response persistence as the primary use case,
  covering: conversation history, response objects,
  input/output items, and metadata. Other consumers
  (rate limiters, session stores, caches) define
  their own schemas.
- Ensure filters that use storage degrade gracefully
  when storage is unavailable. No built-in filter
  should fail outright without storage unless it
  explicitly declares that dependency.

### Non-Goals

- Replacing the existing `KvBackend` runtime cache
  trait.
- Defining typed domain APIs for specific features
  (rate limiters, token ledgers, session stores).
  Those are defined in Part 1 above.
- Making Praxis a database. Storage backends are
  external systems; Praxis provides the integration
  interface.
- SQL databases as a backend. See the open question
  below, which this merge surfaces: the state model
  above lists SQLite/PostgreSQL as storage backends,
  while the storage layer originally treated relational
  conversation/response CRUD as a consumer-side concern
  (proposal #354).

### Why?

#### Ephemeral State

Praxis needs ephemeral state to function as a
stateful proxy.

**Multi-instance deployments.** Distributed
ephemeral state (Valkey) enables session affinity,
rate limiting, and feature flags that are consistent
across proxy replicas. Without a standard backend,
each feature that needs distributed ephemeral state
would build its own Redis client, key format,
timeout policy, and failure semantics.

**Stateful protocols.** MCP uses session IDs that
bind follow-up requests to the backend that owns
the session context. A2A tracks long-running tasks
across multiple request/response exchanges.
WebSocket and gRPC streaming sessions need
connection-to-session mapping. Without shared
ephemeral state, these protocols cannot be proxied
correctly across multiple requests.

**Cross-request state.** Rate limiters track
request counts across calls. Circuit breakers
accumulate failure counts over time. Health
snapshots inform load balancing decisions. These
patterns need state that persists across requests
and survives configuration reloads, but does not
need to survive a process restart.
Today each builds its own in-process store
(DashMap, atomics) with ad-hoc capacity limits
and no shared lifecycle management.

#### Storage State

Praxis needs persistent storage for the Responses
API, Conversations API, and the agentic loop.

**Responses and Conversations APIs.** The OpenAI
Responses API is stateful by design. Each response
can reference a `previous_response_id` to continue
a conversation. The Conversations API maintains
accumulated message history across turns. Both
must persist records so that subsequent requests
can rehydrate context from earlier turns. This
already works today via `ResponseStore` and
`ConversationItemStore` with SQLite and PostgreSQL
backends, but the implementations are
self-contained in the ai/ repository with their
own traits, registries, and configuration. Nothing
else can reuse them.

**Agentic loop orchestration.** The agentic loop
executes multiple inference rounds, accumulating
tool call results, conversation messages, and token
usage across iterations. When `store: true` is set,
the full conversation state must be persisted
durably so that clients can retrieve or continue it
later. Conversation compare-and-swap (already
implemented in the ai/ repo) shows the need for
concurrent-safe persistent state operations.

**Near-term consumers.** MCP session persistence
requires durable session-to-backend mappings that
survive proxy restarts and are shareable across
replicas. A2A task state tracks long-running task
lifecycle (submitted, working, completed, failed)
across multiple request/response exchanges. Both
are protocol-level concerns, not OpenAI-specific.
The Files API needs local file storage for
development environments (praxis-proxy/ai#494).
Each of these would otherwise build its own
trait, registry, and configuration - the same
scaffolding that `ResponseStore` and
`ConversationItemStore` already duplicated.

**Shared need.** Both ephemeral and storage state
need the same infrastructure: named backends
declared in configuration, scoped access control,
managed lifecycle, pluggable implementations, and
availability outside the HTTP request path (for
probes and background processes). Building this
once as a unified interface avoids duplicating the
machinery for each new feature.

#### Storage Layer Motivation

Praxis is an AI-native proxy. AI workloads are
structured around context: multi-turn conversations,
tool-call loops, cached inference results, and
request/response chains that reference prior state.
A stateless proxy that forgets everything between
requests cannot support these patterns at the
infrastructure level.

Today, filters have two options for state: request-
scoped metadata (lost after the response) and the
in-memory `KvBackend` cache (lost on restart, local
to one replica). Neither works for state that must
persist across requests, survive restarts, or be
accessible from any proxy instance in a fleet.

Concrete use cases that require durable, distributed
storage:

- **Conversation rehydration.** A Responses API
  request with `previous_response_id` must load the
  full conversation history. With multiple Praxis
  replicas, the history must be in a shared store,
  not local memory.
- **Expensive computation caching.** A request that
  triggers an expensive inference call or guardrail
  evaluation should store the result once and serve
  it on subsequent requests in the same workflow.
- **Cross-request context.** Agentic loops (MCP tool
  calls, multi-step reasoning) span multiple HTTP
  request/response cycles. The proxy must persist
  intermediate state so any instance can continue
  the loop.
- **Multi-replica correctness.** Local state
  (DashMap, in-process caches) works for single-
  replica development but silently produces wrong
  results in production fleets. Filters need a
  storage interface that makes the local-vs-shared
  distinction explicit.

Without a shared storage layer, each feature that
needs persistence will build its own: separate
connection management, incompatible key formats,
inconsistent failure handling, and no multi-tenancy.
The storage trait centralizes these concerns so
filter authors write against a stable interface and
operators configure backends once.

#### Why Two Traits

**Key-value**: session flags, routing decisions,
counters, tenant metadata. Bytes to low KB, sub-
millisecond access, on the request hot path.

**Object store**: serialized conversation history,
response objects, cached inference responses, file
attachments (up to 32 MiB per OpenAI spec). Tens of
KB to multi-MB per entry, accessed once per request
(rehydration) or once per response (persistence).

A KV interface lacks streaming and multipart
semantics for multi-MB blobs. An object store
interface adds unnecessary overhead for a 64-byte
counter. Different access patterns warrant different
abstractions.

#### Why Object Stores

**Cost.** Object storage is orders of magnitude
cheaper per GB than SSD-backed databases or managed
KV stores. AI workloads generate conversation state
at scale; storage cost is a real constraint.

**Availability.** S3-compatible object storage is
already present where Praxis deploys: Ceph/Rados
on-premise, S3/GCS/Azure Blob in cloud. Praxis
integrates with existing infrastructure rather than
requiring new systems.

The latency trade-off (tens to hundreds of
milliseconds) is acceptable: conversation
rehydration and persistence are once-per-request
operations, not per-token. The inference call itself
takes seconds.

### User Stories

- As a proxy operator, I want to declare state
  backends in YAML so that I manage connectivity,
  credentials, and TLS centrally instead of per
  filter.
- As a proxy operator, I want to configure a shared
  storage backend once so that all filters use it
  without per-filter connection management.
- As a proxy operator, I want to use S3-compatible
  storage for conversation persistence so that I
  reuse infrastructure I already operate.
- As a proxy operator, I want an in-memory storage
  backend so that I can run Praxis without external
  dependencies during development and demos.
- As a filter author, I want to request a named
  state backend through the filter context so that
  I do not manage backend lifecycle or connection
  pooling myself.
- As a filter author, I want to persist and retrieve
  data by key through a storage trait so that my
  filter works with any backend the operator
  configures.
- As a filter author, I want storage operations to
  have configurable timeouts and failure modes so
  that a slow or unavailable backend does not block
  request processing indefinitely.
- As a platform engineer, I want state scoped to
  specific filters by default so that one filter
  cannot corrupt another's state.
- As an AI gateway operator, I want the same
  interface for conversation persistence, file
  caching, and session tracking so that each
  feature does not build its own storage layer.
- As a Praxis developer, I want to add a new
  storage backend in an external crate without
  modifying core.
- As a probe author, I want access to the same
  state backends that filters use so that
  background processes share state without a
  separate configuration path.
- As an SRE, I want per-entry TTLs so that stale
  state is cleaned up without manual intervention.
- As an SRE, I want storage operations to expose
  latency metrics so that I can detect backend
  degradation before it affects request latency.
- As a security engineer, I want tenant-scoped
  storage access so that one tenant's data is never
  readable by another tenant.

## How?

### Overview

- Two async backend traits in `praxis-core`: `KvStore`
  for small hot-path values, `ObjectStore` for large
  blobs.
- A `StateRegistry` built from config at startup, owned
  in `ServerState`, threaded into every pipeline and
  re-attached across reload; reachable from filters via
  `HttpFilterContext` and from background tasks by clone.
- A top-level `state:` config block declaring named
  backends with explicit scope (filter, chain, global).
- Every backend honors the contract: tenant isolation,
  per-entry TTL, per-entry and per-tenant size limits, a
  timeout on every operation, and a per-consumer
  `failure_mode: open | closed`.
- Zero-dependency defaults so Praxis runs with no
  external service: an in-memory key-value store (with
  TTL, tenancy, and quota) and in-memory and filesystem
  object stores.
- No built-in filter fails when storage is absent unless
  it declares the dependency.
- SQL, Valkey/Redis, and S3/GCS/Rados backends live
  behind opt-in cargo features; external crates can add
  backends without changing core.
- The AI Responses/Conversations stores move onto the
  registry with no data migration.

### Design

#### Traits

Both traits are `async` and `Send + Sync`. Every call
takes a `Scope`, so a filter cannot reach another
tenant's data by accident.

```rust
#[async_trait]
pub trait KvStore: Send + Sync + Debug {
    async fn get(&self, scope: &Scope, key: &str)
        -> Result<Option<Bytes>, StateError>;
    async fn set(&self, scope: &Scope, key: &str,
        val: Bytes, ttl: Option<Duration>)
        -> Result<(), StateError>;
    async fn delete(&self, scope: &Scope, key: &str)
        -> Result<bool, StateError>;
    /// Conditional write on an opaque version token,
    /// not on serialized value bytes.
    async fn compare_and_set(&self, scope: &Scope,
        key: &str, expected: Option<&Version>,
        val: Bytes, ttl: Option<Duration>)
        -> Result<Version, StateError>;
    async fn incr_by(&self, scope: &Scope, key: &str,
        delta: i64, ttl: Option<Duration>)
        -> Result<i64, StateError>;
}

#[async_trait]
pub trait ObjectStore: Send + Sync + Debug {
    async fn put(&self, scope: &Scope, key: &str,
        body: ObjectBody, meta: ObjectMeta)
        -> Result<(), StateError>;
    async fn get(&self, scope: &Scope, key: &str)
        -> Result<Option<ObjectRead>, StateError>;
    async fn delete(&self, scope: &Scope, key: &str)
        -> Result<bool, StateError>;
    async fn list(&self, scope: &Scope, prefix: &str,
        cursor: Option<&str>, limit: u32)
        -> Result<ObjectPage, StateError>;
}
```

`StateError` is a typed enum (`NotFound`,
`PreconditionFailed`, `TooLarge`, `Timeout`,
`Unavailable`, `Backend`), so a caller can tell a
precondition or quota failure apart from a generic one.
Object bodies stream instead of buffering whole blobs,
since attachments run up to 32 MiB.

The existing `KvBackend` cache is left alone; these are
new, durable, tenant-aware traits. They are the backend
layer: the typed domain APIs from Part 1 (rate limits,
token ledgers, sessions) are the usual filter-facing
surface and build on top of them.

#### Registry and lifecycle

`StateRegistry` follows `KvStoreRegistry`: a cheap `Arc`
clone over one shared map, so the same handle survives
`ArcSwap` pipeline swaps.

```rust
#[derive(Clone, Debug)]
pub struct StateRegistry { /* Arc<inner> */ }

impl StateRegistry {
    pub fn kv(&self, name: &str)
        -> Option<Arc<dyn KvStore>>;
    pub fn object(&self, name: &str)
        -> Option<Arc<dyn ObjectStore>>;
}
```

The difference is where backends come from.
`KvStoreRegistry::get_or_create` hardcodes the in-memory
backend; `StateRegistry` builds its backends from
`config.state` at startup. From there the wiring matches
the registries we already have: owned in `ServerState`,
passed through `resolve_pipelines` and
`configure_pipeline` onto a `FilterPipeline` field (and
into branch and IRR sub-pipelines), exposed on
`HttpFilterContext`, and carried through `WatcherParams`
into `reload_pipelines`. Background jobs like a TTL sweep
or reconnect get their own clone on a dedicated runtime,
the same as health checks. Filters look up a backend by
name at request time; they never build one. Domain
backends core does not name, like the SQL conversation
store, register and fetch their own trait object by name,
the way the AI repo's registry already does.

#### Configuration

```yaml
state:
  backends:
    - name: hot
      kind: memory          # memory | valkey
      ttl_default: 30s
      max_entry_bytes: 65_536         # 64 KiB
      max_tenant_bytes: 16_777_216    # 16 MiB
    - name: blobs
      kind: filesystem      # memory | filesystem | s3 | gcs | rados
      path: /var/lib/praxis/objects
      max_object_bytes: 33_554_432    # 32 MiB
    - name: convo
      kind: sqlite          # sqlite | postgres (feature: sql)
      database_url: ${DATABASE_URL}
```

Backends are filter-scoped by default; chain or global
scope is opt-in. A filter names the backend it wants
(`store: convo`) and gets a typed handle. The schema
follows the usual conventions: `snake_case` enums,
`deny_unknown_fields`, `try_from` newtypes for bounded
numbers, and `${ENV}` for secrets.

#### Contract enforcement

TTL, size limits, timeouts, and metrics live in the
registry wrapper, so every backend gets them and none can
skip them. Metrics record latency and outcome per
operation, and never use tenant, key, or prompt as a
label, which would wreck cardinality. Encryption at rest
is a hook with a null default; proxy-managed envelope
encryption comes later, since it has to work with the
version-token compare-and-set and needs key management.

#### Default backends

- **In-memory key-value:** a new type (not the
  contract-less `InMemoryKvBackend`) with TTL, tenant
  scoping, and per-tenant quota. This is the zero-config
  default.
- **In-memory and filesystem object stores:** no driver,
  no server. The filesystem store uses tenant-prefixed
  paths, atomic write-then-rename, and a background TTL
  sweep.
- **SQLite and PostgreSQL** (feature `sql`): the existing
  `SqliteResponseStore` and `PostgresResponseStore`,
  reused as they are. The local default points at a file,
  never in-memory SQLite (one connection, gone on reload).

`sqlx` stays behind the feature, out of a default core
build.

#### Adapting the AI repo

None of this is a rewrite:

1. Keep `ResponseStore` and `ConversationItemStore` as
   domain traits; their transaction and pagination
   behavior is unchanged.
2. Register the store into the core `StateRegistry` at
   startup, instead of the current per-request, empty,
   single-`"default"`-key registry. That one change fixes
   reload survival, probe and admin access, the
   one-store-per-instance limit, and the ordering
   dependency between the store and rehydrate filters.
3. Move the generic pieces (pool config, SSL and SSRF
   validation, table-identifier checks, schema
   versioning) into the shared crate, and collapse the
   two duplicated config structs and `StorageBackend`
   enums into one.
4. Swap the serialized-JSON compare-and-swap for the
   version-token form.
5. Keep the existing DDL, table names, and schema version
   so no deployment needs a migration.

The reference schema for responses and conversations
(criterion 6) lives with the domain store, not behind the
generic trait.

#### Ephemeral and storage are separate

They differ in durability and concurrency, not size.
Ephemeral state is a hot-path cache that can be lost on
restart; storage state is durable, off the hot path, and
usually network-backed. One trait cannot do both well:
the hot path needs cheap access, while a durable backend
needs `async` I/O with a timeout on every call. Fold them
together and blocking I/O ends up on the request path,
which Part 1 rules out.

#### SQL is a registered backend, not a core trait

Core owns the registry, the lifecycle, and the two
generic traits (key-value and object store). Relational
access for responses and conversations stays a domain
trait (`ResponseStore`, `ConversationItemStore`) that
registers into the same registry behind an opt-in
feature, so praxis-core never pulls in a SQL driver. That
is what lets the compare-and-swap, transactional writes,
and keyset pagination those APIs depend on keep working,
instead of bending a generic trait until it becomes a
database.

#### Backends and their state survive reload

The registry is built once and re-attached to each
rebuilt pipeline, the way `KvStoreRegistry` and
`HealthRegistry` already work. A backend reconnects only
when its own config changes, and a teardown is logged.

### Implementation

The work splits into steps that can each land on their
own:

1. Core traits, `StateRegistry`, the `state:` config, and
   the zero-dependency defaults, wired through the
   pipeline and reload.
2. Background TTL sweep and eviction, plus the reload
   warning on stateful teardown.
3. The `sql` feature and the AI move above.
4. Distributed backends (Valkey, S3/GCS/Rados) for
   multi-replica correctness.
5. Follow-ups: encryption at rest and retention policies.

Every praxis change carries unit and integration tests,
an example under `examples/configs/state/`, and a
functional test for it.

# Notes

> **Merge note:** This proposal merges the former ENH #99
> (Stateful Proxy State Management) and ENH #412 (Storage
> Layer). #412 tracked the pluggable backend traits beneath
> #99's state model and was on hold; its content is folded
> in here and it is superseded by this proposal. Part 1 is
> the state model and typed domain APIs (from #99). Part 2
> is the unified state interface and the storage backend
> traits (from #99's interface section and #412). The one
> point where the two proposals disagreed, whether SQL is a
> first-class backend, is settled in the How? section.
