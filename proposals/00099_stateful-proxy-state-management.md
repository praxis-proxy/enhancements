---
issue: https://github.com/praxis-proxy/praxis/issues/99
discussion: https://github.com/praxis-proxy/praxis/issues/99#issuecomment-4411378263
status: proposed
repos:
  - praxis
  - ai
  - extproc
  - policy
authors:
  - nerdalert
  - shaneutt
  - rikatz
graduation_criteria:
  - State class taxonomy agreed by stakeholders
  - Scoping model (tenant plus consumer namespace; filter, chain, or global access) agreed by stakeholders
  - Storage trait API design (KvStore, SqlStore, ObjectStore) reviewed by stakeholders
  - KvStore value model (typed values, per-backend encoding) and conditional-write form (versions or load-link/store-conditional) settled by a written comparison with usage examples
  - SQL as a core variant with a SQLite file default agreed by stakeholders
  - Every variant has a local default implementation that runs with no external service
  - KvBackend deprecation path agreed by stakeholders
  - Backend connectivity fields aligned with the shared service definition once it lands
  - Reference schema for conversation/response storage
experimental_exempt: true
experimental_exempt_reason: "Core infrastructure and configuration schema change"
supersedes: 00412
related:
  - 00354
  - 00108
  - 00121
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
as the primary filter-facing abstraction. Every storage
variant ships a default implementation that runs entirely
locally, so Praxis starts and serves with no external
service; that default covers tests, development, demos,
explicitly single-replica deployments, and some niche HA
scenarios. The production guidance for multi-replica
deployments is to replace the local default with an
external backend (e.g. Valkey), with strict timeouts, TTLs,
key conventions, failure-mode defaults, and bounded metrics
labels.

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
  affinity across requests and replicas. Served by
  the key-value trait.

- **Storage state** covers persistent backends like
  PostgreSQL, SQLite, and file stores. Data survives
  restarts. Suitable for conversation history,
  response records, and durable business data.
  Served by the SQL and object store traits.

Filters and other consumers (such as probes)
declare their state needs in configuration. Praxis
manages backend lifecycle, configuration, and
scoped access. State backends are scoped to a
specific filter, a named chain, or made globally
available with explicit opt-in.

Beneath the typed domain APIs from Part 1, the interface
defines three pluggable storage backend traits, one per
variant: key-value (ephemeral), relational SQL (durable
records), and object/blob storage (durable payloads). Each
comes with the contract every backend must satisfy
(multi-tenancy, TTL, size limits, timeouts, failure
semantics), a default implementation that runs entirely
locally, and optional external backends. This is internal
storage for proxy operation, not storage exposed to
clients: Praxis does not become a storage proxy or gateway
to S3/GCS/Rados; it uses these systems internally to
persist its own operational state.

The existing `KvBackend` trait in `praxis-core` is a
runtime cache (in-memory, non-durable, no TTL, no
tenancy). The key-value trait introduced here supersedes
it: the admin key-value endpoints move onto the new
registry over a deprecation window, so core ends up with
one key-value abstraction rather than two.

### Goals

- A single interface that covers both ephemeral
  and persistent state under one abstraction.
- Config-driven backend lifecycle: operators
  declare backends in YAML, Praxis provisions
  them at startup.
- Scoped access by default: backends are bound to
  a filter or chain. Global access requires
  explicit configuration.
- Pluggable backends: core defines the traits and
  ships a local default for each variant (in-memory
  key-value, SQLite file, local filesystem). Valkey,
  PostgreSQL, and S3-compatible backends are opt-in
  cargo features. External crates provide additional
  key-value and object backends without modifying
  core; SQL backends are a closed set owned by core.
- Accessible to filters and to probes (background
  processes), decoupled from the HTTP request
  lifecycle.
- Define three storage backend traits accessible to
  filters via `HttpFilterContext`:
  - **Key-value trait**: small typed values, low
    latency, keyed lookups. Backends: in-memory
    (default), Valkey/Redis (multi-replica).
  - **SQL trait**: relational records with
    transactions and keyset pagination. Backends:
    SQLite file (default), PostgreSQL (multi-replica).
  - **Object store trait**: large payloads, higher
    latency, blob storage. Backends: filesystem-backed
    (default), in-memory (tests), S3-compatible
    including MinIO, GCS, Rados (multi-replica).
- Define the contract that all backend implementations
  must satisfy:
  - Multi-tenancy: tenant-scoped access, one tenant
    cannot read another tenant's data.
  - TTL: per-entry expiration with configurable
    defaults.
  - Encryption: an at-rest encryption hook on every
    backend, backend-native where the backend offers
    it; proxy-managed envelope encryption is a
    follow-up.
  - Size limits: per-entry and per-tenant quotas.
  - Failure semantics: configurable fail-open or
    fail-closed per filter, with timeouts on every
    storage operation.
- Provide reference implementations:
  - A local default per trait that needs no external
    service: in-memory key-value, SQLite file, local
    filesystem. These are real defaults for
    single-replica deployments, not demo-only stubs.
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

- Defining typed domain APIs for specific features
  (rate limiters, token ledgers, session stores).
  Those are defined in Part 1 above.
- Making Praxis a database. Storage backends are
  external systems or embedded engines; Praxis
  provides the integration interface.
- Cache APIs, replicated (CRDT) state, append-only
  sinks, and service lifecycle. See "Not in this
  proposal" in the How? section.

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

#### Why Three Traits

**Key-value**: session flags, routing decisions,
counters, tenant metadata. Bytes to low KB, sub-
millisecond access, on the request hot path. May be
lost on restart.

**SQL**: response records, conversation items,
pending approvals, catalogs. Rows with indexes,
transactions, and keyset pagination. On the request
path only for the requests that carry state (a
`previous_response_id`, a `conversation`), never per
token and never for every request. Must survive
restarts. The Responses and Conversations stores
already depend on transactional writes and
compare-and-swap that neither a key-value nor a blob
interface can express.

**Object store**: serialized conversation history,
response objects, cached inference responses, file
attachments (up to 32 MiB per OpenAI spec). Tens of
KB to multi-MB per entry, accessed once per request
(rehydration) or once per response (persistence).

A KV interface lacks streaming and multipart
semantics for multi-MB blobs and cannot express a
transaction across rows. An object store interface
adds unnecessary overhead for a 64-byte counter. A
relational interface is the wrong shape for either.
Different access patterns warrant different
abstractions, and each one gets a local default.

Choosing between them follows one rule: `SqlStore` is
the last resort. Its latency is the least predictable
of the three, because a query's cost depends on the
plan, the indexes, the table size, and lock contention,
none of which Praxis controls, and a long `UPDATE` or a
forgotten index lands on Praxis directly. Use it only
where a domain needs transactions, relational queries,
or keyset pagination over durable records, and never
for state every request touches (counters, flags,
affinity, per-request policy state), which belongs in
`KvStore`. A consumer that reaches SQL on the request
path gates it so only the requests that need it get
there.

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
  key-value or object storage backend in an external
  crate without modifying core.
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

### Requirements

- Three backend traits in `praxis-core`, one per
  variant: `KvStore` (ephemeral), `SqlStore` (durable
  records), `ObjectStore` (durable payloads).
- Every variant has a default implementation that
  runs entirely locally with no external service, and
  at least one external backend for multi-replica
  deployments.
- One registry, built from config at startup, that
  survives pipeline reload and is reachable from
  filters, probes, and background tasks.
- Every backend honors one contract: tenant isolation
  (enforced by the registry wrapper for key-value and
  object, and by the owning domain schema for SQL,
  where every domain store keys rows by tenant and
  proves it with a cross-tenant-read test), per-entry
  TTL and size limits where the variant supports them,
  a timeout on every operation, and a per-consumer
  failure mode.
- A backend declares its capabilities (shared across
  replicas, atomic operations, durable) so a consumer
  can refuse a configuration that cannot enforce what
  it promises.
- No built-in filter fails when storage is absent
  unless it declares the dependency.
- The default `praxis-core` library build carries no
  database or object-storage driver; external backends
  and `sqlx` sit behind cargo features.
- Existing AI stores move onto the registry with no
  data migration.

### Overview

- Three async backend traits in `praxis-core`:
  `KvStore` for small hot-path values, `SqlStore` for
  relational records, `ObjectStore` for large blobs.
- A `StateRegistry` built from config at startup,
  owned in `ServerState`, threaded into every pipeline
  and re-attached across reload; reachable from
  filters via `HttpFilterContext` and from background
  tasks by clone.
- A top-level `state:` config block declaring named
  backends grouped by variant (`kv`, `sql`, `object`),
  each with a kind, a timeout, and limits, and an
  access scope (filter, chain, global) on key-value and
  object entries.
- Every backend honors the contract: tenant isolation,
  per-entry TTL and per-entry and per-tenant size
  limits where the variant supports them, a timeout on
  every operation, and a per-consumer
  `failure_mode: open | closed`.
- Local defaults so Praxis runs with no external
  service: an in-memory key-value store (with TTL,
  tenancy, and quota), a SQLite file store, and a
  filesystem-backed object store, plus an in-memory
  object store for tests.
- Valkey/Redis, PostgreSQL, and S3-compatible
  (including MinIO), GCS, and Rados backends live
  behind opt-in cargo features; external crates can
  add key-value and object backends without changing
  core.
- `KvStore` supersedes the existing `KvBackend`
  runtime cache over a deprecation window.
- The AI Responses/Conversations stores move onto the
  registry with no data migration.

### Design

#### Types

```rust
/// Who owns the data and which consumer wrote it.
#[derive(Clone, Debug, Eq, PartialEq, Hash)]
pub struct Scope {
    /// Trusted tenant identity. Never derived from an
    /// unauthenticated request header.
    pub tenant: Arc<str>,
    /// Consumer namespace: the filter instance name for
    /// filter-scoped backends, the chain name for
    /// chain-scoped ones, or `global`.
    pub namespace: Arc<str>,
}

/// Opaque, backend-assigned version of one key.
#[derive(Clone, Copy, Debug, Eq, PartialEq)]
pub struct Version(u64);

/// The core value type (`String`, `Bytes`, `Bool`,
/// `Int(i64)`, `UInt(u64)`, `Double(f64)`) from the
/// core type system work (praxis-proxy/praxis#1234).
/// `KvStore` reuses it; this proposal does not define
/// its own.
pub use crate::value::Value;

/// What a backend can promise.
#[derive(Clone, Copy, Debug)]
pub struct Capabilities {
    /// Writes are visible to every replica.
    pub shared: bool,
    /// `compare_and_set` and `incr_by` are atomic.
    pub atomic: bool,
    /// Data survives a process restart.
    pub durable: bool,
}

/// One variant per dialect core ships.
#[non_exhaustive]
pub enum SqlDialect { Sqlite, Postgres }

/// The `sqlx` pool behind a SQL backend.
#[non_exhaustive]
pub enum SqlPool {
    Sqlite(sqlx::SqlitePool),
    Postgres(sqlx::PgPool),
}

/// A named, versioned set of DDL statements a domain
/// store installs once per backend.
pub struct SchemaSpec<'a> {
    pub name: &'a str,
    pub version: i64,
    pub ddl: &'a [&'a str],
}

/// Streaming body; never a whole blob in memory.
pub struct ObjectBody(Pin<Box<dyn AsyncRead + Send>>);

pub struct ObjectMeta {
    pub content_type: Option<String>,
    pub size_hint: Option<u64>,
    pub ttl: Option<Duration>,
}

pub struct ObjectInfo {
    pub size: u64,
    pub content_type: Option<String>,
    pub etag: Option<String>,
    pub modified: SystemTime,
}

pub struct ObjectRead {
    pub info: ObjectInfo,
    pub body: ObjectBody,
}

pub struct ObjectPage {
    pub entries: Vec<(String, ObjectInfo)>,
    pub next: Option<String>,
}
```

`Scope` carries two things the traits need on every
call: the tenant, so one tenant can never read
another's data by accident, and the consumer
namespace, which is how "filter-scoped by default"
works. Two filters that name the same backend get
disjoint namespaces unless the backend is declared
chain- or global-scoped. The namespace is the
configured filter or chain name, which is stable
across reloads, so a rebuilt filter finds its own
data again. The AI repo's `StateOwner` maps its
`tenant_id` onto `Scope::tenant` and keeps issuer and
subject in its own schema.

`Capabilities` is how a consumer refuses an unsafe
configuration. A token quota configured to enforce a
fleet-wide budget checks `shared` at startup and
fails closed on a memory backend instead of silently
enforcing per replica. A consumer that needs
durability checks `durable` the same way.

#### Traits

All three traits are `async` and `Send + Sync`. Every
`KvStore` and `ObjectStore` call takes a `Scope`.
`SqlStore` hands out a pool; its rows are
tenant-scoped by the domain schema that owns them.

```rust
#[async_trait]
pub trait KvStore: Send + Sync + Debug {
    fn capabilities(&self) -> Capabilities;
    async fn get(&self, scope: &Scope, key: &str)
        -> Result<Option<Value>, StateError>;
    /// Value and version together, so a
    /// read-modify-write never needs a probing CAS.
    /// Provisional; see "Conditional writes".
    async fn get_versioned(&self, scope: &Scope,
        key: &str)
        -> Result<Option<(Value, Version)>, StateError>;
    async fn set(&self, scope: &Scope, key: &str,
        val: Value, ttl: Option<Duration>)
        -> Result<(), StateError>;
    async fn delete(&self, scope: &Scope, key: &str)
        -> Result<bool, StateError>;
    /// Conditional write on an opaque version token.
    /// Provisional; see "Conditional writes".
    async fn compare_and_set(&self, scope: &Scope,
        key: &str, expected: Option<Version>,
        val: Value, ttl: Option<Duration>)
        -> Result<Version, StateError>;
    async fn incr_by(&self, scope: &Scope, key: &str,
        delta: i64, ttl: Option<Duration>)
        -> Result<i64, StateError>;
}

#[async_trait]
pub trait SqlStore: Send + Sync + Debug {
    fn capabilities(&self) -> Capabilities;
    fn dialect(&self) -> SqlDialect;
    /// The `sqlx` pool for the dialect. Reached only
    /// through `SqlHandle::run`.
    fn pool(&self) -> &SqlPool;
    /// Install a domain schema once, keyed by name
    /// and version.
    async fn ensure_schema(&self, schema: &SchemaSpec<'_>)
        -> Result<(), StateError>;
}

#[async_trait]
pub trait ObjectStore: Send + Sync + Debug {
    fn capabilities(&self) -> Capabilities;
    async fn put(&self, scope: &Scope, key: &str,
        body: ObjectBody, meta: ObjectMeta)
        -> Result<(), StateError>;
    async fn get(&self, scope: &Scope, key: &str)
        -> Result<Option<ObjectRead>, StateError>;
    async fn head(&self, scope: &Scope, key: &str)
        -> Result<Option<ObjectInfo>, StateError>;
    async fn delete(&self, scope: &Scope, key: &str)
        -> Result<bool, StateError>;
    async fn list(&self, scope: &Scope, prefix: &str,
        cursor: Option<&str>, limit: u32)
        -> Result<ObjectPage, StateError>;
}
```

`StateError` is a typed enum (`PreconditionFailed`,
`InvalidValue`, `TooLarge`, `Timeout`, `Unavailable`,
`Backend`), so a caller can tell a precondition or
quota failure apart from a generic one. A miss is
`Ok(None)` on every `get` and `head`, never an error:
there is no `NotFound` variant, so backends cannot
disagree about which to return, and callers that
expect misses (a TTL expiry is a miss) handle one code
path. Object bodies stream instead of buffering whole
blobs, since attachments run up to 32 MiB.

`SqlStore` is `sqlx` on purpose. A domain store has to
run queries, so either the handle exposes `sqlx` types
or core grows a query layer of its own, which is the
bend-a-trait-until-it-is-a-database outcome this
proposal rejects. `SqlPool` is a `#[non_exhaustive]`
enum over the `sqlx` pool per dialect, `praxis-core`
re-exports `sqlx` under the `sql` feature so consumers
never land on a second copy with mismatched types, and
a `sqlx` major bump is a breaking change for the `sql`
feature, which is acceptable before 1.0. SQL backends
are therefore a closed set owned by core: a new dialect
is a core change, unlike key-value and object backends,
which external crates can add. Domain stores write
dialect-specific SQL inside `SqlHandle::run`, the only
way to reach the pool. The traits are the backend
layer. The typed domain APIs from Part 1 (rate limits,
token ledgers, sessions) are the usual filter-facing
surface and build on top of them. Relational domain
traits (`ResponseStore`, `ConversationItemStore`) take
a `SqlHandle` and install their schema through
`ensure_schema`, instead of registering beside the
generic traits.

**Values.** `KvStore` speaks the core `Value` type,
not raw bytes, so two consumers never have to agree on
a byte encoding to share a key, and the typed domain
layer gets integers and strings back as what they
are. Each backend owns how it encodes a `Value` and
keeps enough type information to hand back the variant
it stored. The in-memory backend keeps the enum as is.
An embedded store is free to use fixed-width or varint
integers. The Valkey backend stores integers as decimal
strings, because that is the only form `INCRBY`
accepts, so `incr_by` is one native command and a
plain `GET` still reads the key. A backend whose data
is shared across replicas treats its encoding as a
versioned wire format, since a rolling upgrade has two
Praxis versions reading the same keys. `max_entry_bytes`
applies to the encoded value. `ObjectStore` stays
bytes: its values are blobs. Where `Value` lives so the
policy engine can speak the same contract without
depending on Praxis is settled with the type system
work, not here.

**Conditional writes.** The form here is provisional.
`compare_and_set` with `expected: None` succeeds only
when the key is absent (create-if-absent); with
`Some(v)` it succeeds only when the key's current
version equals `v`. Success returns the new `Version`.
Failure returns `StateError::PreconditionFailed
{ current }` with the key's current version, or `None`
if it no longer exists. `get_versioned` returns the
value and its version together, so a read-modify-write
is one read and one conditional write, never a CAS
issued only to learn the version. `Version` is `Copy`,
backend assigned, monotonic per key, and never derived
from the value, so two writers storing equal values
still get distinct versions; `set` and `incr_by` bump
it too. The alternative is a load-link/store-conditional
form, where only in-flight pairs carry a token and the
backend stores no version per entry. The two differ in
cost on Valkey, where a per-key version turns every
write into a script or hash update while
`WATCH`/`MULTI`/`EXEC` is itself load-link shaped but
pins a connection per pair, and they differ in how
pleasant they are to use. A written comparison of the
two forms with worked usage examples settles which one
the trait ships; that is a graduation criterion, and
`get_versioned` is the stopgap it replaces if load-link
wins.

**Counters.** `incr_by` operates on integer values. An
absent key counts as `0`; a stored `Value::Int` is
adjusted by `delta` and the result returned; any other
variant, or a result outside `i64`, fails with
`StateError::InvalidValue`. `delta` may be negative, so
a gauge (an in-flight count) and a counter share one
operation. Whether counters should instead be unsigned,
as a rate limiter's are, is settled with the value
comparison above. For `incr_by` and `set` alike,
`ttl: Some(d)` sets or refreshes the key's expiry,
while `ttl: None` leaves an existing expiry alone and
gives a new key the backend's `ttl_default`.

**Objects.** `head` returns metadata without the body,
for size checks and conditional fetches. Multipart
upload is an implementation detail of `put` on
S3-class backends: `ObjectBody` streams, so a backend
chunks as it likes and the trait never exposes parts.
Object tagging and attribute queries beyond
`ObjectInfo` are follow-ups; the S3 and Swift object
APIs both map onto `ObjectMeta` and `ObjectInfo` once a
consumer needs them.

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
    pub fn sql(&self, name: &str) -> Option<SqlHandle>;
    pub fn object(&self, name: &str)
        -> Option<Arc<dyn ObjectStore>>;
}

/// A SQL backend behind the timeout and metrics
/// wrapper; the only way to reach its pool.
#[derive(Clone, Debug)]
pub struct SqlHandle { /* Arc<dyn SqlStore> + limits */ }

impl SqlHandle {
    pub fn dialect(&self) -> SqlDialect;
    /// Run one named operation with the backend
    /// timeout applied and its latency and outcome
    /// recorded.
    pub async fn run<T>(&self, op: &'static str,
        f: impl AsyncFnOnce(&SqlPool)
            -> Result<T, sqlx::Error>)
        -> Result<T, StateError>;
}
```

`sql` returns a `SqlHandle` rather than a bare trait
object so every query, not only pool checkout, passes
through the timeout and metrics wrapper. The `kv` and
`object` handles are wrapped the same way inside the
registry.

The difference is where backends come from.
`KvStoreRegistry::get_or_create` hardcodes the in-memory
backend; `StateRegistry` builds its backends from
`config.state` at startup. From there the wiring matches
the registries we already have: owned in `ServerState`
next to `KvStoreRegistry`, `HealthRegistry`, and the
sticky-session `SessionStoreRegistry`, passed through
`resolve_pipelines` and `configure_pipeline` onto a
`FilterPipeline` field (and into branch and IRR
sub-pipelines), exposed on `HttpFilterContext`, and
carried through `WatcherParams` into `reload_pipelines`.
The ExtProc server builds its `HttpFilterContext` from
the same pipeline fields and picks the registry up the
same way. Background jobs like a TTL sweep or reconnect
get their own clone on a dedicated runtime, the same as
health checks. Filters look up a backend by name at
request time; they never build one.

#### Configuration

```yaml
state:
  kv:
    - name: hot
      kind: memory          # memory | valkey
      scope: filter         # filter | chain | global
      timeout: 50ms
      ttl_default: 30s
      max_entry_bytes: 65_536         # 64 KiB
      max_tenant_bytes: 16_777_216    # 16 MiB
    - name: ledger
      kind: valkey
      url: valkey://valkey.internal:6379
      topology: standalone  # standalone | sentinel | cluster
      credential:
        env_var: VALKEY_PASSWORD    # or value: ...
      scope: global
      timeout: 100ms
  sql:
    - name: convo
      kind: sqlite          # sqlite | postgres
      path: /var/lib/praxis/state/convo.db
      timeout: 2s
  object:
    - name: blobs
      kind: filesystem      # memory | filesystem | s3 | gcs | rados
      scope: chain
      path: /var/lib/praxis/objects
      timeout: 5s
      max_object_bytes: 33_554_432    # 32 MiB
```

Backends are grouped by variant, so the family is the
config key and `kind` is an enum per family: `memory`
and `valkey` under `kv` register a `KvStore`, `sqlite`
and `postgres` under `sql` register a `SqlStore`, and
`memory`, `filesystem`, `s3`, `gcs`, and `rados` under
`object` register an `ObjectStore`. A `sql` entry is a
relational backend with transactions and pagination
that the generic traits do not expose; it never
masquerades as a key-value or object store, and the
domain traits that need SQL (`ResponseStore`,
`ConversationItemStore`) build on it. Names are unique
across families, so `store: convo` resolves without
ambiguity and `StateRegistry::sql("convo")` is the only
accessor that returns it.

A consumer names the backend it wants and sets its own
failure mode:

```yaml
filters:
  - name: openai_response_store
    config:
      store: convo
      failure_mode: closed  # open | closed
```

Key-value and object backends are filter-scoped by
default; chain or global scope is opt-in per backend,
and `Scope::namespace` carries the choice on every
call. A `sql` entry takes no `scope` and the schema
rejects one: `SqlStore` never sees a `Scope`, so
sharing and tenancy are the owning domain schema's
business, and any consumer that names the backend may
use it, which is how the AI store, rehydrate,
conversations, and compact filters share `convo`. A
filter names the backend it wants (`store: convo`) and
gets a typed handle; a consumer that asks for a `kv`
handle by a `sql` name fails at startup.
`failure_mode` is set where the
backend is consumed, since the same backend can be
advisory for one filter and enforcement for another.
`timeout` bounds every operation, and a backend that
omits it gets a conservative default for its kind. The
local defaults take a `path`; only external backends
take a `url`. The schema follows the usual conventions:
`snake_case` enums, `deny_unknown_fields`, and
`try_from` newtypes for bounded numbers. A credential
sits in its own field as either a literal `value` or an
`env_var` reference, the convention the credential
filters already use, and the config dump's redaction of
those keys extends to the `state:` block; a credential
never rides in the `url`, which ends up in logs. Praxis
has no `${VAR}` interpolation, and this proposal does
not add one.

Connectivity fields on external backends (`url`, TLS,
auth) are a placeholder for the shared service
definition being drafted separately. When it lands, a
backend references a service by name instead of
carrying its own connection fields, so state backends
and upstream clusters share one definition of TLS and
auth. This proposal does not define that service model
and does not block on it.

Deployment topology is the backend's business too. A
Valkey entry declares standalone, Sentinel, or Cluster
mode in its `topology` field, and the backend handles
discovery, slot routing, and failover through the
client library; consumers never see a shard map, a
Sentinel, or a reconnect. `KvStore` operations are
single-key, so they are safe on Cluster without hash
tags; a backend that stores a version beside a value
keeps both under one key so that stays true. Multi-key
scripted ledgers (proposal 00121) borrow the pool and
own their own slot discipline.

#### Contract enforcement

TTL, size limits, timeouts, and metrics live in the
registry wrapper, so every backend gets them and none
can skip them. Per-entry limits are checked in the
wrapper before a write reaches the backend. Per-tenant
quotas are exact on the local defaults, which see every
write, and best-effort on distributed backends, which
would need their own usage accounting; the wrapper
records what it can and the backend's native quotas do
the rest. Metrics record latency and outcome per
operation, labeled by backend name, kind, and operation
only, never by tenant, key, or prompt, which would wreck
cardinality. Encryption at rest is a hook with a null
default; backends that encrypt natively (PostgreSQL,
S3) report it through the hook, and proxy-managed
envelope encryption comes later, since it has to work
with conditional writes and takes its keys from the
secrets interface that is out of scope here.

For SQL the wrapper is `SqlHandle::run`. It applies the
backend `timeout` to the whole operation, records
latency and outcome under the operation name, and maps
`sqlx` errors onto `StateError`, so the rule that no
backend skips the contract holds for every query and
not only for pool checkout. The timeout is pushed
server-side too, so a query the caller gave up on stops
holding a connection and the database: the PostgreSQL
backend derives `statement_timeout` and `lock_timeout`
from the backend `timeout` and sets them in the
connection options rather than the URL, the SQLite
backend sets `busy_timeout` from it and aborts long
statements through a progress handler, and each pool's
`acquire_timeout` defaults to the backend `timeout`
instead of `sqlx`'s 30 seconds. Each named backend has
its own pool, so one slow consumer cannot drain
another's connections, and the metrics include pool
saturation (in use, idle, waiting) per backend. One
operator note follows from `lock_timeout`: DDL and bulk
rewrites against a live backend belong in a maintenance
window, since they now turn into fast failures (closed,
for rehydration) rather than slow requests.

#### Default backends

- **In-memory key-value:** a new type with TTL, tenant
  scoping, and per-tenant quota. This is the zero-config
  default for the key-value variant and the backend the
  admin key-value endpoints move onto.
- **SQLite file** (feature `sql`): the zero-config
  default for the SQL variant. It reuses the pool,
  schema-version, and identifier-validation code the AI
  stores already have. The default points at a file,
  never in-memory SQLite (one connection, gone on
  reload), and runs in WAL mode so readers never wait
  on the single writer, with `busy_timeout` taken from
  the backend `timeout`. A Rust-native embedded engine
  could replace it later as another dialect behind a
  `sqlx` driver; `SqlDialect` and `SqlPool` are
  `#[non_exhaustive]` for that reason.
- **Filesystem-backed object store:** the default for
  the object variant, a first-class backend rather than
  a demo stub. It uses tenant-prefixed paths, atomic
  write-then-rename, and a background TTL sweep. An
  in-memory object store exists for tests.

`ObjectStore` is a contract, not a storage technology:
any backend that offers put, get, head, delete, and
list-by-prefix over opaque blobs qualifies, whether it
is a POSIX filesystem, an in-memory map, an
S3-compatible service, or an embedded store such as
RocksDB. A filesystem is a POSIX store and an object
store is a different category of system, which is why
the default is named for what backs it: the
filesystem-backed object store satisfies the object
contract on local disk and offers none of the POSIX
semantics (append, seek, rename, locking) that the
contract leaves out. A consumer that needs those gets
its own variant rather than a bent `ObjectStore`. The
filesystem backend is the local default because it
needs nothing installed, not because it is a demo. It
lists by walking the directory under the tenant prefix
and has no multipart or tagging, which the contract
does not require. The `list` cursor is opaque per
backend (a continuation token on S3, the last path on
the filesystem).

`sqlx` stays behind the `sql` feature, out of a default
`praxis-core` library build. The `praxis` server binary
turns the feature on so the SQLite default works out of
the box; embedding consumers opt in.

The FIPS build is the one exception to "every variant
has a local default". The `sqlx` facade enables its
`migrate` feature unconditionally, which pulls in
`sha2`, a crate the FIPS dependency gate denies, so the
FIPS binary is built without `sql`, has no SQL variant,
and rejects a `state.sql` block at startup with an
error that says so. The carve-out lifts when upstream
`sqlx` makes `migrate` optional in the facade;
PostgreSQL authentication pulls `md-5` and `hmac` on
top, so it stays out of the FIPS build longer than
SQLite does.

#### Adapting the AI repo

None of this is a rewrite:

1. Keep `ResponseStore` and `ConversationItemStore` as
   domain traits; their transaction and pagination
   behavior is unchanged. They take a `SqlHandle`
   from the registry instead of building their own
   pool.
2. Register the backend into the core `StateRegistry`
   at startup, instead of the current per-pipeline
   `ResponseStoreRegistry` that is rebuilt on every
   reload and filled lazily under one `"default"` key.
   That one change fixes reload survival, probe and
   admin access, the one-store-per-instance limit, and
   the ordering dependency between the store and
   rehydrate filters.
3. Move the generic pieces (pool config, SSL and SSRF
   validation, table-identifier checks, schema
   versioning) into the core SQLite and PostgreSQL
   backends, and collapse the two duplicated config
   structs and `StorageBackend` enums into the `state:`
   block.
4. Swap the serialized-JSON compare-and-swap for a
   version column in the domain schema.
5. Keep the existing DDL, table names, and schema
   version so no deployment needs a data migration.
   Operators do move `backend:` and `database_url:`
   from each filter into one `state.sql` entry and
   reference it by name; the release notes carry that
   mapping.

The reference schema for responses and conversations
(the reference-schema graduation criterion) lives with
the domain store, not behind the generic trait.

#### Ephemeral and storage are separate

They differ in durability and concurrency, not size.
Ephemeral state is a hot-path cache that can be lost on
restart; storage state is durable, reached only by the
requests that need it, and usually network-backed. The
traits follow that line:
`KvStore` is the ephemeral variant and its backends may
drop data on restart, while `SqlStore` and `ObjectStore`
are the durable variants and their backends must not.
One trait cannot do both well: the hot path needs cheap
access, while a durable backend needs `async` I/O with a
timeout on every call. Fold them together and durable
I/O ends up on every request path, which Part 1 rules
out.
`Capabilities::durable` makes the line visible at
runtime, so a consumer that needs durability can check
for it.

#### SQL is a core variant

SQL is the third core variant, not a consumer-side
concern. Core owns the `SqlStore` trait, the SQLite and
PostgreSQL backends behind the `sql` feature, and the
schema-version hook. Relational domain traits
(`ResponseStore`, `ConversationItemStore`, the Files API
catalog) build on a `SqlHandle` and own their own DDL.
That keeps compare-and-swap, transactional writes, and
keyset pagination available to the domain stores
without bending a generic key-value trait until it
becomes a database, and it gives every deployment
outside the FIPS build a working SQL default with no
external service. This
settles the point on which the two source proposals
disagreed.

#### KvStore supersedes KvBackend

Two key-value abstractions in core is the outcome Part 1
exists to prevent, so `KvStore` replaces `KvBackend`
rather than sitting beside it. `KvBackend` has no TTL,
tenancy, quota, or async I/O, and its only consumers are
the admin key-value endpoints and filters that call
`get_or_create` on demand. The path: land `KvStore` and
the in-memory default; point the admin `/api/kv`
endpoints at a named `KvStore` backend; migrate the
built-in consumers; deprecate `KvBackend` and
`KvStoreRegistry` for one release; remove them. The
sticky-session `SessionStoreRegistry` and the policy
engine's `SessionStore` are key-value-with-TTL consumers
and migrate onto `KvStore` in the same pass, so the
policy engine's Valkey session store becomes the core
Valkey backend instead of a second client stack.

#### Backends and their state survive reload

The registry is built once and re-attached to each
rebuilt pipeline, the way `KvStoreRegistry` and
`HealthRegistry` already work. A backend reconnects only
when its own config changes, and a teardown is logged.

#### Not in this proposal

- Cache APIs with load-through and coalescing (the
  policy engine's `moka` token cache, `tinyufo`, the
  per-IP rate-limit maps). Part 1 keeps caches separate
  from state; they may later be built on `KvStore` but
  are not a variant here.
- Replicated, gossip-based state (the Grid `crdt`
  crate's LWW registers, OR-sets, and G-counters). A
  CRDT-backed `KvStore` is possible later; the
  state-class table treats it as a shared hot-path
  backend without a central server.
- Append-only sinks for audit and usage export. Part 1
  exports durable business state asynchronously; the
  sink interface is a separate proposal.
- Service lifecycle and the shared service definition
  (endpoints, TLS, auth). Backend connectivity fields
  adopt it when it lands; this proposal does not define
  it. State backends are host-owned and reached only
  through handles, so a named backend can later be
  provided by that service model without the traits
  changing. Retries and circuit breaking belong to that
  model, not here.
- Scripted atomic ledgers (the token rate limit's
  reserve/reconcile `EVAL` scripts). The typed domain
  layer in proposal 00121 owns them and borrows the
  Valkey pool from the named backend rather than
  opening its own.
- Secrets management (Vault, KMS, HSM). A secrets store
  has its own contract: read-only, resolved at startup
  and on refresh rather than per request, zeroized on
  drop, never in errors or logs, and addressed by name.
  That is a different store from the three here, so it
  gets its own interface and its own epic rather than
  a fourth variant. The policy engine's `SecretProvider`
  (env, file, and Vault backends) is the shape to lift
  into core. When it lands, backend credentials become
  named secrets instead of `env_var` references, and
  the encryption-at-rest follow-up takes its keys from
  the same interface; an HSM belongs there as a
  key-wrap backend, not here as storage.

### Experimental Phase

This is core infrastructure and a configuration schema
change, both exempt categories, so
`experimental_exempt: true` is set. The alternative path
is the cargo feature: external backends land behind
`valkey`, `sql`, and object-storage features that are
off in the core library, and the `state:` block itself
lands behind the `experimental` build tag until the
trait API graduates.

### Implementation

The work splits into steps that can each land on their
own:

1. Core types and traits (`Scope`, `Version`,
   `Capabilities`, `KvStore`, `SqlStore`,
   `ObjectStore`), `StateRegistry`, the `state:`
   config, and the local defaults (in-memory key-value
   and filesystem objects; SQLite is step 3), wired
   through the pipeline, reload, and the ExtProc
   server. The
   registry, config, lifecycle, and reload work can
   land first; the `KvStore` operations wait on the
   core `Value` type and the conditional-write
   comparison.
2. Background TTL sweep and eviction, plus the reload
   warning on stateful teardown.
3. The `sql` feature: the SQLite file default, the
   PostgreSQL backend, and the AI move above.
4. Migrations onto `KvStore`: the admin `/api/kv`
   endpoints, the sticky-session `SessionStoreRegistry`,
   the policy engine's `SessionStore`; then deprecate
   `KvBackend`.
5. Distributed backends (Valkey, S3/GCS/Rados) for
   multi-replica correctness, and the token rate
   limit's Valkey ledger borrowing the shared pool.
6. Follow-ups: encryption at rest, retention policies,
   and object tagging.

Every praxis change carries unit and integration tests,
an example under `examples/configs/state/`, and a
functional test for it.

## Notes

> **Merge note:** This proposal merges the former ENH #99
> (Stateful Proxy State Management) and ENH #412 (Storage
> Layer). #412 tracked the pluggable backend traits beneath
> #99's state model and was on hold; its content is folded
> in here and it is superseded by this proposal. Part 1 is
> the state model and typed domain APIs (from #99). Part 2
> is the unified state interface and the storage backend
> traits (from #99's interface section and #412). The one
> point where the two proposals disagreed, whether SQL is a
> first-class backend, is settled in the How? section: SQL
> is a core variant with a local default.
