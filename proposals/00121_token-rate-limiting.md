---
issue: https://github.com/praxis-proxy/ai/issues/121
discussion: https://github.com/praxis-proxy/ai/issues/121
status: proposed
experimental_impl: https://github.com/praxis-proxy/ai/pull/796
experimental_exempt: true
experimental_exempt_reason: "AI filter and shared-backend configuration schema are implemented behind an experimental Cargo feature"
repos:
  - praxis
  - ai
authors:
  - shaneutt
  - jland-redhat
graduation_criteria:
  - How? section with requirements and design
  - Current experimental API reconciled with the target design
  - Shared-backend failure and high-availability requirements documented
  - Trusted quota-key contract reviewed by Praxis and AI stakeholders
  - Standalone single-instance and multi-instance qualification documented
stakeholders:
  - jland-redhat
  - leseb
  - mkoushni
  - crstrn13
  - eoinfennessy
  - alexsnaps
related:
  - 00099
  - 00119
  - 00210
  - 00211
  - 00212
  - 00214
  - 00216
  - 00220
origin:
  repo: ai
  issue: https://github.com/praxis-proxy/ai/issues/121
  file: 00121_token-rate-limiting.md
---

# Tokenomics + Token Rate Limiting

## What?

Token rate limiting adds quota enforcement denominated
in tokens to Praxis. Request-count quotas cannot
express constraints like "this team may consume 1M
tokens per hour" because a single LLM call can vary
wildly in terms of token cost. This capability lets
operators define budgets in the unit that actually
drives inference cost.

The system requires reservation-based admission:
requests are admitted by reserving an estimated cost up
front, then the reservation is reconciled against
actual usage reported by the provider after the
response completes. The estimation method must be
configurable because different deployments have
different cost models.

Different token types must be accounted for separately
with configurable weights. A cached input token does
not carry the same cost as an uncached token, and
quotas should reflect that difference. We also need to
support multiple modular strategies for what to do
about input token counting, output token counting,
estimation, what to do with unused tokens, etc.

### Goals

**Must have (MVP):**

- **[M1]** Bucket `Rules` which apply a `TokenBudget`
  based on a specific condition such as a header (or,
  by default, treats all requests under a `default`
  rule).
- **[M2]** Reservation-based admission: admit requests
  while reserving an estimated token cost, then
  reconcile count based on usage retrieved from the
  response.
- **[M3]** Configurable estimation: allow operator to
  define how cost is estimated based on request
  metadata.
- **[M4]** Token-type-aware accounting: a flexible way
  to capture different token types (input, output,
  cached, thinking) from different providers. Each
  tracked separately with configurable weights so
  quotas reflect real cost differences.
- **[M5]** Flexible bucket keys: quotas keyed by trusted
  request information such as authenticated subject,
  canonical model identity, or bounded compound keys so
  different clients and models get independent budgets.
  Raw caller-controlled headers are not identity. Header
  keying is suitable only for non-security policy or after
  an authoritative filter has replaced the caller value.
  (See upstream [wg-ai-gateway-keys] work and related
  issues #123, #129, #232.)
- **[M6]** Hard deny with 429 when a budget is
  exhausted, with standard rate limit response headers
  (`Retry-After`, `X-RateLimit-*`).
- **[M7]** Observability: metrics and tracing
  distinguishing admitted vs. limited requests,
  including estimated cost, actual cost, and remaining
  budget. The ability to log tokenomic results
  distinctly for accounting.

**Should have (same effort if capacity):**

- **[S0]** Multi-environment observability: aggregate
  token counts from proxies spanning multiple
  environments into one centralized view. (Moved from
  MVP; requires its own design proposal covering
  shared counter aggregation -- see [#155] and
  [ai#126].)

- **[S1]** Soft limits: usage tiers that modify request
  headers instead of rejecting, enabling downstream
  systems (e.g. llm-d `InferenceObjective`) to degrade
  gracefully as a client approaches its budget.
  (Related: #856 soft rate limiting, #549 quota
  exhaustion failover.)
- **[S2]** Batch workload awareness: separate quota
  rules or deferred accounting for batch/async API
  patterns so bulk jobs do not starve interactive
  traffic.
- **[S3]** Exact token metering: an accounting path
  that records precise actual-usage counts for billing
  and chargeback, independent of rate-limit counters.
- **[S4]** Proven-unused reservation recovery: release
  reserved capacity only when the system can establish
  that provider work did not occur. Ambiguous timeouts,
  disconnects, and lost responses remain conservatively
  charged.

### Non-Goals

- Replacing request-count rate limiting. Token and
  request-count quotas are independent concerns;
  operators may use both.
- Identity resolution and credential verification. The
  limiter consumes a trusted, private request-local
  `AuthenticatedIdentity` established by an earlier
  authentication filter. It must not infer identity from
  a caller-controlled header. Basic Auth is the first
  producer; JWT, OIDC, OAuth, API-key, mTLS, and external
  authentication can publish the same authentication-
  neutral identity contract.

These are non-goals for this _iteration_ but are
otherwise long term capabilities we do want.

- Usage-based dynamic rates: adjust quotas or refill
  rates based on observed usage patterns over a sliding
  window.
- Input message tokenization: fully tokenize request
  content before admission to use a real input token
  count rather than relying on estimations,
  `max_tokens` as a proxy, etc.
- Token Rate Limiting over a static window

### Prior Art

- **Kubernetes AI Gateway WG**: The upstream
  [wg-ai-gateway] working group is exploring a
  rate-limiting standard (see
  [wg-ai-gateway#60][wg-ai-gateway-rl]). This proposal
  should track that effort for compatibility where
  feasible. A related key-calculation discussion is in
  [wg-ai-gateway#57][wg-ai-gateway-keys].
- **Praxis request-count rate limiting**: Praxis
  already supports request-count rate limiting via the
  `rate_limit` filter. Token rate limiting is a
  complementary capability, not a replacement.

[wg-ai-gateway]: https://github.com/kubernetes-sigs/wg-ai-gateway
[wg-ai-gateway-rl]: https://github.com/kubernetes-sigs/wg-ai-gateway/pull/60
[wg-ai-gateway-keys]: https://github.com/kubernetes-sigs/wg-ai-gateway/pull/57

### Implementation and Experiment Inventory

This proposal describes the target architecture. Delivery is
incremental, and not every configuration sketch below is available
in a released build. The following table records the implementation
state as of September 2026 so architectural review does not confuse
merged behavior with active experiments.

| Area | Status | Implementation or tracker |
| --- | --- | --- |
| Ordered static-match rules | Merged, experimental feature | [ai#796] |
| Sliding-window admission | Merged, memory and Valkey | [ai#796] |
| Token-bucket admission | Merged, memory and Valkey | [ai#796] |
| Reserve then reconcile actual usage | Merged | [ai#796] |
| Hard denial before routing | Merged; HTTP 429 | [ai#796] |
| Shared multi-replica Valkey/Redis-compatible ledger | Merged; Valkey qualified experimentally | [ai#796] |
| Authenticated-subject budget key | Open | [praxis#1108], [ai#980] |
| Configurable estimation | Open | [ai#1008] |
| Soft enforcement | Local experiment; not an upstream PR or release | [ai#881] |
| Provider token extraction | Merged across supported response paths | [ENH-119], [ENH-210] |
| Token usage response headers | Merged | [ENH-214] |
| External metering | Merged separately from quota enforcement | [ai#581] |
| Quota metrics, tracing, and accounting logs | Open | [ai#883] |
| Three-subject shared-endpoint qualification | Experimental; not released | [ai#980] |
| Valkey/Redis authentication, TLS, HA, and sharding | Open | [ai#831], [ai#833], [ai#843] |
| Multiple budgets/tiers and weighted token types | Proposed | This proposal |

The merged `token_rate_limit` filter remains behind the
`token-rate-limit-filter` experimental Cargo feature. Its current
configuration is narrower than the target tier model:

```yaml
- filter: token_rate_limit
  key: global # global | authenticated_subject (ai#980)
  backend:
    kind: valkey # memory | valkey
    url: "${TOKEN_RATE_LIMIT_VALKEY_URL}"
    namespace: praxis:token_rate_limit
  rules:
    - name: team-alpha
      match:
        headers:
          x-plan: alpha
      algorithm: sliding_window
      window: 1h
      capacity: 100000
      reserved_tokens: 500
      reservation_timeout: 30s
```

Each current rule has one algorithm and one capacity. Static header
matching chooses a rule; it is not a trusted identity mechanism.
The default key source is `global`, which preserves one budget per
rule. [ai#980] adds opt-in `authenticated_subject` keying. It reads
only Praxis `AuthenticatedIdentity`, hashes the subject before using
it in backend keys or metrics, and returns 401 when that identity is
required but absent.

#### Current Ownership Boundaries

```text
Praxis authentication
  -> establishes private request-local AuthenticatedIdentity
Praxis AI token_rate_limit
  -> matches policy, reserves quota, and admits or rejects
Praxis AI routing
  -> selects a provider only after admission
Praxis AI token_count
  -> extracts provider-reported actual usage
Praxis AI token_rate_limit response processing
  -> settles the original reservation
Valkey or a qualified Redis-compatible service
  -> provides shared atomic quota state across gateway replicas
```

Quota identity must remain independent of the selected provider and
gateway replica. Changing providers does not mint capacity, and
scaling proxy instances does not multiply a shared-backend budget.
Quota enforcement does not depend on Kubernetes, a control plane, or
a particular routing implementation.

#### Current Reservation Contract

Admission enforces the reservation invariant:

```text
active reserved tokens <= capacity
```

The fixed or estimated reservation is an admission bound, not a
promise that final provider usage cannot exceed capacity. Settlement
may record more actual tokens than were reserved. That is truthful
accounting, not proof of over-admission. Tests must assert the peak
reserved-token invariant separately from settled actual usage.

Memory and Valkey use atomic reserve operations, including under
concurrency. The shared backend is authoritative across participating
replicas.
A backend failure returns 503 and stops before provider contact;
shared enforcement must not silently fall back to process-local
state.

An abandoned reservation is currently retained conservatively after
`reservation_timeout`: the estimate remains charged and ages out or
refills according to the configured algorithm. Refund-on-loss [S4]
is therefore future policy, not current behavior.

#### Standalone and Multi-Instance Qualification

The quota filter is independently deployable. A standalone Praxis AI
process can use memory for development, tests, or intentionally local
limits. Multiple standalone Praxis AI instances can point at the same
Valkey or qualified Redis-compatible service to enforce one shared
budget. The filter can precede static routing, Praxis load balancing,
an external scheduler, or another routing implementation.

The experimental multi-instance Kubernetes qualification exercises
the current sliding-window contract with real gateway processes and
shared Valkey. The generally applicable assertions are:

- admission and denial before provider selection;
- atomic concurrent reservation without over-admission;
- shared quota state across consumer gateway replicas;
- natural sliding-window expiry without editing Valkey or time;
- state persistence across consumer restart;
- fail-closed Valkey outage and recovery;
- provider attribution only for admitted requests;
- NetworkPolicy positive and negative controls;
- automatic teardown and structured evidence.

The three-subject experiment extends that proof to three applications
using one endpoint and one rule configuration. Each verified subject
gets an independent budget while the same subject shares state across
replicas. This depends on the merged Praxis identity producer
[praxis#1108] being published and consumed by [ai#980]. The contract is
not Kubernetes-specific.

The configured backend kind is currently named `valkey`, but the
implementation supports a compatible single-endpoint Redis service.
It uses the Rust `redis` client, `redis://` connection URLs, Redis
protocol commands, and Lua `EVAL`. Valkey is the implementation
exercised in CI and the multi-instance qualification. Redis products
must support the commands and Lua semantics used by the ledger.

This support does not yet imply every production topology is ready.
TLS (`rediss://`), authentication, clustered or sharded operation,
failover, script caching, and partition behavior require explicit
implementation or qualification; see [ai#831], [ai#833], and
[ai#843].

#### Soft-Enforcement Experiment

The active soft-quota experiment intentionally extends the existing
single-capacity rule instead of prematurely implementing the complete
multi-tier design shown later in this proposal. Its candidate API is:

```yaml
rules:
  - name: team-alpha
    algorithm: sliding_window
    window: 1h
    capacity: 100000
    reserved_tokens: 500
    enforcement:
      mode: soft # hard (default) | soft
      headers:
        x-praxis-quota-state: exceeded
```

`hard` preserves today's denial behavior. `soft` admits ordinary
capacity overage, creates a real reservation, reconciles actual usage,
and optionally replaces configured downstream signal headers. Soft
mode is not fail-open: missing required identity, backend failure,
invalid configuration, state-cardinality limits, numeric overflow,
and other operational safety failures still reject.

Soft state and hard state use the same rule and backend identity so a
hard-to-soft-to-hard policy change does not reset usage. Sliding-window
overage remains visible until it ages out. Token-bucket overage is
represented as debt and must recover through refill. A lost soft
request remains conservatively charged, matching hard mode.

This experiment has static coverage but is not currently qualified
against released Praxis identity support. It must remain explicitly
experimental until it is rebased onto released dependencies and its
multi-instance qualification proves shared overage, subject isolation,
concurrency accounting, restart persistence, state-preserving mode
changes, natural recovery, and fail-closed backend outage.

#### Related Tokenomics Proposals

- [ENH-99] defines shared hot-path state and typed token-ledger
  requirements. Valkey quota state belongs there; billing history
  does not.
- [ENH-119] and [ENH-210] define the current provider-specific token
  extraction design. [ENH-211] and [ENH-216] are withdrawn historical
  designs superseded by [ENH-210].
- [ENH-212] describes the request-local normalized usage contract
  consumed during settlement. Its implementation exists even though
  the proposal remains `proposed`.
- [ENH-214] exposes usage to clients but is not an enforcement or
  billing ledger.
- [ENH-220] covers integration evidence for token extraction. Quota
  qualification adds reserve, deny, reconcile, concurrency, failure,
  and multi-replica assertions on top.
- [ENH-191] may later generalize rule conditions; it must not turn
  caller-controlled identity headers into trusted principals.
- [ENH-664] defines trusted gateway-to-gateway metadata boundaries.
  Quota remains a data-plane concern owned by Praxis AI rather than
  a routing or control-plane concern.
- [ENH-784] and [ENH-794] are relevant to bounded audit and metrics
  output. Raw subjects, complete quota keys, credentials, prompts, and
  completions must not become metric labels.

## Why?

### Motivation

AI inference is a unique type of workload where
request-count rate limiting is meaningfully wrong.
Inference cost scales with token count, and a single
request can vary by orders of magnitude. An operator
limiting a team to 100 req/min has no control over
whether those requests consume 1,000 tokens or
10,000,000.

Three realities shape the requirements:

1. **Precise token counts are only available after the
   response.** Providers are unaware of output token
   counts before receiving the output and are therefore
   forced to report usage in the response body or
   headers. By the time actual counts are known, the
   tokens have been consumed. Admission decisions
   however must happen at request time, and therefore
   we must rely on token count estimates, with
   reconciliation after the fact. This is why a
   reservation-based model is necessary rather than
   simple post-hoc accounting.

2. **Not all tokens cost the same.** Providers charge
   significantly less for cached input tokens than
   uncached (roughly 10x for Anthropic). A quota system
   that treats all tokens equally will over-restrict
   users who benefit from caching or under-restrict
   those who do not. Token-type-aware weighting is
   essential for fair quotas.

3. **AI workloads span processes and environments.**
   Enterprise deployments run multiple proxy instances,
   whether as standalone services or under an
   orchestrator. Per-instance budgets cause effective
   quota to scale with instance count, the opposite of
   the intended control.

Beyond hard enforcement, operators need graduated
controls. When a team approaches its budget the
platform should be able to signal downstream schedulers
to route to cheaper models or lower-priority queues,
rather than rejecting outright. Hard deny is the
backstop, not the first line of defense.

### User Stories

- As a **platform operator**, I need per-team token
  budgets so shared inference infrastructure remains
  fair across consumers.

- As a **platform operator**, I need different models
  to carry different quota weights so budgets reflect
  actual inference cost.

- As a **platform operator**, I need cached tokens to
  consume less quota than uncached tokens so teams are
  not penalized for efficient caching.

- As a **platform operator**, I need quota state shared
  across proxy instances so teams cannot bypass limits
  by hitting different endpoints.

- As a **platform operator**, I need graduated controls
  that signal downstream systems at usage thresholds so
  the platform can degrade gracefully before hard deny.

- As a **FinOps engineer**, I need accurate per-team,
  per-model token consumption records for cost
  attribution and chargeback.

- As a **platform operator running batch workloads**, I
  need batch calls accounted for without starving
  real-time traffic.

## How?

### Requirements

- Rules for assigning token budgets based on traffic
  information
- A process-local memory backend for intentionally local
  limits and a shared Valkey/Redis-compatible backend for
  one authoritative budget across proxy instances
- Atomic admission against shared state, with no silent
  fallback to process-local enforcement when that state is
  unavailable
- Trusted subject keying through private authenticated
  identity metadata, never through an untrusted
  caller-supplied identity header
- Pluggable estimation strategies for request-time cost
  prediction
- Per-type token weights applied during reconciliation
  when actual usage by type is known
- Graduated soft limit tiers with header injection
  before hard deny
- Separate accounting for batch vs. interactive
  workloads
- Exact metering records independent of rate limit
  counters
- Conservative handling of abandoned reservations, with
  refunds only when provider work is proven not to have
  occurred

### Design

#### Token Budgeting

The configuration in this section is the target API, not
the configuration accepted by the current experimental
filter. The currently implemented single-budget API and
its delivery status are recorded in the implementation
inventory above. Migration from that API must preserve
existing `global` keying, hard-deny behavior, rule state
identity, and backend state unless an operator explicitly
changes them.

Define a set of `Rules` based on traffic information
(headers, model, path). Each rule binds one or more
`token_budgets` (for example an hourly budget and a
daily budget). Unmatched requests fall to a `default`
rule.

**Windows are sliding:** a `window: 1h` budget tracks
usage in the most recent 60 minutes from the current
instant (likewise `24h` for the most recent day).
Fixed/tumbling and calendar-aligned windows are out of
scope for MVP (see Non-Goals: static window). The
sliding window algorithm itself is tracked in [#551];
shared counter coordination is tracked in [#155].
Both primitives are required for this proposal's MVP
as currently scoped.

**Rule matching:** first matching rule wins. Conditions
within a match block are ANDed; multiple match blocks
are ORed.

**Budget evaluation:** every `token_budget` on the
matched rule is evaluated. Deny wins -- if any budget
hits a tier with `action.type: deny`, the request is
rejected with 429. Otherwise the request continues and
inject headers from all currently exceeded soft tiers
are unioned onto the request. If two budgets inject the
same header name, the last budget in config order wins;
prefer distinct header names per window.

**Tiers:** each budget is a capacity ladder over a
window. Thresholds use defined `capacity`. Every tier
has an `action` with a `type`:

- `inject` -- continue the request and apply
  `headers` (soft / signal tier). This is [S1]
  functionality; MVP implements the tier mechanism
  and `deny` action. `inject` support ships when S1
  is scheduled.
- `deny` -- hard-deny with 429.

Omit any `deny` tier for header-only / signal-driven
enforcement -- usage is still tracked (including
overage) for observability. Modeling soft and hard
outcomes as the same `action` field keeps the API
surface tight and leaves room for later action types
without a second parallel field. Rule-level
`estimation` and optional weight overrides are shown
below; token-type capture and default weights are
filter-wide.

```yaml
rules:
  - name: team-alpha
    match:
      # Static matchers (MVP)
      - headers:
          subscription: team-alpha-key
          x-praxis-ai-model: gpt-4o
      # Dynamic matchers (post-MVP / CEL) -- optional
      - cel: >
          request.auth.claims.sub == "team-alpha"
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 80_000
            action:
              type: inject
              headers:
                X-Token-Hour-Tier: warning
          - capacity: 95_000
            action:
              type: inject
              headers:
                X-Token-Hour-Tier: degraded
                x-gateway-inference-fairness-id: "85"
          - capacity: 100_000
            action:
              type: deny
      - window: 24h
        tiers:
          - capacity: 800_000
            action:
              type: inject
              headers:
                X-Token-Day-Tier: warning
          - capacity: 1_000_000
            action:
              type: deny

  # No `match` -> catch-all default rule
  - name: default
    token_budgets:
      - window: 1h
        tiers:
          # capacity 0 + deny -> reject unmatched traffic.
          # Omit token_budgets entirely for unlimited.
          - capacity: 0
            action:
              type: deny
```

Header-only example (no hard deny):

```yaml
token_budgets:
  - window: 1h
    tiers:
      - capacity: 80_000
        action:
          type: inject
          headers:
            X-Token-Tier: warning
      - capacity: 100_000
        action:
          type: inject
          headers:
            X-Token-Tier: exhausted
            x-gateway-inference-fairness-id: "85"
```

#### Request Lifecycle

Each request passes through four phases:

1. **Admission** - Match the request to a rule, compute
   an estimated cost using the configured strategy, and
   evaluate every `token_budget` on that rule. Inject
   headers from exceeded soft tiers; if any budget hits
   a `deny` tier, reject with 429. Estimation at
   admission can be optionally disabled in favor of
   response-only accounting.

2. **Inference** - The request is forwarded upstream.
   The provider performs inference and returns token
   usage.

3. **Reconciliation** - After inference completes, the
   admission reservation is settled against actual
   provider-reported token usage. The delta between
   estimated and actual cost is applied to the budget:
   refund unused capacity when actual < estimate, or
   charge the shortfall when actual > estimate.
   Overshoot (actual exceeding the reservation) is
   permitted -- the budget may temporarily go negative
   for observability rather than silently dropping
   usage. Weighted costs (per-type weights) are applied
   at this stage since actual token-type breakdowns are
   now available. The estimate-vs-actual difference is
   logged for observability.

4. **Abandoned reservation handling** - If a request is
   not reconciled, retain its estimated charge and let it
   age out or refill according to the configured
   algorithm. A future recovery mechanism may refund only
   when it can prove that provider work did not occur;
   timeout, connection reset, or a lost response alone is
   insufficient proof.

#### Estimation Strategies

Estimation is pluggable and configured **once per
rule** (not per `token_budget`). Hourly and daily
budgets on the same rule share one cost model.
Strategies operate on request metadata available before
forwarding (e.g. `max_tokens`, model identity, content
length).

Built-in strategies (MVP starting point):

| Strategy | Basis | Use case |
| --- | --- | --- |
| `max_tokens` | `max_tokens` field | Simple upper bound |
| `input_plus_max_tokens` | body size + `max_tokens` | Conservative estimate |
| `fixed` | constant per request | Uniform cost model |
| `model_scaled` | `max_tokens` * model multiplier | Model-aware budgets |

```yaml
rules:
  - name: team-alpha
    estimation:
      strategy: input_plus_max_tokens
      multiplier: 1.2
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 1_000_000
            action:
              type: deny
```

When a rule omits the `estimation` key entirely,
requests are admitted without reservation and the
budget is charged only at reconciliation. This is
equivalent to response-only accounting with no
admission-time protection.

When a strategy depends on `max_tokens` but the
request does not include it, the strategy falls back
to the rule's configured `fallback_estimate` (a fixed
token count). If no fallback is configured, the
request is admitted without reservation (same as
omitting `estimation`). This prevents false denials
on requests that legitimately omit `max_tokens`.

The fixed set of named strategies covers common
patterns. Additional strategies can be added as
requirements emerge without changing the configuration
surface. Smarter approaches (historical usage, external
estimators) are post-MVP -- see open questions / PR
discussion.

#### Token Type Capture

Providers expose usage differently -- OpenAI vs
Anthropic field names diverge, and even within one
provider the shape varies by API and feature (Chat
Completions vs Responses; cache, reasoning, audio,
thinking). See OpenAI
[prompt caching / usage details](https://developers.openai.com/api/docs/guides/prompt-caching)
and Anthropic
[prompt caching usage](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
(plus Messages `usage` /
`output_tokens_details.thinking_tokens`).

Capture is configured **once at filter scope** and
applies to every rule. Prefer a small set of presets
(start with OpenAI; Anthropic when ready) that map
provider fields onto a shared type set (`input`,
`output`, `cached_input`, `reasoning`, ...). Operators
can also define a custom mapping (logical type ->
response path) so accounting keeps working when a
provider adds fields and our presets lag -- a failure
mode LiteLLM-style hardcoding hits often.

```yaml
filter: token_rate_limit
token_type_capture:
  preset: openai
  # Or custom (paths illustrative):
  # mapping:
  #   input: usage.prompt_tokens
  #   cached_input: usage.prompt_tokens_details.cached_tokens
  #   output: usage.completion_tokens
  #   reasoning: usage.completion_tokens_details.reasoning_tokens
```

Reuse existing Praxis token-usage extraction where it
already covers a provider; capture config is the
operator-facing extension surface on top of that.

**Streaming considerations:** for streaming responses,
token usage is typically reported in the final SSE
chunk. The existing `token_usage` filter already
handles this for Chat Completions (forcing
`include_usage` when needed). Verify that all
supported streaming paths (Responses API, Anthropic
Messages) reliably surface usage before relying on
reconciliation for those paths.

#### Token Type Accounting

Default weights are also **filter-wide**. A weight
below 1.0 means that type consumes proportionally less
budget. Rules may override weights when one tenant
needs a different cost model; omitted types fall back
to the filter defaults (then 1.0).

```yaml
filter: token_rate_limit
default_weights:
  input: 1.0
  output: 1.0
  cached_input: 0.1
  reasoning: 0.9
rules:
  - name: team-alpha
    # optional per-rule override
    weights:
      cached_input: 0.05
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 1_000_000
            action:
              type: deny
```

Weighted cost at reconciliation:

```text
cost = Sum (tokens_of_type * weight_of_type)
```

Admission still uses a single estimated cost; typed
weights apply when the provider reports actual counts.

#### Batch Workloads

Batch traffic can be a dedicated `Rule` with its own
`token_budgets`, or a nested match under a parent rule
that shares identity but uses different budgets (for
example a larger daily window for `/v1/batches`).

> **Note**: The batch configuration shape below is an
> illustrative sketch. The Open Questions section
> acknowledges that batch APIs have a fundamentally
> different settlement path. This example will evolve
> once the metrics path is understood.

```yaml
rules:
  - name: team-alpha
    match:
      - headers:
          x-api-key: team-alpha-key
    token_budgets:
      - window: 1h
        tiers:
          - capacity: 1_000_000
            action:
              type: deny
    # Optional nested batch rule (same parent identity,
    # different budgets). Exact nesting TBD.
    batch:
      match:
        - path_prefix: /v1/batches
      token_budgets:
        - window: 24h
          tiers:
            - capacity: 5_000_000
              action:
                type: deny
```

When no separate batch rule/budgets are configured,
batch traffic shares the parent rule's budgets.

> **Note**: Nested batch rules can also be used only
> for accounting when no separate budget is provided.

#### Metering

Rate limiting and metering serve different purposes.
Rate limiting admits on estimates, then reconciles
quotas to actual provider usage. Metering records exact
actual usage for billing and chargeback.

The metering path emits records after reconciliation
containing: rule name, model, exact token counts by
type (from the provider), weighted cost, and timestamp.
These records are independent of rate limit counters
and can be consumed by external billing systems.

#### Observability

The system emits:

- **Metrics**: tokens reserved, reconciled, and
  refunded; budget remaining; requests admitted vs.
  denied; soft limit tier activations; overage amounts
- **Tracing**: per-request spans with estimated cost,
  actual cost, matched rule, and admission decision
- **Accounting logs**: structured records at a
  dedicated log target for tokenomic auditing, separate
  from operational logs

## Open Questions

### Estimation configurability

The estimation method must be operator-defined ([M3]).
Named strategies with parameters are the MVP starting
point. Bring back to the group: how far should we go
beyond that -- expression languages, historical usage,
or an external estimation source? Those are post-MVP
but should shape the strategy extension surface.

### Estimate reconciliation overshoot

MVP reconciles budgets to actual provider usage
(refund underestimates / charge overestimates relative
to the admission reservation). An open question remains
for when actual cost exceeds what was reserved and the
budget has little or no remaining capacity: do we allow
the overshoot (budget goes negative / over-capacity for
observability), clamp at capacity, or apply a different
policy? That edge case should be decided before
implementation hardens the accounting lifecycle.

### MVP token-type capture presets

Which provider presets should ship out of the box for
MVP? OpenAI is required. Anthropic is a strong
candidate given existing Messages support in Praxis --
confirm whether it is MVP or immediately post-MVP.
Other providers would use custom mappings until presets
exist.

### Batch workload accounting

Separate or nested rules can isolate batch traffic, but
that may not solve the real problem. Batch APIs usually
return an "accepted" response on submit -- token usage
is not on that request/response path unless we are
wired into whatever system later reports job
completion.

Our reservation + reconcile model assumes usage comes
back on the same request. Batch likely needs a
different reservation / settlement path. We should work
with the model-serving team on batch to learn how and
when token metrics become available before treating the
current sketch as sufficient.

Open shape questions remain: separate rules vs nested
rules under a parent, both, or something else entirely
once the metrics path is clear.

### Lost request handling

The current implementation uses a configurable
`reservation_timeout`. When an admitted request is never
settled, its estimate remains charged rather than being
refunded: sliding-window state retains it until it ages
out, and token-bucket admission has already decremented
the bucket. This is conservative and avoids granting free
usage after an ambiguous failure.

[S4] proposes a refund policy, but it must first define
which failures prove that provider work did not occur.
Timeout, disconnect, or lost response alone is not that
proof. Any future refund mechanism needs idempotent
settlement, a bounded hold time, and evidence that it
cannot refund inference that actually completed.

### `Retry-After` for token budgets

[M6] promises `Retry-After` and `X-RateLimit-*` on
hard deny. For request-count limiters, remaining
window time is a natural `Retry-After`. For token
budgets it is not: "when capacity becomes available"
depends on window type (sliding vs tumbling -- itself
still open), how usage ages out of the window, and
whether in-flight reservations release early.

Open question: how would we calculate a meaningful
`Retry-After` for token budgets, and how difficult
is that in practice (especially once window semantics
and reservation release are fixed)? We should
understand the cost and complexity of doing this
correctly before committing MVP to a specific
formula. `X-RateLimit-*` remaining/limit fields are
comparatively straightforward; the hard part is the
retry hint.

### Streaming token usage

OpenAI's `include_usage` flag for streaming is
currently only forced in the chat completions path.
Streaming responses from other APIs (Responses API,
Anthropic Messages) may not always surface usage in a
consistent location. The proposal should verify that
token usage is reliably available for all supported
streaming paths before MVP hardens the reconciliation
lifecycle.

### Concurrent in-flight reservations

The implemented memory and Valkey backends atomically
check and create reservations. Concurrent requests must
not cause active reserved tokens to exceed capacity.
Valkey performs the multi-window decision in one Lua
operation; the memory backend provides the same contract
with in-process synchronization.

Actual settled usage can exceed capacity when a request
uses more tokens than its estimate. That is reconciliation
overshoot, not concurrent over-admission. Evidence and
metrics must keep these two conditions distinct.

### Hot reload and window state

The memory backend is process-local and can reset when a
filter instance is rebuilt or the gateway restarts. It is
not a fleet-wide production quota. Valkey-backed state is
external to the filter instance and survives consumer
restart and configuration reload when the namespace,
rule name, algorithm identity, and key source remain
stable.

Renaming a Valkey-backed rule or namespace changes state
identity and starts a new budget while the old keys expire.
There is no migration mechanism today. Policy-only changes
such as hard-to-soft enforcement must not change state
identity.

### Estimation-reconciliation weight asymmetry

Admission estimates are unweighted (a single cost
number), but reconciliation applies per-type weights.
If the weight profile skews heavily (e.g.
`cached_input: 0.1`), the unweighted estimate may
reserve far more capacity than reconciliation actually
charges, halving the usable budget in practice. Consider
whether estimation should apply weights to a rough
type breakdown, or whether documenting the asymmetry
as acceptable for MVP is sufficient.

### Tier validation constraints

The proposal does not specify validation rules for
tier definitions: must capacities be strictly
ascending? Are duplicate capacities allowed? What
happens when a budget has zero tiers? Define these
constraints before implementation to avoid ambiguous
configurations.

[#155]: https://github.com/praxis-proxy/praxis/issues/155
[#551]: https://github.com/praxis-proxy/praxis/issues/551
[ai#126]: https://github.com/praxis-proxy/ai/issues/126
[ai#581]: https://github.com/praxis-proxy/ai/pull/581
[ai#796]: https://github.com/praxis-proxy/ai/pull/796
[ai#831]: https://github.com/praxis-proxy/ai/issues/831
[ai#833]: https://github.com/praxis-proxy/ai/issues/833
[ai#843]: https://github.com/praxis-proxy/ai/issues/843
[ai#881]: https://github.com/praxis-proxy/ai/issues/881
[ai#883]: https://github.com/praxis-proxy/ai/issues/883
[ai#980]: https://github.com/praxis-proxy/ai/pull/980
[ai#1008]: https://github.com/praxis-proxy/ai/pull/1008
[praxis#1108]: https://github.com/praxis-proxy/praxis/pull/1108
[ENH-99]: ./00099_stateful-proxy-state-management.md
[ENH-119]: ./00119_streaming-token-accumulation.md
[ENH-191]: ./00191_filter-chain-condition-expressions.md
[ENH-210]: ./00210_response-based-token-counting.md
[ENH-211]: ./00211_streaming-token-counting.md
[ENH-212]: ./00212_token-count-filter-context.md
[ENH-214]: ./00214_token-usage-response-headers.md
[ENH-216]: ./00216_provider-token-mapping.md
[ENH-220]: ./00220_token-counting-integration-tests.md
[ENH-664]: ./00664_gateway-to-gateway-connectivity.md
[ENH-784]: ./00784_structured-security-audit-log-format.md
[ENH-794]: ./00794_expand-prometheus-metrics-surface.md
