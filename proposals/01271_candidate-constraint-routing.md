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
  - Praxis accepts PPE's typed candidate-constraint contract as the mechanism for restricting candidates.
  - The request handoff and the cluster-label configuration surface in Praxis are agreed.
  - The host vocabulary and ordering for cost_tier are defined or explicitly deferred.
---

# Enforce Policy Candidate Constraints During Candidate Selection

## What?

Make the candidate restrictions emitted by the Praxis Policy Engine (PPE)
effective when Praxis routes a request to a backend. A policy using APL
`restrict` must limit the candidate backends that can serve its request,
rather than merely reporting an allow decision while the constraint is
discarded.

Praxis will retain a non-empty constraint for the request lifetime, give each
logical candidate cluster operator-configured labels such as model, region,
site, and cost tier, and apply PPE's typed matching operation while selecting
a candidate. If no eligible candidate remains, `on_empty` will either reject
the request or restore the unconstrained candidates, as PPE specifies.

This proposal does not alter PPE semantics, add endpoint-level labels, or
change Praxis AI's `intelligent_route`. [AI #1282](https://github.com/praxis-proxy/ai/issues/1282)
remains related work that can later use the same established contract.

## Why?

PPE already evaluates APL `restrict` rules, combines every restriction that
fires for a request, and returns a typed `CandidateConstraintExtension` in
`PipelineResult.modified_extensions`. Praxis currently consumes the sibling
`raw_credentials` output from the same result, but does not consume the
candidate constraint. Its routing and load-balancing selection therefore has
no information with which to remove ineligible backends.

This creates a silent policy bypass. An operator can require that a request
be served only in a permitted region and choose fail-closed behavior when no
such candidate exists, yet a higher-scoring candidate outside that region can
still be selected. The policy loads and evaluates successfully, so neither
the policy author nor the operator receives an indication that the intended
restriction was not enforced.

Using PPE's typed output at the existing Praxis boundary prevents this silent
policy bypass without duplicating policy semantics. It establishes the host
contract that related work, including [AI #1282](https://github.com/praxis-proxy/ai/issues/1282), may later reuse.
