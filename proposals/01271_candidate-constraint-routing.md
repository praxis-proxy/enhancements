---
issue: https://github.com/praxis-proxy/praxis/issues/1271
discussion:
status: proposed
repos:
  - praxis
authors:
  - yehuditkerido
stakeholders:
  - araujof
graduation_criteria:
  - The typed request-state handoff from PPE's candidate constraint to Praxis's eligible logical-cluster list is agreed.
  - The cluster-label configuration surface and the behavior for no constraint, deny, and fallback are agreed.
  - The host vocabulary and ordering for cost_tier are defined or explicitly deferred.
---

# Publish Policy-Eligible Candidates for Downstream Routing

## What?

Make the candidate restriction emitted by the Praxis Policy Engine (PPE)
available as request-scoped eligibility information for downstream routing. A
policy using APL `restrict` must no longer have its non-empty result silently
discarded after policy evaluation.

Praxis will retain a non-empty constraint for the request lifetime, give each
logical candidate cluster operator-configured labels such as model, region,
site, and cost tier, and apply PPE's typed matching operation to every
declared logical candidate. It will publish the eligible logical-cluster names
for a downstream routing consumer. If no eligible candidate remains,
`on_empty` will either reject the request or restore the unconstrained
candidates, as PPE specifies.

This proposal does not alter PPE semantics, add endpoint-level labels, or
choose the final logical cluster or endpoint. A downstream router must honor
the published eligible list to provide end-to-end enforcement; that consumer
remains outside this proposal.

## Why?

PPE already evaluates APL `restrict` rules, combines every restriction that
fires for a request, and returns a typed `CandidateConstraintExtension` in
`PipelineResult.modified_extensions`. Praxis currently consumes the sibling
`raw_credentials` output from the same result, but does not consume the
candidate constraint. The information is therefore lost before a downstream
router has any opportunity to use it when choosing a logical cluster.

This leaves Praxis without the typed request-state and cluster-label contract
that a router needs to honor policy restrictions. For example, a rule can
require an EU region, but Praxis currently cannot publish the corresponding
EU-only candidate list for a downstream router to consume.

Using PPE's typed output at the existing Praxis boundary establishes that
contract without duplicating policy semantics. It makes the constraint
available to a later router consumer while keeping the ownership of final
routing and endpoint selection outside Praxis.

## How?

### Request-flow contract

The design introduces the experimental, off-by-default
`candidate-constraints` feature and one additional mode of the existing HTTP
`load_balancer`: `cluster_source: policy_candidates`.

```text
policy filter
  -> stores PPE CandidateConstraintExtension in private request state
  -> policy_candidates examines every declared logical cluster
  -> publishes PolicyEligibleCandidates([logical cluster names])
  -> a downstream router chooses one eligible logical cluster
  -> that router's endpoint selection chooses an endpoint in the cluster
```

For example, PPE can restrict a request to `region: eu`. Given five declared
logical clusters—`eu_fra`, `eu_ams`, `us_virginia`, `us_oregon`, and
`us_texas`—the Praxis publisher exposes only `[eu_fra, eu_ams]`. It does not
choose between those two names and does not select an endpoint. That later
choice remains the downstream router's responsibility and is outside this
Praxis proposal.

The publisher is a top-level, unconditional step before any cluster or
endpoint selection. It cannot have request conditions and cannot be placed in
a branch: otherwise a request for which PPE produced a restriction could skip
publication and a later consumer could see an unrestricted candidate set.

### Typed handoff from policy

After each successful policy dispatch path—HTTP authorization, JSON-RPC, and
LLM—the policy filter reads the non-empty
`PipelineResult.modified_extensions.candidate_constraint`, alongside the
existing `raw_credentials` handling. It stores the typed PPE value in a
private request extension and exposes it read-only through
`HttpFilterContext::policy_candidate_constraint()`.

This is intentionally not an HTTP header. A client cannot forge, replace, or
inspect the restriction by sending request headers. The configured
`policy_candidates` step consumes the private value later in the same request.

### Candidate labels and matching

Each HTTP logical cluster gains an operator-configured `http.labels` map. The
labels describe the whole logical cluster, not one endpoint. If two endpoints
have different region, model, site, or cost properties, they must be modeled
as separate logical clusters.

The label vocabulary remains open because PPE defines the meaning of constraint
keys. Praxis carries the map from parsed `ClusterHttpOptions` into the runtime
`ClusterEntry`, so matching does not parse YAML during a request. The one
host-owned input is `cost_tier_order` on `policy_candidates`, ordered from
cheapest to most expensive, for example `[standard, premium]`. Praxis turns
that order into the rank PPE's typed matcher requires; PPE does not define tier
names or their order.

For every declared candidate, Praxis calls PPE's
`CandidateConstraintExtension::accepts(labels, tier_rank)`. Praxis does not
reimplement region, model, site, custom-label, or cost-tier semantics. If
policy produced no constraint, every declared candidate is published.

### Empty candidate results

When PPE produced a restriction but no declared cluster matches it, Praxis
uses PPE's existing `on_empty` behavior:

- `deny` rejects the request with HTTP 403 before an upstream is contacted.
- `fallback` republishes the original declared candidates for the downstream
  router.

These modes are intentionally resolved by the Praxis publisher, while the
eventual selection of one eligible logical cluster remains outside Praxis.

### Eligible-list read contract

Downstream filters read the published names through
`HttpFilterContext::policy_eligible_clusters() -> Option<&[Arc<str>]>`,
available with `candidate-constraints`. This accessor borrows the list
read-only; publication remains internal to the Praxis filter crate.

`None` means no eligible list has been published for this request. It must
not be interpreted as proof that PPE emitted no restriction. `Some` contains
a non-empty list after the publisher has resolved `on_empty` as described
above. When there is no PPE constraint, the publisher still publishes all
declared candidates as `Some`, rather than leaving the value absent.

### Implementation plan and validation

The implementation:

- Add the feature wiring in the Core, Filter, and Server crates, and include
  the feature in the feature-check matrix.
- Add HTTP-cluster labels and carry them into the load balancer's runtime
  cluster entries.
- Preserve the PPE constraint in request-scoped extensions and expose the
  read-only context accessors for the constraint and eligible-name list.
- Add `policy_candidates` matching and empty-result behavior to the HTTP load
  balancer.
- Add pipeline validation that distinguishes the publisher from an
  endpoint-selecting load balancer and rejects conditional, branched, late, or
  duplicate publishers.
- Add unit tests for all three policy dispatch paths; one eligible candidate;
  no eligible candidates with `deny` and `fallback`; cost-tier ordering; and
  the invalid pipeline shapes above.
- Add an experimental example configuration and a functional integration test
  for that example, then regenerate the examples README.

The acceptance tests must establish the Praxis-side contract, not pretend that
a downstream router already consumes it. In particular, they verify the
published eligible names and fail-closed `deny` response; final logical-cluster
selection is outside this Praxis design.
