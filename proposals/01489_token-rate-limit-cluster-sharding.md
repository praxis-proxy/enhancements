---
issue: https://github.com/praxis-proxy/ai/issues/1489
status: proposed
repos:
  - ai
  - praxis
authors:
  - jordigilh
graduation_criteria:
  - Complete accounting-group and global-cap semantics are approved by maintainers
  - Shared-backend ownership and topology boundaries are approved
  - Generation, fencing, and existing-state compatibility rules are reviewed
  - A follow-up How? section defines the implementation requirements and design
stakeholders:
  - nerdalert
  - shaneutt
  - leseb
  - jland-redhat
related:
  - https://github.com/praxis-proxy/ai/issues/843
  - https://github.com/praxis-proxy/ai/issues/1421
  - https://github.com/praxis-proxy/ai/issues/1482
  - https://github.com/praxis-proxy/ai/issues/844
  - https://github.com/praxis-proxy/enhancements/pull/17
---

# Redis/Valkey Cluster sharding for token-rate-limit state

## What?

Define a production-safe accounting contract for running the token-rate-limit
shared backend on Redis Cluster and Valkey Cluster while preserving the
reservation and reconciliation semantics established by the existing token-rate
limit proposal [00121][token-rate-proposal].

### Current design

The current experimental filter matches ordered rules, reserves an estimate
before forwarding a request, and reconciles the same reservation against
provider usage after the response. State can be process-local or shared through
Valkey/Redis, and the shared path uses one atomic Lua operation for each
budget/bucket.

The current v1 key layout derives physical keys from the namespace, rule, and
bucket, then maintains namespace-level indexes. Those keys do not share one
canonical Cluster hash tag, and the current client connection is not a native
Cluster topology owner. This proposal changes placement and topology ownership,
not the reservation lifecycle.

### Complete accounting group

The atomicity boundary is a **complete accounting group**: all state that must
change atomically for one admission or settlement, including budgets, caps,
reservations, settled usage, and correctness-relevant indexes and counters.

Every key participating in one group uses one canonical flat or opaque
Redis/Valkey Cluster hash tag and therefore remains in one hash slot. The tag is
the Cluster placement mechanism; it is not the Redis Hash data type.

For example, an exact organization-wide cap covering two teams is one group:

```text
quota:{org-42}:global-cap
quota:{org-42}:team-x:active
quota:{org-42}:team-y:active
```

`team-x` and `team-y` are suffixes, not independent Cluster tags. The global cap
is a separate logical component of the same group and must not be copied into
each team ledger.

### Reservation and sharding boundary

Admission against an organization cap and a team budget is one atomic
operation: assess all applicable limits, deny without creating a reservation if
any limit rejects, or update all applicable state and return one handle.
Reconciliation uses the same group and atomically updates the state represented
by that handle. Separate application reads and writes are not equivalent.

Independent complete groups use different tags and may distribute across
Cluster primaries. A shared global cap or any other exact cross-group dependency
joins the groups; it cannot be copied, divided, or approximated. Joined state is
co-located on one slot, or the configuration is rejected for Cluster mode.

Groups are finite and operator-declared. Request data cannot create or select an
undeclared group, and missing or ambiguous trusted scope fails closed. Cluster
hash tags are not hierarchical; identities that form one group must first be
canonicalized into one flat or opaque tag.

### Backend and topology boundary

There is one named shared backend and one reserve/reconcile flow. Standalone,
Sentinel, and Redis/Valkey Cluster are topology modes of that backend. The
maintained cluster-aware client owns discovery, routing, failover, timeouts, and
lifecycle; consumers do not maintain shard maps or cross-shard coordinators.

Redis and Red Hat Valkey have the same support contract where qualification
passes. Separate datastore clusters are separate backend realms, not one
federated ledger. Asynchronous replication remains an accepted durability
limitation; this proposal does not claim RPO=0 or unconditional exactly-once
behavior.

### Generation and compatibility boundary

The Cluster contract uses an explicit generation and immutable accounting
bundle. Incompatible changes require an explicit quiescent handover, fencing,
and a supported state transition.

There is no mixed-generation serving, implicit reset, dual-writing, automatic
importer, or ambiguous mutation replay. Existing standalone keys are not
silently summed or reinterpreted to create a new joined cap. A transition must
preserve outstanding debt or establish a supported empty/naturally-aged state;
otherwise it is rejected.

### Goals

- Preserve exact reserve/reconcile accounting within each complete group.
- Distribute independent groups across Cluster primaries.
- Keep topology, client, failover, and lifecycle ownership behind one backend.
- Make unsupported cross-group accounting requirements explicit and fail closed.

### Non-Goals

- Application-managed shard maps or consistent hashing.
- A Praxis-owned cross-shard coordinator, escrow layer, or two-phase protocol.
- Divided, copied, or approximate global caps.
- A second reserve/reconcile implementation for Cluster.
- Mixed-generation serving, implicit reset, dual-writing, or ambiguous replay.
- Performance benchmarking or Limitador/Kuadrant compatibility, which remain in
  [ai#1482][ai-1482] and [ai#844][ai-844].

### Open questions for the follow-up design

- What configuration declares finite groups and maps trusted request scope to
  one of them?
- Which shared-backend ownership changes belong in Praxis core versus AI?
- What migration mechanism proves debt preservation or a supported
  empty/naturally-aged transition?
- What bounded retry behavior is safe for topology changes and ambiguous
  mutation outcomes, and which Redis/Valkey versions can claim the same
  contract?

## Why?

### Motivation

The current ledger performs multi-key atomic reserve, reconcile, and cleanup
operations. Redis/Valkey Cluster can execute a Lua operation atomically only when
all supplied keys map to one hash slot. Splitting the current layout causes
cross-slot failures; putting the entire namespace under one tag removes useful
sharding; and dividing a global cap changes its meaning.

The complete-accounting-group boundary is therefore the smallest unit that can
preserve exactness while allowing independent groups to scale across primaries.
An organization-wide cap intentionally makes all governed teams one atomic
group.

### Rejected alternatives

- **Enable Cluster without changing the key contract:** cross-slot failures or
  non-atomic client-side decomposition.
- **Put the entire namespace in one tag:** preserves atomicity but provides no
  useful sharding.
- **Divide a global cap across primaries:** weakens the configured limit.
- **Use consumer-owned shard maps or coordinators:** duplicates topology
  ownership and creates cross-shard correctness problems.
- **Silently reset or reinterpret existing state:** can erase reservations or
  change the meaning of settled debt.

### Qualification context

Existing qualification establishes the reserve/reconcile and single-primary HA
behavior for Redis and Red Hat Valkey and exposes the Cluster same-slot
constraint. The sharding implementation must demonstrate real multi-primary
distribution without weakening typed atomic ledger semantics.

[ai-1482]: https://github.com/praxis-proxy/ai/issues/1482
[ai-844]: https://github.com/praxis-proxy/ai/issues/844
[token-rate-proposal]: https://github.com/praxis-proxy/enhancements/blob/main/proposals/00121_token-rate-limiting.md
