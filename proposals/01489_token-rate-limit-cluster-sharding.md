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
shared backend on Redis Cluster and Valkey Cluster while preserving the exact
reserve/reconcile behavior of the existing ledger.

This proposal establishes the semantic boundary that implementation and
qualification must satisfy. It does not select the final public YAML/API
fields or commit to a particular client-library implementation.

### Current design

The current experimental `token_rate_limit` filter has one reservation
lifecycle, with two admission algorithms available per rule:
`sliding_window` and `token_bucket`. Ordered rules select a budget from static
header matches; a rule reserves a fixed `reserved_tokens` estimate against
either one global bucket or a bucket derived from a verified authenticated
subject.

State is either process-local memory or a shared Valkey/Redis backend. The
shared backend is configured once for the filter and its cached multiplexed
connection is reused by the rules. Admission executes one Lua `EVAL` operation
with the budget state, active reservations, settled usage, cleanup indexes,
active counts, and reservation sequence supplied as explicit keys. A Valkey
reconciliation is queued after the response so the response path does not wait
for a second network round trip; in-process reconciliation is synchronous.

At the end of the response stream, the filter uses provider-reported
`token.total` when available and otherwise settles at the reserved estimate.
The reservation handle, bucket key, and originating rule are carried through
request metadata so settlement targets the same ledger entry. Admission
backend failures fail closed rather than forwarding an unaccounted request.

The committed backend's v1 key layout derives physical keys from the namespace,
rule, and bucket, then adds namespace-level indexes. Those keys do not share one
canonical Redis/Valkey Cluster hash tag, and the current Redis client connection
is not the proposed native Cluster topology owner. Consequently, this proposal
changes placement and topology ownership while preserving the reservation
lifecycle and typed ledger operations described above.

### Scope

There is one token-ledger path:

```text
request -> shared backend reserve -> provider -> same handle reconcile
```

The filter does not choose a separate HA or sharding implementation. One named
shared backend owns the maintained datastore client, topology discovery and
recovery, timeouts, atomic operations, and lifecycle. Standalone, Sentinel, and
Redis/Valkey Cluster are topology modes of that backend.

This proposal covers native Redis/Valkey Cluster sharding for the token-rate-
limit backend. Sentinel HA is tracked by [ai#1421][ai-1421], performance
qualification by [ai#1482][ai-1482], and Limitador/Kuadrant compatibility by
[ai#844][ai-844]. [ai#843][ai-843] remains the parent tracking issue.

### Complete accounting groups

The atomicity boundary is a **complete accounting group**. A group contains
every state component that must change atomically for one admission or
settlement, including the applicable budgets and caps, active reservations,
settled/window usage, cleanup indexes, fingerprints, sequences, and any
correctness-dependent counters.

All physical keys participating in one group use one canonical flat or opaque
Redis/Valkey Cluster hash tag and therefore remain in one hash slot. The tag is
the Cluster placement mechanism; it does not require the Redis Hash data type.
Strings, hashes, and sorted sets may remain separate keys when they share the
same tag and are supplied explicitly to the same atomic operation.

For example, an exact organization-wide cap covering two teams is one group:

```text
quota:{org-42}:global-cap
quota:{org-42}:team-x:active
quota:{org-42}:team-y:active
```

`team-x` and `team-y` are key suffixes, not independent Cluster tags. The
global cap is a separate logical component of the group; it must not be copied
into every team ledger because those copies could diverge.

### Reservation and reconciliation semantics

Reserving against an organization cap and a team budget is one atomic
operation:

1. read and prune the relevant organization and team state;
2. assess the estimate against every applicable limit;
3. deny without creating a reservation if any limit rejects; or
4. update all applicable state and return one reservation handle.

Reconciliation uses the same complete group and atomically updates the global
and team state represented by that handle. The application must not perform a
separate global read followed by a team read or write, because another request
could interleave between those commands.

### Sharding boundary

Independent complete groups use different canonical tags and may distribute
across different Cluster primaries. A shared global cap or any other exact
cross-group atomic dependency joins the groups; it cannot be copied, statically
divided, or approximated. The joined state is co-located on one slot, or the
configuration is rejected for Redis/Valkey Cluster.

Redis/Valkey Cluster is one logical deployment containing multiple primary/
replica shards. The maintained cluster-aware client receives seed nodes,
discovers the datastore slot map, and routes commands. The application does not
maintain a per-shard map, search across separate clusters, or federate separate
backend realms into one ledger.

Hash tags are not nested. In a key such as
`quota:{group1}:{group2}:state`, only `group1` selects the slot. If multiple
identities form one atomic group, they must first be canonicalized into one
flat composite or opaque digest tag. Independent groups cannot be joined by one
atomic operation.

Groups are finite and operator-declared. Request data cannot create a new group
or select an undeclared group. Missing or ambiguous trusted scope fails closed.

### Generation and compatibility boundary

The Cluster contract uses an explicit generation and immutable accounting
bundle. The bundle identifies the approved group membership, placement
derivation, and accounting semantics. Owners validate the approved generation
before activation and mutation.

Incompatible changes require an operator-controlled handover with the following
observable properties:

```text
quiesce old writers -> drain requests and settlements -> fence old access
-> prove supported state transition -> activate the new generation
```

There is no mixed-generation serving, implicit reset, dual-writing, automatic
importer, or ambiguous mutation replay. Existing standalone keys are not
silently summed or reinterpreted to create a new joined global cap. A transition
must preserve outstanding debt through a supported conversion or establish a
supported empty/naturally-aged state; otherwise it is rejected.

Asynchronous replication remains an accepted durability limitation. This
contract does not claim RPO=0 or unconditional exactly-once behavior after a
primary failure.

### Goals

- Preserve exact reserve/reconcile accounting within each complete group.
- Provide genuine distribution of independent groups across Cluster primaries.
- Keep all state participating in one atomic operation in one Cluster slot.
- Keep topology, client, failover, and lifecycle ownership behind one shared
  backend.
- Make unsupported cross-group accounting requirements explicit and fail closed.
- Keep Redis and Red Hat Valkey qualification symmetric where the same contract
  is claimed.

### Non-Goals

- Application-managed shard maps or consistent hashing.
- A Praxis-owned cross-shard coordinator, escrow layer, or two-phase protocol.
- Divided, copied, or approximate global caps.
- Moving the entire namespace into one slot as a substitute for sharding.
- A second reserve/reconcile implementation for Cluster.
- Mixed-generation serving, implicit state reset, dual-writing, or ambiguous
  mutation replay.
- Dedicated-hardware performance benchmarking, tracked by [ai#1482][ai-1482].
- Limitador/Kuadrant compatibility, tracked by [ai#844][ai-844].

### Open questions for the follow-up design

- What exact configuration shape declares finite accounting groups and maps a
  trusted request scope to one of them?
- Which shared-backend ownership changes belong in Praxis core versus the AI
  implementation?
- What migration tooling proves debt-preserving conversion or the supported
  empty/naturally-aged transition?
- What bounded retry behavior is safe for `MOVED`, `ASK`, failover, timeout, and
  ambiguous mutation outcomes without replaying a mutation unsafely?
- Which Redis and Red Hat Valkey versions can claim the identical contract after
  qualification?

## Why?

### Motivation

The current token-rate-limit ledger performs multi-key atomic reserve,
reconcile, and cleanup operations. Its state includes more than a single
budget counter: reservations, settled usage, indexes, fingerprints, sequences,
and operational state participate in the accounting lifecycle.

Redis/Valkey Cluster executes a Lua operation atomically only when all supplied
keys map to one hash slot. Distributing the current keys across slots causes
cross-slot or non-local-script failures. Placing the complete namespace under
one tag preserves atomicity but removes meaningful sharding. Copying or dividing
a global cap across primaries changes the contract from one exact cap to an
approximation.

The complete-accounting-group boundary is therefore the smallest unit that can
preserve correctness while allowing independent groups to scale across
primaries. An organization-wide cap intentionally makes all of its governed
teams one atomic group; that is the required trade-off for exactness.

### Rejected alternatives

#### Keep the existing key layout and enable Cluster mode

This produces cross-slot failures or forces the client to decompose one
accounting operation into multiple non-atomic operations.

#### Put every token-rate-limit key in one tag

This preserves atomicity but makes the entire backend one Cluster slot and does
not provide useful sharding.

#### Divide a global cap across primaries

This changes the meaning of the configured cap and can admit more than the
operator's declared limit.

#### Maintain a consumer-side shard map or coordinator

This duplicates topology ownership, creates cross-shard correctness and failover
problems, and conflicts with the shared-backend direction in
[enhancements PR #17][enhancements-17].

#### Silently reset or reinterpret existing state

This can erase outstanding reservations or change the meaning of settled debt.
Generation changes therefore require an explicit fenced handover and a proved
state transition.

### User stories

- As a platform operator, I need one exact cap across all teams in an
  organization so that sharding does not weaken enforcement.
- As a platform operator, I need independent organizations to distribute across
  Cluster primaries so that one global hot slot does not limit the whole fleet.
- As an SRE, I need one named backend to own clients, topology recovery,
  timeouts, and failover so that consumers do not implement inconsistent HA or
  sharding behavior.
- As an operator changing accounting configuration, I need an explicit fenced
  generation handover so that reloads cannot mix incompatible ledgers or erase
  outstanding usage.
- As a maintainer, I need unsupported cross-group requirements rejected rather
  than silently approximated so that the accounting contract remains auditable.

### Qualification context

The existing qualification established the reserve/reconcile behavior and
single-primary HA behavior for Redis and Red Hat Valkey, while also exposing the
Cluster same-slot constraint. The sharding work must demonstrate actual
distribution across multiple primaries and must retain the typed atomic ledger
semantics rather than substituting generic counters or client-side composition.

The proposal is intentionally separate from performance qualification and
Limitador/Kuadrant compatibility. Those concerns remain tracked by the related
issues above.

### Approval requested

Maintainers are asked to approve:

1. the complete-accounting-group boundary;
2. organization/global caps joining one atomic group and one Cluster slot;
3. one shared backend and one reserve/reconcile flow across topology modes;
4. no application shard map, cross-slot coordinator, approximation, or
   ambiguous mutation replay; and
5. immutable generation approval with explicit fenced handover for incompatible
   changes.

[ai-843]: https://github.com/praxis-proxy/ai/issues/843
[ai-844]: https://github.com/praxis-proxy/ai/issues/844
[ai-1421]: https://github.com/praxis-proxy/ai/issues/1421
[ai-1482]: https://github.com/praxis-proxy/ai/issues/1482
[enhancements-17]: https://github.com/praxis-proxy/enhancements/pull/17
