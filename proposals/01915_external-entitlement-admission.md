---
issue: https://github.com/praxis-proxy/ai/issues/1915
discussion: https://github.com/praxis-proxy/ai/issues/1915
status: proposed
repos:
  - praxis
  - ai
  - policy
authors:
  - jland-redhat
graduation_criteria:
  - Maintainers agree spend cut-off is driven by external entitlement, not an in-Praxis wallet ledger
  - Admission contract supports subject-keyed go/no-go as the MVP admit input
  - Proposal explicitly allows non-token cost dimensions to influence entitlement outside Praxis
  - Relationship to Tokenomics (00121) is documented: keep usage reporting and rates; narrow wallet/drain scope
stakeholders:
  - shaneutt
  - jland-redhat
  - asaadbalum
  - leseb
  - alexsnaps
related:
  - 00121
# Refines spend-control direction of 00121; does not replace usage reporting.
---

# External entitlement admission (Tokenomics spend-control refinement)

## What?

**Spend cut-off should not be a wallet Praxis drains.**

Tokenomics ([00121](00121_tokenomics.md)) already gives Praxis a strong path for
**usage reporting**, **canonical token breakdowns**, and **rate / QoS** control.
This proposal refines the **spend-control** part of that direction:

| Keep in Praxis | Move outside Praxis |
| --- | --- |
| Emit accurate multi-type usage | Wallet balance, drain, top-up, rollover as cost-of-record |
| Optional weights for observability / rate math | Invoice pricing and commercial rate cards |
| Local **admission** against an entitlement signal | Aggregating “budget” across tokens, GPU, and other rated spend |
| Hooks to refresh / raise the local limit signal | Being the FinOps ledger or Quote-to-Cash system |

### Goals

- Treat external systems as the place that **specifies limits** and decides when
  those limits change (exhaustion, top-up, period grant, policy override).
- Give Praxis a **flexible admission hook** that consumes that decision locally
  (no per-request outbound call to the meter on the hot path).
- Start with a simple **go / no-go** decision keyed by subject
  (**user / organization / agent** — a key/value entitlement map).
- Keep the hook open so entitlement may already include **non-token cost**
  computed elsewhere (for example GPU usage already rated in another system)
  before Praxis sees a boolean (or richer) admit result.
- Preserve Tokenomics reporting-without-budgets and rate-limit concerns as
  separate from FinOps cut-off.

### Non-goals (this proposal’s What?)

- Defining Monetize / Price Tag / Cost Management connector APIs in detail
  (follow-up How?).
- Replacing Tokenomics usage normalization or CloudEvents emit.
- Requiring Praxis to price inference or hold prepaid balances.
- Guaranteeing zero overrun between external cut-off and local admit refresh.

## Why?

### Motivation

AI platforms need the **customer outcome** peers set expectations for: stop
(or clearly signal) when spend entitlement is exhausted, distinct from short-window
QoS 429s. That outcome does **not** require Praxis to own a drained wallet.

In practice:

1. **Budgets are bigger than tokens.** A tenant’s real allowance often includes
   GPU time, platform fees, committed contracts, and other rated usage. If Praxis
   only drains a token/credit wallet, it either under-models the budget or grows
   into a second ledger that duplicates FinOps systems.
2. **Cost of record belongs outside the gateway.** Commercial pricing, wallets,
   and cut-off floors (for example a meter that only meaningfully cuts off above
   a minimum wallet) already live in external metering / Quote-to-Cash products.
   Re-implementing drain/rollover/reserve-settle inside Praxis recreates that
   surface and invites dual-ledger drift.
3. **Admit must stay fast and local.** Hot-path calls to an external meter are
   fragile. The durable pattern is: external system decides → platform pushes or
   refreshes a **local entitlement** → Praxis admits on that signal.
4. **Tokenomics already separates “report usage” from “enforce budget.”**
   The reporting-without-budgets path matches product priority (accurate emit
   first). Spend control should plug in as **entitlement admission**, not as a
   mandatory in-proxy allowance engine.

This refinement keeps Praxis focused on **enforcement of a limit signal** and
**honest usage export**, while operators choose Price Tag, Monetize 360,
Cost Management, or BYO systems as the limit authority.

### User Stories

- **External limits:** As a platform owner, I want Praxis to enforce limits that
  an external system defines, so I do not run a second wallet inside the gateway.
- **Raise the limit:** As a budget owner, I want topping up or raising a limit in
  the external system to restore access through a Praxis hook, without editing
  gateway wallet YAML as the source of truth.
- **Simple go/no-go:** As a gateway operator, I want admission to start as a
  subject-keyed allow/deny map (user, organization, or agent), so the first
  integration is a clear boolean rather than full in-proxy cost math.
- **Composite budgets:** As a FinOps integrator, I want GPU and other non-token
  costs already calculated outside Praxis to influence the same entitlement
  decision, so cut-off reflects the real budget rather than tokens alone.
- **Local admit:** As an SRE, I want the hot path to read local entitlement
  state only, so meter outages and poll lag are explicit and do not become
  synchronous dependencies on every inference.
- **Usage still first:** As an integrator, I want raw token breakdowns and
  attribution emitted even when no local wallet is configured, so external
  systems can rate and decide.
- **QoS stays separate:** As an operator, I want short-window rate limits to
  remain distinct from spend entitlement, so a funds deny is not confused with
  a TPM/RPM throttle.
- **No dual ledger:** As a product owner, I want invoice price and remaining
  balance to live in one external system of record, so Praxis weights and
  reports never become a competing cost ledger.

### Relationship to [00121 Tokenomics](00121_tokenomics.md)

| Topic | 00121 today | This refinement |
| --- | --- | --- |
| Usage reporting without budgets | Supported | **Keep** — primary near-term path |
| Canonical multi-type tokens | Supported | **Keep** |
| Rates / soft-hard QoS signals | In Tokenomics API | **Keep** as QoS; do not conflate with FinOps cut-off |
| In-Praxis budget allowance, drain/rollover, reserve/settle wallet | Central MVP for spend control | **Narrow** — not the preferred spend-control SoT |
| Budget manager owns HA history | Yes | **Yes, and** the manager (or meter) owns **cut-off decision**; Praxis consumes entitlement |
| Weighted costs / prices in gateway config | First-class for enforcement | Optional for observability / rates; **not** invoice SoT |

This proposal does **not** ask to delete Tokenomics. It asks to **steer spend
control** toward external entitlement admission so Tokenomics does not become
the wallet product.
