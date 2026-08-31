---
issue: https://github.com/praxis-proxy/ai/issues/753
discussion: https://github.com/orgs/praxis-proxy/discussions/1044
status: proposed
authors:
  - liavweiss
repos:
  - praxis
  - ai
graduation_criteria:
  - Pipeline security checks use SecurityClass,
    not a hardcoded name list
  - SECURITY_FILTERS is removed
  - Tests cover a custom Security-class filter
    (fail-open, conditions, SkipTo)
  - Operator docs describe security-class filters,
    not a short builtin name list
stakeholders:
  - shaneutt
  - twghu
  - jordigilh
---

## What?

Pipeline validation treats a hardcoded name list as
the source of truth for "this filter is
security-critical." Custom filters registered through
the public registry API (including praxis-ai filters)
cannot join that set, even when they are already
marked `SecurityClass::Security`.

`SecurityClass` already exists on the registry.
Builtins are classified with it, and out-of-tree
filters can opt in via `register_with_class`. The
fail-open, conditional-security, and SkipTo-bypass
checks still match names against `SECURITY_FILTERS`
instead of that class.

Registration decides whether a filter is
security-critical. The existing checks follow that
classification. Core does not list downstream filter
names.

### Goals

- Config-time fail-open, condition, and SkipTo
  checks apply to any filter registered as
  `SecurityClass::Security`, not only builtins.
- Out-of-tree and AI-registered security filters
  can opt in without a core name-list change.
- Builtins keep the same operator-facing behavior
  they have today (`failure_mode: open` still
  requires
  `insecure_options.allow_open_security_filters`).
- `SECURITY_FILTERS` is removed so the two sources
  of truth cannot drift.

## Why?

### Motivation

A security filter with `failure_mode: open` skips
the check on provider timeout, network error, or
other `FilterError` and forwards the request. Core
already refuses that misconfiguration at startup
for builtins (`ip_acl`, `guardrails`, `csrf`, …).

That safety net is a closed list of core names. A
filter registered later, even as
`SecurityClass::Security`, is not in the list, so
operators get no warning.

This showed up downstream in
[praxis-proxy/ai#753](https://github.com/praxis-proxy/ai/issues/753)
for `ai_guardrails`: it is a security gate, but
`failure_mode: open` is accepted with no
config-time error. The same gap exists for other
AI filters that already call
`register_with_class(..., SecurityClass::Security)`
today (for example `credential_inject`). Marking
them Security does not currently change validation.

`ai_guardrails` is one consumer, not the core
change. Core should not special-case that name.
Any future security-class filter (AI or otherwise)
should get the same checks by registering as
Security.

The registry comment already describes
`SecurityClass` as enabling future validation.
That future is now: checks should use the class
that registration already records.

### User Stories

- As a proxy operator, I want `failure_mode: open`
  on a security-class filter to fail startup
  (unless I set `allow_open_security_filters`), so
  a provider outage cannot silently bypass the
  check.
- As a filter author outside core, I want to mark
  a filter `SecurityClass::Security` and have the
  existing pipeline checks apply, so I do not need
  a core PR per filter name.
- As a Praxis maintainer, I want one
  classification used by both the registry and the
  checks, so the two cannot drift.

## How?

### Requirements

- `check_open_security_filters`,
  `check_conditional_security`, and
  `check_skip_to_bypasses_security` classify a
  filter by `SecurityClass`, not by name.
- `SECURITY_FILTERS` is deleted. Builtins stay
  security-critical because they are registered
  with `register_http_security`.
- Pipeline build stamps each `PipelineFilter` with
  the registry class for that filter type, so
  checks do not need the registry at validation
  time.
- A custom filter registered as
  `SecurityClass::Security` with
  `failure_mode: open` fails pipeline validation
  unless `allow_open_security_filters` is set. The
  same filter registered as `Standard` does not.
- Builtin cases (`ip_acl`, `forwarded_headers`,
  …) keep their current errors and the insecure
  override.
- Operator docs talk about security-class filters,
  not a hardcoded pair of names.

### Non-goals

- Changing which builtins are `Security`.
- Adding new `SecurityClass` variants.
- Implementing the praxis-ai registration change
  in this proposal. Downstream follow-up is
  described separately below.

### Design

**Source of truth.**
`FilterRegistry` already stores `SecurityClass` on
each registration. `register()` defaults to
`Standard`. `register_with_class` and
`register_http_security` set `Security`. Pipeline
checks should read that metadata.

**Stamp at build.**
`FilterPipeline::build` and
`build_with_chains` already have the registry.
Each entry is instantiated with
`registry.create(&entry.filter_type, ...)` first.
Unknown, unregistered, or mistyped filter names
already fail there (`unknown filter type`). Only
after `create` succeeds is `is_security` set from
`registry.is_security_filter(&entry.filter_type)`.
Do the same in `build_branch.rs`, which builds
branch sub-chains. `ordering_errors` then has the
bit on every filter, including nested branches.

The stamp cannot see a missing type:
`is_security_filter` is only called for names
that just instantiated. Config reload uses the
same build path, so an unregistered name still
fails at `create`, not at the security stamp.

**`PipelineFilter`.**
Add `is_security: bool` (fields stay alphabetical,
`name` pinned first). A bool is enough: checks
only distinguish Security from Standard.

**Checks.**
In `filter/src/pipeline/checks.rs`, replace
`SECURITY_FILTERS.contains(name)` with
`filters[i].is_security` in:

- `check_open_security_filters`
- `check_conditional_security`
- `check_skip_to_bypasses_security`

Error strings can keep using `filter.name()` so
operators still see which filter is misconfigured.

Delete `SECURITY_FILTERS` and the test that it
matches `registry.security_filters()`. That test
encoded the dual source of truth this proposal
removes.

**Tests.**
Keep existing builtin fail-open / allow-flag /
conditional / SkipTo cases. Add:

- Custom filter, `SecurityClass::Security`,
  `failure_mode: open` → validation error.
- Same filter with
  `allow_open_security_filters` → no error
  (warning path).
- Custom filter, `SecurityClass::Standard`,
  `failure_mode: open` → no error.
- Custom Security-class filter with request
  conditions → error.
- SkipTo that jumps over a custom Security-class
  filter → error.

**Docs.**
Update `docs/developing/getting-started.md` and
`docs/filters/README.md`. Today they name
`ip_acl` and `forwarded_headers`. They should say
any filter registered as
`SecurityClass::Security`, including out-of-tree
filters.

### Behavior change

This is a behavior change for out-of-tree filters
already registered as Security. Today
`failure_mode: open` on those filters is accepted.
After this change, startup fails unless
`allow_open_security_filters` is set. That is the
intended outcome: core and external security
filters are treated the same.

`provider_route` already has a stricter
praxis-ai-local fail-closed check, so it is
unaffected in practice. `credential_inject` is
the existing consumer that gains the core check
with no praxis-ai code change, once praxis-ai
takes a `praxis-filter` that includes this work.

### Implementation PRs

- Core: stamp `is_security`, switch the three
  checks, delete `SECURITY_FILTERS`, tests, docs.

### Downstream follow-up

Out of scope for this proposal's PRs. After a
`praxis-filter` release that includes the core
change, praxis-ai registers `ai_guardrails` as
`SecurityClass::Security`. That is enabled by
this work and closes
[praxis-proxy/ai#753](https://github.com/praxis-proxy/ai/issues/753)
Gap 1. It is not delivered in the core PR.
