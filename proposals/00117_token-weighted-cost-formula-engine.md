---
issue: https://github.com/praxis-proxy/ai/issues/117
discussion: https://github.com/praxis-proxy/ai/issues/117
status: proposed
repos:
  - ai
authors:
  - mkoushni
graduation_criteria:
  - How? section defining evaluation inputs, override precedence, numeric and unit semantics, failure behavior, and the FilterContext result contract
  - Compatibility and migration plan for the token-type weighting delivered by ai#884
  - Validation plan covering formula configuration, provider usage shapes, fallback behavior, and downstream consumers
stakeholders:
  - shaneutt
  - asaadbalum
  - jland-redhat
  - noyitz
  - leseb
related:
  - 00121
  - https://github.com/praxis-proxy/ai/issues/118
  - https://github.com/praxis-proxy/ai/issues/577
  - https://github.com/praxis-proxy/ai/issues/893
  - https://github.com/praxis-proxy/ai/issues/1454
epic: https://github.com/praxis-proxy/ai/issues/78
---

# Token-Weighted Cost Formula Engine

## What?

Add a shared cost-calculation capability to Praxis AI that derives weighted
cost units from normalized provider token usage. A calculation can distinguish
input, output, prompt-cache read, prompt-cache creation, and reasoning tokens
instead of treating every token as having the same cost.

Praxis AI already extracts these token types and the experimental
`token_rate_limit` filter already applies static per-type weights during quota
reconciliation. That implementation is the compatibility baseline, not a
second system to replace. This proposal generalizes the calculation so one
configured result can be reused by rate limiting, metering, observability, and
future spend-budget filters.

Existing `token_rate_limit` configuration maps to the shared calculation
without changing its result. Each token type starts at weight `1.0`,
filter-wide `default_weights` overlay those values, and the matched rule's
`weights` overlay the filter defaults. An omitted per-rule weight therefore
inherits the filter default, and an omitted filter default inherits `1.0`.
Reconciliation must retain the current partitioning, fallback, saturation, and
rounding behavior while it moves to the shared result.

The selected calculation may vary by the trusted request and routing context,
including route, backend, provider, and effective model. Operators can define a
global fallback and narrower overrides. Selection must be deterministic when
several dimensions match, and a missing override must fall back rather than
silently disabling accounting.

The result is distinct from the provider's raw token counts. Downstream filters
receive both the derived value and enough bounded metadata to identify its
formula, unit, and evaluation outcome. Existing `token.*` metadata remains the
source of reported usage and is not overwritten by a weighted result.

Formula evaluation must preserve the accounting semantics already established
by token-type weighting:

- an absent provider field is different from a provider-reported zero;
- cache counts partition input tokens rather than adding to them;
- reasoning may be nested in output or reported additively, depending on the
  provider;
- incomplete, inconsistent, or overflowed usage cannot silently reduce a
  charge; and
- a response with only a total count still has an explicit, conservative
  fallback path.

The proposal does not require a particular expression language in its first
stage. CEL, a constrained expression format, and named built-in formulas are
candidate mechanisms for the follow-up design. The chosen mechanism must be
bounded, validated when configuration loads, and safe for the response hot
path.

### Goals

- Provide one reusable calculation contract over normalized input, output,
  total, cache-read, cache-creation, and reasoning token counts.
- Support global formulas with deterministic per-route, per-backend,
  per-provider, and per-model overrides.
- Preserve the shipped `ai#884` weight behavior as a compatible formula and
  avoid duplicating its overlap-safe token partitioning.
- Publish weighted cost and evaluation status through `FilterContext` for
  downstream quota, metering, logging, and metrics consumers.
- Keep raw provider token counts separately observable from derived cost units.
- Reject invalid formulas and non-finite or negative configuration at load
  time, with explicit runtime behavior for missing or unusable usage.
- Keep evaluation bounded in time, memory, and numeric range for streaming and
  non-streaming responses.

### Non-Goals

- Provider-specific token extraction or normalization, which belongs to the
  completed token-counting foundation (`ai#71` and `ai#885`).
- Reimplementing token-rate-limit ledgers, reservation, reconciliation, or
  enforcement from proposal `00121` and `ai#121`.
- Defining provider price catalogs, currency conversion, billing metrics, or
  invoice reproduction. Per-model monetary attribution remains `ai#118`.
- Implementing dollar-denominated budget windows or enforcement, which remains
  `ai#893`.
- Replacing the external metering transport or identity contract from
  `ai#577`; it is a downstream consumer of calculated cost.
- Guessing unavailable modality intersections or defining image, audio,
  duration, GPU-time, or other non-token costs. Modality-specific partitioning
  remains coordinated with `ai#1454`.
- Allowing formulas to read prompt contents, credentials, arbitrary headers,
  response bodies, or other unbounded request data.

## Why?

### Motivation

Raw token totals are a poor proxy for resource consumption and provider cost.
Output tokens can cost several times more than input tokens, prompt-cache reads
can be substantially cheaper, cache creation can be more expensive, and
reasoning tokens may have their own rate. The correct relationship also varies
by provider, backend contract, and model.

Praxis AI has already proven weighted reconciliation in `token_rate_limit`, but
that calculation is private to the limiter and configured only at filter and
rule scope. Other consumers still receive raw counts and would need to recreate
the same partitioning, fallback, rounding, and overflow rules independently.
That duplication would allow quota enforcement, metering events, access logs,
and future spend budgets to disagree about the cost of the same request.

A shared calculation contract makes the derived value consistent while keeping
reported usage intact for audit. Contextual overrides let an operator express
real deployment differences without cloning entire filter chains merely to
change a model's weights. An explicit formula identity and unit prevent a
weighted quota unit from being mistaken for a provider token total or a dollar
amount.

This is also a prerequisite for the broader cost-control work. Per-model cost
attribution (`ai#118`) needs a stable calculated input, and spend budgets
(`ai#893`) need the same actual cost at reconciliation that metering and
observability report. Establishing the calculation boundary first prevents
those consumers from embedding incompatible formula engines.

### User Stories

- As a **platform operator**, I want one global token-cost formula with
  provider, backend, route, and model overrides so that quota units reflect the
  workloads actually served.
- As a **FinOps engineer**, I want raw token counts and derived cost units kept
  separate so that chargeback records are explainable and auditable.
- As a **rate-limit operator**, I want existing static token weights to remain
  compatible while the same result becomes available to other filters.
- As a **metering integrator**, I want a bounded, named calculated value in
  request context so that usage events do not reimplement provider accounting.
- As a **budget-enforcement filter**, I want actual response cost evaluated by
  the same policy used for logs and metrics so that reconciliation cannot drift
  from reported spend.
- As a **gateway maintainer**, I want invalid or ambiguous accounting
  configuration to fail predictably rather than silently undercharge traffic.
