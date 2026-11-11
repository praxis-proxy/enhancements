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
