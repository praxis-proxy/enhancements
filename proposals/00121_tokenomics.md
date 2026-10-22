---
issue: https://github.com/praxis-proxy/ai/issues/121
discussion: https://github.com/praxis-proxy/ai/issues/121
status: proposed
repos:
  - policy
  - praxis
  - ai
authors:
  - shaneutt
  - jland-redhat
  - asaadbalum
graduation_criteria:
  - Internal and managed stores enforce tenant raw-token and weighted budgets and rate limits
  - Usage reporting works without budgets and preserves raw counts alongside any calculated consumption
  - Hard and soft rules account for usage, preserve trusted over-quota signals, and support optional alerts
  - One-budget selection, shared and reserved allocations, and drain and rollover pass concurrency tests
  - Managed consumption replay and acknowledgment preserve one accounting effect
  - Both stores pass shared fixtures for canonical usage and store-failure outcomes
  - Missing stores, outages, reloads, and lost usage have tested and documented behavior
  - Configuration examples, CloudEvents, logs, and rollout guidance are published
stakeholders:
  - jland-redhat
  - leseb
  - alexsnaps
  - maleck13
  - abdallahsamabd
  - araujof
  - terylt
related:
  - "00099"
  - "00210"
  - "00212"
  - "01042"
  - https://github.com/praxis-proxy/enhancements/issues/11
  - https://github.com/praxis-proxy/ai/issues/123
  - https://github.com/praxis-proxy/ai/issues/125
  - https://github.com/praxis-proxy/ai/issues/126
  - https://github.com/praxis-proxy/ai/issues/979
  - https://github.com/praxis-proxy/ai/issues/1241
origin:
  repo: ai
  issue: https://github.com/praxis-proxy/ai/issues/121
  file: 00121_token-rate-limiting.md
---

# Tokenomics

## What?

**Tokenomics** gives applications a budget for inference and controls how
quickly they spend it. Praxis enforces token, rate, and cost limits, reports
consumption, and emits optional alerts.

| Measure | Meaning |
| --- | --- |
| Tokens | Input and output usage, including cache and reasoning breakdowns |
| Rate | Requests or tokens over time |
| Cost | Tokens weighted by type, model, and capabilities, in a declared budget unit |

Count tokens directly, or assign weights so expensive models spend more of the
same budget. The unit can be credits, USD, or another operator-defined quantity.
Budget enforcement requires an explicit internal or highly-available store;
usage reporting also works on its own.

## Why?

AI inference platforms need to control who spends, how much, and how fast.
Realtime applications and background jobs share funding but need different
priority. Enforce that close to inference, and send usage back to a budget manager.

Praxis supplies enforcement and the contract. Operators choose their budget
manager and infrastructure. Simple deployments start with YAML and an internal store.

### User Stories

- **Shared funding:** As an application owner, I want realtime and batch
  workloads to spend from one budget, so I can manage their total allowance.
- **Protected funding:** As an application owner, I want to reserve part of
  that budget for a child tenant, so shared traffic cannot take its allocation.
- **Separate funding:** As an operator, I want a parent to define several
  budgets and assign one to each child, so experiments need not spend production's allowance.
- **Workload priority:** As an operator, I want parent-wide and child-specific
  rates, so batch jobs can run more slowly while realtime traffic gets headroom.
- **Rate behavior:** As an operator, I want rolling windows or controlled
  bursts, so I can match the traffic pattern my models can handle.
- **Tenant identity:** As an operator, I want tenant selection through trusted
  headers or authentication, so existing application identities work here too.
- **Isolation:** As an application owner, I want tenant selection to respect
  access controls, so another caller cannot spend my budget by changing a header.
- **Token budgets:** As a platform owner, I want to count tokens without
  assigning money values, so a single-model deployment stays simple.
- **Credit budgets:** As a platform owner, I want model and capability weights
  in my own credit unit, so I can allocate capacity independently of currencies.
- **Cost budgets:** As a platform owner, I want prices by model, capability,
  and token type, so expensive output and discounted cache usage count differently.
- **Consistent usage:** As a budget owner, I want providers' token breakdowns
  counted once, so the same usage consumes the same allowance in either store.
- **Simple cost management:** As an operator, I want to supply USD-per-token
  rates and receive USD consumption, so a small budget manager needs no conversion layer.
- **Usage without budgets:** As an integrator, I want raw tokens and capabilities
  without configuring budgets or prices, so I can analyze and price usage externally.
- **Reusable prices:** As an operator, I want shared prices and budget-local
  prices together, so common pricing is maintained once and exceptions stay local.
- **Explicit coverage:** As an operator, I want unmatched requests rejected
  and incompatible prices rejected during configuration, so usage cannot slip past a budget.
- **Dynamic prices:** As a budget manager integrator, I want to update prices
  without repricing work already admitted, so consumption remains explainable.
- **Hard and soft limits:** As an application owner, I want either rejection
  or continued accounting at a limit, so I can enforce policy or observe its impact.
- **Downstream handling:** As an operator, I want a trusted over-quota signal
  on forwarded requests, so downstream schedulers can deprioritize soft overages.
- **Optional alerts:** As an operator, I want thresholds in either enforcement
  mode and alerts on hard denials, so I can act when capacity runs low.
- **Renewal:** As a budget owner, I want to choose use-it-or-lose-it or rollover,
  so unused allowance follows my allocation policy.
- **Concurrent requests:** As a budget owner, I want in-flight usage reserved
  before dispatch, so concurrent requests cannot all spend the same available allowance.
- **Estimation:** As an operator, I want explicit reservation estimates and
  fallbacks, so requests without complete usage metadata still reserve capacity.
- **Streaming:** As an application owner, I want streaming and ordinary
  inference accounted for without buffering the full response.
- **Retries and missing usage:** As a budget owner, I want retries counted,
  settlements deduplicated, and unknown usage kept visible, so failures do not create free usage.
- **Accounting failures:** As an application owner, I want accounting failures
  to stop unsafe admission without replaying completed inference or breaking its response.
- **Simple deployment:** As a developer or small-deployment operator, I want
  static YAML budgets and local renewal without running another service.
- **Managed deployment:** As an operator, I want my budget manager to own
  persistence and high availability, so Praxis does not become another budget management product.
- **Reliable feedback:** As a budget manager integrator, I want replayable
  consumption records that I acknowledge after saving, so I can recover from interruptions.
- **Observability:** As an SRE, I want logs, metrics, and CloudEvents from the
  same accounting decisions, so I can explain denials, overages, and reporting gaps.
- **Data minimization:** As an application owner, I want usage records without
  prompts or credentials, so accounting does not become another copy of my request data.
- **Safe changes:** As an operator, I want invalid updates rejected and live
  usage preserved across reloads, so a typo or policy edit cannot refill a budget.

## How?

### Goals

- Give operators one clear API for tenants, budgets, costs, rates, and alerts.
- Support shared and protected allocations with predictable accounting.
- Keep local deployments useful and managed deployments independent of a store product.
- Reuse Praxis authentication, token extraction, rate limiting, state, and CloudEvents.
- Make usage and failure behavior visible to the budget manager and operator.

### Non-goals

- Building an upstream budget management service, billing system, or general ledger.
- Managing external store replication, failover, persistence, or deployment.
- Currency conversion, exchange-rate sourcing, or other financial pricing policy.
- Mixing accounting methods or units within a tenant, or charging several budgets per request.
- Arbitrary matching expressions, grandchildren, or scheduled policy changes in this iteration.
- Per-tool or per-image charges, provider-managed batch jobs, or a zero-overspend guarantee in this iteration.

> **API direction:** Named objects, references, and explicit variants keep the
> API growable. Lists leave room for later capabilities; validation rejects
> unsupported combinations today. Scope limits may change with demand.

### Ownership and Storage

Praxis resolves tenants, enforces policies, reserves capacity, and records usage.
The **budget manager** owns tenants' managed policies, prices, allocations,
reconciliation, durable history, and all high-availability concerns.

| Store | Configuration and ownership |
| --- | --- |
| **Internal K/V** | YAML supplies policies; Praxis maintains local accounting and renewal |
| **Highly-available store** | The budget manager supplies policies and operates the backing system; Praxis consumes the contract and records usage |

Topology and failover are opaque to Praxis. The integration must provide atomic
admission, idempotent settlement, and acknowledged consumption records; high
availability alone does not guarantee those semantics.

**A root tenant _must_ specify `store`**, naming a K/V backend from [state
management]. Children inherit it. Never default or fall back to memory.
Unavailable or untrustworthy accounting blocks admission, including for soft
budgets.

Internal accounting survives reloads, but not process restarts. Replicas have
independent balances. Managed-store durability and recovery are the budget
manager's responsibility and must be documented by the integration.

### Tenants and budget selection

A tenant represents an application, group, or other consumer. Select one identity
mode: trusted headers or authenticated attributes. Missing, unknown, duplicate,
or unauthorized identities fail. Header mode requires trusted ingress to replace
caller-supplied identity headers. Authentication mode reads verified attributes.

A parent defines a budget catalog. Each tenant serving requests selects
**exactly one** entry through `budget_refs`. A parent without a selection only
groups its children; requests addressed directly to it are rejected.

Children inherit the parent's store, accounting method/unit, and budget catalog,
and may add rates.
Parent rates cover all traffic in the hierarchy; child rates cover that child.
Every applicable hard rate limit must pass.

> **First iteration:** Parent and child use the same `Tenant` type. A child
> cannot define a store, accounting method, budgets, or children of its own. `budget_refs`
> is a list, but more than one entry is invalid. There is no first-match budget
> selection or automatic default budget.

#### Shared or reserved

Every budget reference explicitly selects its allocation mode:

- **Shared:** Spend from the budget's shared pool, alongside other shared users.
- **Reserved:** Protect a fixed portion of that budget for this tenant. Shared
  users cannot spend it, and this tenant cannot borrow from the shared pool.

Reserved portions inherit the budget's unit and renewal. Their sum cannot exceed
its allowance; the remainder funds the shared pool. Charge one allocation in one
budget. Hard enforcement checks that pool; soft overages stay attributed to it
without reducing another pool's recorded allowance.

### Accounting, costs, and matching

A root tenant selects one accounting method, inherited by its budgets and children:

| Method | Unit | Consumption |
| --- | --- | --- |
| `tokens` | Implicitly `tokens` | Normalized input plus output, without weights |
| `weighted` | Required identifier, such as `compute_credits` or `USD` | Token counts multiplied by configured per-token coefficients |

Praxis compares unit identifiers for consistency and includes them in reports.
It does not interpret currencies or fetch exchange rates. USD-per-token prices
and credits-per-token weights use the same arithmetic and rate table. A simple
manager can therefore use USD directly, without translating it to an intermediate unit.

> **API direction:** Future unit-specific shorthand can expand into this same
> weighted form, preserving exact coefficients without implicit conversion.

Every cost must match the tenant's accounting method and, when weighted, its
unit identifier exactly. A token-only cost in a weighted budget, or a credit cost
in a USD budget, is invalid even with soft enforcement. One method and unit per
tenant hierarchy keeps this iteration simple; currency conversion stays external.

Budgets may mix global `Cost` references and inline costs. Global edits affect
every referring budget; inline costs belong to one budget. New attempts use the
validated revision; work already admitted retains its original prices.

#### One matching cost

Every request under budget enforcement must match a cost in its selected budget.
Match the final model and capabilities, requiring all specified fields to agree.
Adding constraints makes a match more specific: model plus adapter beats model
alone. Reject duplicate or overlapping matches where neither is strictly more specific.
Declaration order never resolves a conflict.

An explicit `kind: any` is the catchall. Without one, an unknown model fails.
A token catchall permits new models and reports capabilities for later analysis.
A weighted catchall must supply coefficients in the budget's unit. Unmatched
usage is never silently free.

> **First iteration:** Exact matches and an explicit catchall are supported.
> Expressions and tenant-specific overrides of identical global matches will
> follow later. A more specific inline match already supports local exceptions.

#### Token weights and prices

Weighted costs supply input and output coefficients in budget units per token.
Optional cache-read, cache-write, and reasoning coefficients replace the base
coefficient for those subsets; omitted overrides use the input or output value.

Coefficients and weighted allowances use exact decimal strings with the same
fixed precision, regardless of unit. No currency-specific rounding or minor-unit
conversion applies. The example's USD coefficients are illustrative prices.

#### Canonical token usage

Normalize usage once in Praxis, before calculating consumption. Both stores
consume the same normalized result; neither interprets provider fields itself.

| Field | Meaning |
| --- | --- |
| `input` (`I`) | All input tokens, including cache reads and writes |
| `output` (`O`) | All output tokens, including reasoning |
| `cache_read` (`R`) | Input subset served from cache |
| `cache_write` (`W`) | Input subset written to cache, disjoint from cache reads |
| `reasoning` (`Q`) | Output subset used for reasoning |

For known counts, with `w_*` denoting the configured per-token coefficients:

```text
raw_tokens = I + O
weighted = (I - R - W) * w_input + R * w_cache_read + W * w_cache_write
         + (O - Q) * w_output + Q * w_reasoning
```

Token budgets and token rate limits use `raw_tokens`. Omitted cache coefficients
use `w_input`; omitted reasoning uses `w_output`.

For example, `I=100, R=20, W=10, O=50, Q=15` means **150 raw tokens**.
The weighted buckets are 70 ordinary input, 20 cache reads, 10 cache writes,
35 ordinary output, and 15 reasoning tokens.

- **Provider mapping:** Adapters explicitly distinguish included subsets from
  additional counts. Add standalone cache counts to input and standalone reasoning
  to output once, retaining them as subsets. Derive a missing aggregate only from
  complete, disjoint components; never infer inclusion from the numbers.
- **Validity:** Known counts must be nonnegative, with `R + W <= I` and `Q <= O`.
  Use checked arithmetic; unsupported overlap, overflow, or inconsistent counts
  leave affected accounting unresolved, never clamped or silently accepted.
- **Missing fields:** Unknown is not zero. With a known parent total, a missing
  subset still permits calculation when its coefficient equals its parent's;
  otherwise leave that calculation unresolved. Provider-defined absence may
  establish a known zero.
- **Provider totals:** Use them to check the mapped counts, never as another
  charge. Map extra categories explicitly or report them as unsupported; a
  mismatch leaves affected accounting unresolved.
- **Streaming:** Cumulative snapshots replace earlier snapshots. Genuine deltas
  accumulate once; a final snapshot replaces the accumulated count for its scope.
  Never sum snapshots or add a final total to preceding deltas.

Preserve original usage alongside canonical counts, their completeness, and a
normalization version. Missing data may prevent weighted settlement while still
allowing an exact raw-token total. Both stores use shared conformance fixtures.

#### Reporting without budgets

Use the existing token extraction and CloudEvents filters without enabling this
budget engine. No `tokenomics` configuration, budget, pricing match, or enforcement
store is required. Existing authentication and attribution policies still apply.

Always preserve raw token breakdowns and model/provider/capability metadata.
Budgeted traffic adds calculated consumption when available, its unit, and the
rate revision; reporting alone leaves those fields absent. Soft budgets still
perform accounting.
Reporting without budgets is an explicit pipeline choice, never a fallback for
failed matching or unavailable budget state.

### Budgets, rates, and alerts

Every budget and rate limit explicitly selects `enforcement`:

| Mode | At the limit |
| --- | --- |
| **Hard** | Reject admission when its reservation will not fit; return 429 with the limiting rule |
| **Soft** | Continue with a trusted over-quota signal, record the overage, and count all usage |

Budget availability is funded allowance minus consumption and in-flight holds,
within the selected allocation. Soft mode does not bypass identity, matching,
configuration, or store failures.

Rates count inference attempts or raw tokens. Choose a rolling `sliding_window`
or a `token_bucket` with capacity and **refill**. Chargeable retries count
again; denials consume no capacity and have separate metrics. Settle token
reservations against their original rate accounting, including late responses.

> **Refill**: Replenishes token-bucket capacity over time. Returning an unused
> request reservation happens separately, during settlement.

Alerts are optional in both modes. Budget thresholds compare a period's consumed
and reserved usage with its allowance, including carry, once per threshold per
period. Keep old holds attributed to their original period. Rate thresholds observe
window usage or bucket depletion and rearm below the threshold. `on_denial` reports
hard denials. Use logs, CloudEvents, and metrics for external Prometheus rules;
notification delivery stays external.

#### Soft over-quota signal

Soft mode retains trusted `tokenomics.over_quota` request metadata for later
filters. Set it from the same atomic admission result: true when this attempt's
reservation would not fit its selected soft budget allocation or any applicable
soft rate; false otherwise, including exact fits. Hard limits still apply, and
soft usage is always reserved and settled.

Optional `over_quota.header` forwards the literal value `true` to downstream
schedulers when over quota; otherwise the header is absent. Strip caller-supplied
and stale values on every path through Tokenomics, including below-quota and
hard-only traffic. Omitting this setting disables header projection, not trusted
metadata or accounting. Consumers may trust the signal only on this configured path.

Replaying an accounting operation returns its original decision. A new inference
attempt resets and reevaluates the signal, including retries and fallbacks.
Overage first discovered during settlement is recorded and reported; it cannot
annotate a request already forwarded. This signal is independent of optional alerts.

#### Renewal

Each budget explicitly chooses `drain` or `rollover`; there is **no default**.
Periods use a fixed duration and UTC anchor. Initialization grants only the
current period's allowance; the anchor does not create historical grants.

> **First iteration:** Calendar-month renewal, including a chosen start day,
> is outside this fixed-duration contract. `30d` means 30 days, not a calendar
> month. A future calendar-period option can define those boundaries explicitly.

- **Drain:** Discard positive, unused allowance at the boundary and grant the
  next period's allowance.
- **Rollover:** Carry positive, unused allowance and add the next grant.
  Optional `max_carry` caps the amount carried; omitting it permits uncapped carry.

After initialization, apply each boundary once, including missed boundaries in
order. Local admission checks renewal; timers help with maintenance. Reloads and
repeated callbacks cannot grant twice.

Renew allocations independently: reserved portions receive their fixed grants,
and the shared pool receives the remainder. Divide `max_carry`, if set, in those
same proportions. Round reserved shares down to the unit's precision; the shared
pool gets the remainder. With no reserved portions, it gets the whole cap.

Never renew away debt or treat pending holds as unused allowance. Holds retain
their original period. A later release from a drained period expires; a release
under rollover becomes available when resolved, with the carry cap applied at
the next boundary. Usage beyond an old hold reduces current availability.

### Configuration

This proposed API shows two local budgets, shared funding, both rate algorithms,
and global plus inline costs. It is not supported by today's release. Configure
authentication and routing through their existing Praxis interfaces.

```yaml
state:
  kv:
    - name: local_budgets
      kind: memory
      scope: global # Shared within this process, not across replicas.

tokenomics:
  version: v1alpha1
  identity:
    kind: headers
    tenant_header: x-tenant
    child_header: x-child-tenant
  estimation:
    kind: request
    fallback: {input_tokens: 4096, output_tokens: 1024}
  over_quota: {header: x-tokenomics-over-quota} # Optional request-header projection.
  costs:
    - name: sol
      match: {kind: exact, model: gpt-6-sol}
      pricing:
        kind: weighted
        unit: USD
        per_token:
          input: "0.000003"
          output: "0.000012"
          cache_read: "0.0000003"
    - name: astra
      match: {kind: exact, model: gpt-6-astra}
      pricing:
        kind: weighted
        unit: USD
        per_token: {input: "0.000010", output: "0.000040"}

  tenants:
    - name: commerce
      store: local_budgets # Required on every root; children inherit it.
      accounting: {kind: weighted, unit: USD}
      budgets:
        - name: daily
          limit: "1000"
          renewal:
            mode: drain # Required choice: drain or rollover.
            period: 24h
            anchor: "2026-01-01T00:00:00Z"
          enforcement: hard
          alerts: {thresholds: [80, 100], on_denial: true}
          costs:
            - {kind: reference, name: sol}
            - {kind: reference, name: astra}
            - kind: inline
              cost:
                name: sol-custom-adapter
                match:
                  kind: exact
                  model: gpt-6-sol
                  capabilities: {lora: commerce}
                pricing:
                  kind: weighted
                  unit: USD
                  per_token: {input: "0.000004", output: "0.000016"}
        - name: experiments
          limit: "50"
          renewal:
            mode: rollover
            period: 24h
            anchor: "2026-01-01T00:00:00Z"
            max_carry: "100"
          enforcement: soft
          alerts: {thresholds: [100]}
          costs:
            - {kind: reference, name: sol}

      # No budget_refs here: commerce groups children but serves no direct traffic.
      child_tenants:
        - name: realtime
          budget_refs:
            - budget: daily
              allocation: {mode: shared} # No allocation-mode default.
          rate_limits:
            - name: realtime-requests
              unit: requests
              algorithm:
                kind: token_bucket
                capacity: 100
                refill: {amount: 20, period: 1s}
              enforcement: hard
        - name: batch
          budget_refs:
            - budget: daily
              allocation: {mode: shared}
          rate_limits:
            - name: batch-tokens
              unit: tokens
              algorithm: {kind: sliding_window, limit: 50000, window: 1m}
              enforcement: hard
        - name: research
          budget_refs:
            - budget: experiments
              allocation: {mode: shared}
          # First iteration: children cannot declare budgets, stores, or children.
```

For protected funding, change realtime's allocation to
`{mode: reserved, amount: "200"}`. It then gets 200 of the daily allowance and
batch shares the remaining 800. This does not add another budget.

Authentication mode replaces `identity` with
`{kind: authentication, tenant_attribute: tenant, child_attribute: workload}`.
These are verified identity attribute names. Omitting the child header or
attribute field permits only direct parent identification; when configured,
an absent child value selects the parent, whose own budget reference must exist.

For a raw-token tenant, use `accounting: {kind: tokens}`, integer-valued budget
amounts, and `pricing: {kind: tokens}`. An explicit `match: {kind: any}` covers all models.

For a credit budget, use `accounting: {kind: weighted, unit: compute_credits}`.
Its costs use `pricing: {kind: weighted, unit: compute_credits, per_token: {input: "1", output: "4"}}`, for example. Supply coefficients in that unit;
relabeling a live USD balance does not convert it to credits.

Managed deployments use the same policy types. Connections and credentials stay
in [state management]; deployment configuration fixes each root's store binding.
The manager publishes complete revisions, including referenced costs. It does
not merge partial policies with local YAML. Tokenomics has no HA settings.

#### Rust types

These types describe the Tokenomics YAML shape; `state` uses [state management].
Semantic validation below is required before building an executable policy;
deserialization alone is not validation. The Rust blocks form one module.
Tagged variants keep choices explicit, following [Serde's enum representation].
Empty struct variants keep unknown-field rejection working for choices such as
`shared` and `tokens` too.

##### Tokenomics

The root holds reusable costs and the tenant catalog, plus the identity and
estimation modes used by this engine.

```rust
use serde::Deserialize;
use std::collections::BTreeMap;

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Tokenomics {
    pub version: SchemaVersion,
    pub identity: Identity,
    pub estimation: Estimation,
    pub over_quota: Option<OverQuota>, // Omit to keep the signal internal.
    #[serde(default)]
    pub costs: Vec<Cost>,
    pub tenants: Vec<Tenant>,
}

#[derive(Debug, Deserialize)]
pub enum SchemaVersion {
    #[serde(rename = "v1alpha1")]
    V1alpha1,
}
```

##### Tenant and BudgetRef

A tenant defines a catalog or selects from its inherited catalog. Optional root
fields allow the same type to represent children without duplicating the API.

```rust
#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Tenant {
    pub name: Name,
    pub store: Option<Name>, // Required at root; forbidden on children.
    pub accounting: Option<Accounting>, // Required at root; inherited by children.
    #[serde(default)]
    pub budgets: Vec<Budget>, // Only roots may define budgets.
    #[serde(default)]
    pub budget_refs: Vec<BudgetRef>, // At most one for now.
    #[serde(default)]
    pub rate_limits: Vec<RateLimit>,
    #[serde(default)]
    pub child_tenants: Vec<Tenant>, // One level for now; no grandchildren.
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct BudgetRef {
    pub budget: Name,
    pub allocation: Allocation, // Required; no shared/reserved default.
}

#[derive(Debug, Deserialize)]
#[serde(tag = "mode", rename_all = "snake_case", deny_unknown_fields)]
pub enum Allocation {
    Shared {},
    Reserved { amount: Amount },
}
```

##### Accounting

The method selects the calculation; the unit names the resulting quantity.
Weighted units are opaque identifiers, so adding a currency or credit scheme
does not require another enum variant.

```rust
#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum Accounting {
    Tokens {},
    Weighted { unit: UnitId },
}

#[derive(Debug, Deserialize)]
#[serde(transparent)]
pub struct UnitId(pub String);
```

##### Budget and Renewal

A budget sets the allowance, renewal, and matching costs. Its accounting method
and unit come from the parent tenant; renewal and enforcement are explicit.

```rust
#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Budget {
    pub name: Name,
    pub limit: Amount,
    pub renewal: Renewal,
    pub enforcement: Enforcement,
    pub alerts: Option<Alerts>,
    pub costs: Vec<CostSource>,
}

#[derive(Debug, Deserialize)]
#[serde(tag = "mode", rename_all = "snake_case", deny_unknown_fields)]
pub enum Renewal {
    Drain { period: Interval, anchor: UtcTimestamp },
    Rollover {
        period: Interval,
        anchor: UtcTimestamp,
        max_carry: Option<Amount>,
    },
}
```

##### Cost and CostSource

References reuse global costs; inline entries let a budget supply its own.
Both resolve to the same `Cost` type before matching.

```rust
#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Cost {
    pub name: Name,
    pub r#match: CostMatch,
    pub pricing: Pricing,
}

#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum CostSource {
    Reference { name: Name },
    Inline { cost: Cost },
}

#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum CostMatch {
    Exact {
        model: Option<String>,
        #[serde(default)]
        capabilities: BTreeMap<String, String>,
    },
    Any {},
}

#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum Pricing {
    Tokens {},
    Weighted { unit: UnitId, per_token: TokenRates },
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct TokenRates {
    pub input: Amount,
    pub output: Amount,
    pub cache_read: Option<Amount>,
    pub cache_write: Option<Amount>,
    pub reasoning: Option<Amount>,
}
```

##### RateLimit

Rates stay separate from funding. An explicit algorithm avoids making the same
`limit / period` syntax mean either a rolling cap or a burst allowance.

```rust
#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct RateLimit {
    pub name: Name,
    pub unit: RateUnit,
    pub algorithm: RateAlgorithm,
    pub enforcement: Enforcement,
    pub alerts: Option<Alerts>,
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum RateUnit { Requests, Tokens }

#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum RateAlgorithm {
    SlidingWindow { limit: u64, window: Interval },
    TokenBucket { capacity: u64, refill: Refill },
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Refill {
    pub amount: u64,
    pub period: Interval,
}
```

##### Identity and Estimation

Identity modes can grow without changing tenant definitions. Estimation reserves
input and output separately so token-type prices also work before dispatch.

```rust
#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum Identity {
    Headers {
        tenant_header: String,
        child_header: Option<String>,
    },
    Authentication {
        tenant_attribute: String,
        child_attribute: Option<String>,
    },
}

#[derive(Debug, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case", deny_unknown_fields)]
pub enum Estimation {
    Request { fallback: TokenEstimate },
    Fixed { tokens: TokenEstimate },
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct TokenEstimate {
    pub input_tokens: u64,
    pub output_tokens: u64,
}
```

##### Enforcement, OverQuota, Alerts, and scalar types

`OverQuota` optionally projects the trusted admission signal onto a request
header; alerts are separately opt-in. Amounts and intervals arrive as strings
for validation before conversion to checked integer arithmetic and durations.

```rust
#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Enforcement { Hard, Soft }

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct OverQuota {
    pub header: String,
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Alerts {
    #[serde(default)]
    pub thresholds: Vec<u16>, // Positive, increasing percentages; may exceed 100.
    #[serde(default)]
    pub on_denial: bool,
}

#[derive(Debug, Deserialize)]
#[serde(transparent)]
pub struct Name(pub String);

#[derive(Debug, Deserialize)]
#[serde(transparent)]
pub struct Amount(pub String);

#[derive(Debug, Deserialize)]
#[serde(transparent)]
pub struct Interval(pub String);

#[derive(Debug, Deserialize)]
#[serde(transparent)]
pub struct UtcTimestamp(pub String);
```

#### Validation and updates

Validate the whole policy, including referenced costs, before activation:

| Area | Required checks |
| --- | --- |
| Names and references | Unique root names, sibling names, budget names, and resolved cost names in their scopes; every reference resolves. Names are 1–63 lowercase ASCII letters, digits, `_`, or `-`, starting with a letter. |
| Tenants | Root store/accounting required; children inherit both and the catalog; no grandchildren; every leaf has exactly one budget reference and every other tenant has at most one. |
| Allocations | Explicit mode; positive reserved amounts; total reserved portions no greater than the budget allowance. |
| Methods and units | Costs match the root's method and exact unit identifier. Weighted identifiers match `[A-Za-z][A-Za-z0-9_.-]{0,62}`, are case-sensitive, and cannot be `tokens` (reserved for raw counts). No aliases or conversion. |
| Amounts | Nonnegative decimal strings with up to 12 fractional digits, scaled by `10^12` with checked `i128` arithmetic for every weighted unit; raw-token allowances are whole `u64` counts. Currency symbols, exponents, floats, and overflow are rejected. |
| Matches | At least one exact constraint, or explicit catchall; nonempty exact values; no duplicate or incomparable overlapping matches after resolving references. |
| Renewal and rates | Explicit renewal mode; valid UTC RFC 3339 anchor; positive integer durations with `ms`, `s`, `m`, `h`, or `d`; rate capacities/refills positive; `max_carry` only with rollover. |
| Identity and estimates | Valid, distinct header or attribute names; fallback/fixed estimates have a positive total and checked arithmetic. |
| Over-quota signal | Header, when configured, is a valid nonempty HTTP field name owned by Tokenomics. Reject case-insensitive collisions with identity, authentication, framing, hop-by-hop, reserved names, or other configured header writers. |
| Alerts | Thresholds positive, unique, and increasing; denial alerts only on hard rules. |

Reject duplicate YAML keys and non-string amounts before type deserialization;
reject unknown fields too. Invalid startup configuration fails; invalid updates
retain the last valid policy and report the error. Revalidate budgets affected
by global price changes. Reloads never grant fresh allowance.

In this iteration, changing a store, accounting method/unit, or rate algorithm, and renaming or
reassigning live budgets/allocations require explicit migration. Window/refill
changes must preserve existing rate debt. Period and allocation-size changes take
effect at a renewal boundary after validation. Price and limit updates keep
outstanding holds; reducing a limit below usage blocks subsequent hard admission.

### Request lifecycle

For traffic with budget enforcement enabled:

1. Resolve the tenant, its single budget/allocation, and final model/capabilities.
   Require one winning cost match.
2. Estimate input and output from request metadata, falling back per missing
   component, or use fixed estimates. Raw-token budgets reserve their sum;
   weighted budgets apply each side's highest coefficient until breakdowns are known.
3. Atomically reserve budget and applicable rate capacity before dispatch. A hard
   denial leaves no partial holds; soft rules still reserve and record usage.
   Publish the trusted over-quota signal and any configured header projection.
4. Pin policy, price, allocation, and period to the attempt ID. Chargeable retries
   and fallbacks get new IDs and admission, including rematching the model.
5. Normalize actual usage from ordinary or streaming responses without buffering
   the stream. Settle once, release unused capacity, and report consumption.
6. Fully release only work proven never dispatched. Missing usage, disconnects,
   and timeouts remain unresolved holds for reconciliation, never automatic refunds.

Estimates can underrun actual usage. Record the full charge and block subsequent
hard admission without capacity. In local mode, unresolved holds remain charged
conservatively and visible for operator reconciliation; timers cannot erase them.

> **Later iteration:** Model tokenizers can remove input estimation error.
> A zero-overspend mode also needs enforceable bounds on output and every weighted
> capability. Exact input counting alone cannot promise that guarantee.

#### Store failures

Failure handling depends on whether inference could already have happened.
A timeout means the accounting outcome is unknown, not that a write rolled back.
These rules apply to both stores and to hard and soft policies.

| Failure | Gateway behavior | Reservation / consumption |
| --- | --- | --- |
| Policy read fails with no usable validated revision, or accounting is untrustworthy | Return `503 tokenomics_unavailable`; do not dispatch | No new reservation |
| Reservation definitely failed without committing | Return `503`; do not dispatch | No hold |
| Reservation outcome is unknown | Resolve within the request deadline; otherwise return `503` without dispatch | Query/retry the same operation; cancel confirmed unused holds |
| Release fails for work proven never dispatched | Keep the original response or cancellation; do not dispatch | Retain the hold until cancellation is confirmed; retry idempotently |
| Settlement or its consumption-record write fails after dispatch | Preserve the inference response or stream; retry accounting separately | Resolve the same settlement; never release a hold or charge again speculatively |
| Export or acknowledgment fails after settlement | Preserve the response | Keep consumption charged and the record available for replay |

An existing validated policy remains usable only if admission can confirm its
revision and accounting are still valid. A capacity denial remains `429`.
Pre-dispatch `503` responses permit client retry with backoff because inference
was not sent; include `Retry-After` when a meaningful delay is available.

- **Operation identity:** Reserve, settle, and cancel are distinct operations
  on one attempt. Retries reuse each operation's ID and payload; conflicting
  reuse fails. A timeout never justifies retrying with a new ID.
- **Dispatch and cancellation:** Dispatch requires a confirmed, uncanceled
  reservation. Cancellation prevents delayed operations or dispatch from reviving
  the attempt. Recovery treats any attempt that could have been sent as dispatched.
- **After dispatch:** An accounting failure never triggers inference replay or
  turns a successful response into an error. Let admitted work finish; stop new
  admission wherever accounting is untrustworthy. Retain unresolved holds and
  retry accounting with backoff, logging failures for reconciliation.
- **Atomic settlement:** Budget/rate accounting, hold release, the deduplication
  result, and the replayable consumption record must succeed together. Integrations
  must prove this under failure; serial execution alone is insufficient.

CloudEvents delivery failure alone does not block admission when authoritative
accounting and its required replay capacity remain available.

> **First iteration:** A gateway crash before actual usage reaches durable storage
> can lose final counts even when the reservation survives. Retain recoverable
> holds for reconciliation and report the gap. Exact recovery requires durable
> usage capture beyond this MVP; internal accounting remains volatile across restarts.

### Feedback to the budget manager

The contract is versioned policy and accounting data, independent of the backing
store product. Keep writer ownership clear:

| Writer | Data |
| --- | --- |
| Budget manager | Tenant policies, costs, allocations, allowance revisions, and reconciled corrections |
| Praxis accounting operations | Holds, consumption, rate state, attempt deduplication, and consumption records |

The integration applies coherent policy revisions and atomic accounting operations.
Corrections and top-ups are idempotent operations, never remaining-balance
overwrites. Corrections and late gateway settlements resolve the same attempt
without charging twice.

Settlement atomically updates accounting and records consumption. The manager
persists replayed records idempotently, then acknowledges them. Acknowledgment
permits cleanup, never a refund. Bound backlog without deleting unacknowledged
records; stop admission early enough to preserve room for in-flight settlements.

The manager owns replication, recovery, retention, and fencing after failover.
Refuse operations while accounting is missing or untrustworthy. Praxis depends
on these semantics without knowing the topology. Internal mode follows the same
accounting rules but cannot replay durably across a restart.

### Logs, metrics, and CloudEvents

Reuse the existing CloudEvents path for consumption and optional alerts. Usage
records retain original usage, canonical counts and normalization version,
model/provider, capabilities, tenant attribution, stable event/attempt IDs, time,
and usage quality. Available budget accounting adds
budget/allocation, method, calculated amount, unit, policy/rate revision, and period.
Missing or unresolved accounting is marked as such, never reported as zero.
Shared IDs deduplicate records received through both paths; calculated consumption
never replaces the raw measurements used to derive it.

Log decisions, overages, unresolved usage, and delivery failures. Expose capacity,
export lag, and gaps without unbounded metric labels. Exclude prompts and credentials.
CloudEvents delivery is currently best-effort; it supplements the managed store's
acknowledged feed. Soft rules support observation without rejection for capacity.

### Lessons from existing projects

- **[Microsoft API Management]** separates token rates from quotas; its
  [FinOps sample] handles pricing and budget actions externally. Keep those
  operational roles distinct here too.
- **[Envoy ratelimit]** measures shadow-mode traffic. Soft mode must keep counting
  and reporting after a limit is exceeded.
- **[Kuadrant's reservation RFC]** tackles the race between checking capacity
  and reporting usage. Reserve before dispatch and settle afterward.

### Implementation plan

1. Review the YAML, types, validation, revisions, allocation/renewal behavior,
   canonical usage, failure outcomes, and consumption records with users and
   budget manager implementers.
2. Build one shared usage normalizer from the existing provider extractors.
   Cover included and standalone counts, missing fields, invalid totals, and
   streaming snapshots/deltas. Preserve original usage and the normalization version.
3. Build typed reserve, settle, and cancel operations with local accounting,
   following the store-failure matrix. Reuse identity, rates, state, and CloudEvents;
   include the trusted admission signal and sanitized header projection. Keep the
   feature experimental and accounting retries separate from inference.
4. Implement those same operations through the managed-store contract, including
   atomic settlement with its consumption record, idempotent retries, replay,
   acknowledgment, and recovery. The budget manager owns high availability.
5. Verify reporting without budgets, raw-token and generic weighted accounting,
   matching, unit safety, one-budget selection, shared/reserved pools,
   drain/rollover, alerts, streaming, concurrency, retries, missing usage, reloads,
   failover, and replay. Feed both stores identical normalized fixtures and assert
   equal consumption. Inject failures before/after dispatch, including lost
   acknowledgments and partial-write attempts; check response codes, reservation
   transitions, and that accounting failures never replay inference. Verify soft
   signals, exact fits, spoofed headers, and signal reset on new attempts. Publish
   examples and crash-recovery limits, following the [experimental guide].

[state management]: 00099_stateful-proxy-state-management.md
[experimental guide]: ../docs/experimental-phase.md
[Serde's enum representation]: https://github.com/serde-rs/serde-rs.github.io/blob/master/_src/enum-representations.md
[Microsoft API Management]: https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/api-management/llm-token-limit-policy.md
[FinOps sample]: https://github.com/Azure-Samples/AI-Gateway/blob/main/labs/finops-framework/finops-framework.ipynb
[Envoy ratelimit]: https://github.com/envoyproxy/ratelimit/blob/main/README.md#shadowmode
[Kuadrant's reservation RFC]: https://github.com/Kuadrant/architecture/blob/main/rfcs/0021-token-rate-limit-reservations.md
