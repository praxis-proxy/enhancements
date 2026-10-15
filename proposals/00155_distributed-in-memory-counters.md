---
issue: https://github.com/praxis-proxy/praxis/issues/155
discussion: https://github.com/praxis-proxy/praxis/issues/155
status: proposed
repos:
  - praxis
  - ai
authors:
  - henschwartz
graduation_criteria:
  - "How? section with requirements, design, and consumer API sketch landed in a follow-up PR"
  - "Cross-worker shared store: concurrency test proves two workers see each other's incr/decr on the same key"
  - "Atomic incr/decr with TTL: unit tests cover concurrent updates, expiry removal, and late-decrement after expiry"
  - "Key types IP, string, and integer: serde round-trip tests plus one consumer call per type"
  - "Metric families or generic counter: written contract naming families (or the generic path) reviewed in How?"
  - "Configurable max entries and TTL: config schema plus tests that enforce the bound and expire entries on schedule"
  - "Multiple concurrent keys per request: integration test updates at least two keys on one request and reads both back"
  - "Rate-limit / abuse / routing consumer API: documented call surface with at least one in-tree caller stub"
  - "Active connection/session counts: How? locks gauge-or-lease semantics and late-decrement behavior (see What?)"
  - "Peer replication (post-MVP): design note comparing peer gossip vs shared backend accepted by stakeholders"
  - "Bounded-state metrics: named series for table size and eviction/expiry rate with documented label bounds"
  - "Alignment with proposal 00099: How? cites the typed state APIs it consumes once 00099 is accepted"
stakeholders:
  - shaneutt
  - jordigilh
epic: https://github.com/praxis-proxy/praxis/issues/163
related:
  - 00099
  - 00121
  - 00123
origin:
  repo: praxis
  issue: https://github.com/praxis-proxy/praxis/issues/155
  file: 00155_distributed-in-memory-counters.md
---

# Distributed In-Memory Counters

## What?

Add a shared in-memory key-value counter store for
tracking per-client and per-key metrics (connection
count, request rate, error rate, bytes) that rate
limiting, abuse detection, and routing conditions can
consume — with optional peer replication across proxy
instances for multi-replica correctness.

Today Praxis has filter-local counter patterns (for
example the per-IP request `rate_limit` path uses a
local map with soft/hard caps) but no project-wide
keyed counter primitive with atomic incr/decr,
TTL-based expiry, configurable table bounds, and a
documented consumer API. Multi-replica deployments
therefore either under-enforce (each replica tracks
independently) or invent one-off backends per feature.
This proposal adds:

- Keyed counters with key types: IP, string, and
  integer
- Tracked data suitable for connection count/rate,
  session count/rate, HTTP request/error count/rate,
  and bytes in/out. Active connection and session
  counts are not plain TTL entries: a long-lived
  connection can outlive a generic TTL and leave the
  store wrong after a late `decr`. How? must choose
  either non-expiring gauges (remove only on
  explicit `decr` / disconnect) or refreshed leases
  (renew while the connection stays alive), and must
  define late-decrement after expiry (ignore, clamp
  at zero, or error). Rate and byte counters may keep
  TTL-based entry expiry.
- Atomic increment and decrement with TTL-based entry
  expiry for rate/byte style counters (the minimum
  single-instance consumer contract for rate-limit
  style work; token-budget reservation stays in
  00121)
- Configurable table size and entry expiry so memory
  stays bounded under high key cardinality
- Usability from rate limiting, abuse detection, and
  routing conditions without each filter owning a
  private map
- Multiple concurrent tracking keys per request
- Optional peer replication across instances for
  distributed state (staged after a single-instance
  in-memory MVP)

### Goals

- Provide a shared keyed counter store visible to
  proxy workers so filters do not each invent private
  DashMap-style accounting.
- Support atomic increment and decrement with
  TTL-based expiry as the MVP contract for
  single-instance consumers such as rate limiting
  before peer replication lands. Token-budget
  reservation and reconciliation stay in 00121 and
  are out of scope here.
- Accept key types IP, string, and integer so
  operators and filters can key on peer address,
  headers/tenant IDs, and numeric identifiers.
- Track the metric families named in
  [#155](https://github.com/praxis-proxy/praxis/issues/155):
  connection and session count/rate, HTTP
  request/error count/rate, and bytes in/out — or
  expose a generic counter that those consumers can
  specialize.
- Enforce bounded state: configurable max entries,
  TTL/expiry, eviction under pressure, and visible
  metrics for table size and eviction/expiry rate.
- Allow multiple concurrent tracking keys on a single
  request so compound policies (e.g. per-IP and
  per-tenant) share one store.
- Expose a consumer integration surface that rate
  limiting, abuse detection, and routing conditions
  can call without owning storage.
- Align storage class and TTL patterns with proposal
  [#99](https://github.com/praxis-proxy/praxis/issues/99) /
  [00099](00099_stateful-proxy-state-management.md);
  the How? section will adopt that proposal's typed
  state APIs once accepted.
- Design optional peer replication (or shared backend)
  so multi-replica deployments cannot multiply
  effective quotas by replica count — as a follow-on
  to the single-instance in-memory MVP, consistent
  with the staging requested for AI consumers and with
  closed follow-on
  [#929](https://github.com/praxis-proxy/praxis/issues/929).

### Non-Goals

- Replacing the existing request-level `rate_limit`
  filter's policy logic — this proposal is the counter
  store; admission/rejection behavior stays in
  consumer filters.
- Implementing token-budget reservation/reconcile
  semantics — that belongs to
  [00121](00121_token-rate-limiting.md)
  ([ai#121](https://github.com/praxis-proxy/ai/issues/121)).
- Connection-level (L4) rate limiting — tracked in
  [#123](https://github.com/praxis-proxy/praxis/issues/123) /
  [00123](00123_connection-rate-limiting.md).
- Overload Manager resource monitors and protective
  actions — tracked in
  [#145](https://github.com/praxis-proxy/praxis/issues/145).
- Defining the full project-wide state taxonomy and
  storage backend traits — that is proposal
  [#99](https://github.com/praxis-proxy/praxis/issues/99) /
  [00099](00099_stateful-proxy-state-management.md);
  this proposal consumes that model rather than
  redefining it.
- Shipping peer replication in the same MVP as local
  incr/decr+TTL — replication is in scope for the
  overall feature but explicitly staged after the
  single-instance store is validated.

## Why?

### Motivation

Resilience and traffic-management features under epic
[#163](https://github.com/praxis-proxy/praxis/issues/163)
need shared, bounded counters: rate limits, abuse
signals, and routing conditions all ask "how many /
how fast for this key?" Today each feature either
keeps a private in-memory map or waits on an ad-hoc
external store. That creates three problems:

1. **Duplicated and inconsistent bounds** — soft/hard
   caps, TTLs, and eviction differ by filter, so
   operators cannot reason about memory or fairness
   globally.
2. **No reusable consumer API** — AI token rate
   limiting and similar work need atomic incr/decr
   with TTL on a keyed counter; without a shared
   primitive each epic grows its own store (see the
   consumer note on
   [#155](https://github.com/praxis-proxy/praxis/issues/155)
   and the distributed-counter dependency called out
   in
   [00121](00121_token-rate-limiting.md)).
3. **Multi-replica under-enforcement** — when each
   instance tracks independently, a client can obtain
   roughly N times the configured budget across N
   replicas. Optional peer replication (or a shared
   backend) is required for correct multi-instance
   deployments; single-instance in-memory remains the
   zero-dependency default.

Competing proxies expose stick-table / shared-counter
primitives (e.g. HAProxy stick-tables, Envoy local
rate-limit state plus Redis/global options). Praxis
needs an equivalent building block under the
[#99](https://github.com/praxis-proxy/praxis/issues/99)
state model so rate limiting, abuse detection, and
routing share one bounded, observable store.

### User Stories

- As a platform operator, I want a configurable
  in-memory counter table with TTL and max size so
  that rate limiting and abuse detection cannot grow
  unbounded under high key cardinality.

- As an AI gateway developer, I want atomic
  increment/decrement with TTL on a keyed counter
  (single-instance is enough at first) so that
  request-count rate limiting can share a stable store
  without inventing a private map. Token-budget
  reservation remains 00121 on top of that store.

- As an SRE running multiple Praxis replicas, I want
  optional peer replication (or shared backend)
  counters so that a client cannot multiply its quota
  by the replica count.

- As a filter author implementing abuse detection, I
  want to track request, error, and byte counters by
  IP or string key so that routing conditions and
  guardrails can read the same facts.

- As an operator defining compound policies, I want
  multiple concurrent tracking keys per request
  (e.g. per-IP and per-tenant) so that one request
  updates several budgets in one store.

- As a stakeholder of stateful proxy work, I want this
  store to align with proposal
  [#99](https://github.com/praxis-proxy/praxis/issues/99)
  so that TTL, scoping, and backend choice stay
  consistent with sticky sessions and other stateful
  features.
