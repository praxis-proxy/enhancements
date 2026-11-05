# Draft: PR description + review comments — tokenomics budget first pass

Status: draft, not yet posted.

---

## PR

**Title:** tokenomics: first-pass design review feedback on local budget enforcement

**Body:**

This PR starts the first pass of review feedback on the tokenomics proposal (00121), focused on the budget enforcement section. Overall direction is sound — normalized usage reporting, canonical token accounting, and the adapter contract are clearly valuable and well-scoped.

The main concern raised here is the scope of **local budget enforcement**: storing allowances, renewal periods, rollover, reserved allocations, and alert thresholds in static YAML (sourced from ConfigMaps) makes Praxis itself a budget management system, which the doc's own non-goals say it should not become.

This PR does not touch usage reporting, cost matching, or rate limits — those stay untouched. Comments are limited to the budget config surface.

Key open questions are in the review comments. Summary:

- ConfigMap-as-source-of-truth gives budget control to anyone with cluster config write access, with no per-budget authorization, audit trail, or history.
- Lifecycle operations (mid-period refill, limit adjustment, deletion mid-cycle) are unspecified.
- Renewal semantics across cycle boundaries and reloads leave too many undefined behaviors.
- The local store is per-process memory, so balances aren't shared across replicas — what is being enforced?

Suggestion: cut budgets from the first iteration, keep usage reporting + cost matching + rate limits, and define a thin integration contract (quota check / usage report callbacks) that an external budget manager can implement. Local enforcement can return later as a reference implementation of that contract.

---

## Review comment 1 — anchored on `tenants:` block (00121 lines 531–568)

**[Major — design]** I don't think we should support budgets directly in this iteration. This config block turns Praxis into a budget management product, which contradicts the doc's own non-goals ("not building an upstream budget management service... not the source of truth"). Some of the questions this shape forces us to answer:

1. **Mid-cycle lifecycle.** How does an operator refill one user's budget halfway through a period? How is a limit *adjustment* applied — does the delta apply immediately, at next renewal, or pro-rata? What happens to a budget that's deleted or renamed while requests are in flight?
2. **Cycle boundaries.** A request admitted at 23:59 and settled at 00:01 — which cycle pays? Does a renewal mid-request drain reserved funds, and can that push a cycle negative?
3. **Constant change.** ConfigMaps get edited, rolled back, and hot-reloaded. Which revision settles an in-flight request? What does a rollback do to balances already spent under a higher limit?
4. **Ownership and authorization.** Whoever can write this ConfigMap controls the money. There's no per-budget RBAC, no audit log of who changed what, and no history of consumption over time. This data is mutable, transactional, and queryable — that's a database workload, not declarative cluster config. Wouldn't a traditional DB behind a small budget-manager service make more sense?
5. **The store isn't shared.** `local_budgets` is `kind: memory, scope: global` — per-process. Two replicas each believe they have the full allowance. What is local enforcement actually enforcing across replicas?
6. **Rollover accumulation.** `max_carry` is optional — uncapped carry compounds forever. And what happens to a child's reserved allocation when the child is removed mid-cycle — does it return to the shared pool immediately?
7. **Soft enforcement.** `experiments` is soft with `thresholds: [100]` — overage is unbounded and only alerts. Who receives the alert, is it deduplicated, and what's the escalation path?
8. **Settlement drift.** Estimated (admitted) vs actual (reported) usage will diverge — retries, provider-side corrections, refunds. What reconciles the balance after the fact?

Proposal: keep usage reporting, cost matching, and rate limits (all clearly in scope), and replace this block with a narrow integration contract — check-quota and report-usage hooks an external budget manager implements. Local budgets, if we still want them, become a reference implementation of that contract in a later iteration.

---

## Review comment 2 — anchored on `store: local_budgets` (00121 lines 498–502, 533)

**[Major]** `store: local_budgets` with `kind: memory` means balances live in one process. The doc already says "replicas have independent balances" — so a 2-replica deployment effectively doubles every allowance. Either local enforcement requires a real shared store (etcd/Redis/DB) — which is a distributed-consistency problem we'd be signing up for — or it will work locally but it is not going to work in a scalable distributed environment. Both argue for pushing budget state out of Praxis.

---

## Extra questions (optional, hold in reserve)

- Anchor far in past: `2026-01-01` anchor with 24h period — engine must derive current cycle; what about anchors years back, timezone/DST?
- Alerts thresholds `[80, 100]` — percent of what, limit or remaining? Fires once or every request past threshold?
- Scale: hundreds of tenants × budgets × inline costs in one YAML blob — ConfigMap 1MB limit, reviewability.
- No budget without catchall cost = unknown model fails — operational coupling between pricing table completeness and availability.
