---
discussion: https://github.com/orgs/praxis-proxy/discussions/981
issue: https://github.com/praxis-proxy/enhancements/issues/9
status: proposed
repos:
  - praxis
  - ai
authors:
  - leseb
graduation_criteria:
  - How section defining the server composition boundary
  - Synchronous, side-effect-free factory contract reviewed
  - Pipeline extension ownership and reload semantics reviewed
  - Integration with Async Pipeline Activation agreed
  - Candidate activation teardown semantics reviewed
  - Runtime policy publication and atomicity semantics reviewed
  - Offline validation and dump behavior defined
  - Backward-compatibility and migration plan reviewed
stakeholders:
  - shaneutt
  - alexsnaps
experimental_exempt: true
experimental_exempt_reason: Core server infrastructure cannot be prototyped as an external filter.
---

# Embeddable Server Composition

## What?

Introduce a supported composition boundary for downstream
distributions that embed the Praxis server.

Downstream distributions must be able to provide:

- A filter registry.
- Pipeline-scoped extensions.
- Pipeline activators defined by
  [Async Pipeline Activation].
- Distribution-specific configuration defaults, CLI integration,
  and branding.

Distribution defaults are limited to the configuration source and
application metadata used when operator input is absent. Explicit
operator configuration and CLI values always take precedence. The
composition boundary cannot rewrite parsed operator configuration.

Praxis must retain ownership of:

- Pipeline resolution, validation, activation, and publication.
- Dynamic configuration reload and rollback.
- Listener and health-check lifecycle.
- Server-managed runtime resources and policies.
- Offline validation and effective-configuration dumping.

Currently known server-managed state includes shared subrequest
clients and their safety limits, health-check state, the KV store
registry, and listener pipeline handles. Distribution-specific stores
and caches remain pipeline-scoped resources. This boundary does not
make restart-only connector, listener, or TLS settings reloadable.

Startup, reload, validation, and dump must use the same
composition inputs. Validation and dump resolve and validate the
composition without activating resources or performing activation
I/O.

Extension factories must remain synchronous and side-effect-free.
Asynchronous resource or schema preparation remains a separate
resolve, activate, and publish concern.

A successful reload must publish candidate pipelines, activated
resources, and reloadable server-managed policy as one coherent
generation. Failed resolution or activation must publish none of
them. Server-managed policy must remain current for every listener
that continues serving, including listeners whose removal requires
a restart.

If activation fails after partial success, all candidate-owned
resources acquired for the unpublished generation must be released
before the failure is returned. Async Pipeline Activation must define
teardown ownership, ordering, and error handling; composition must
reuse that lifecycle rather than create a second one.

Composition is the static host boundary, not a plugin loader. Plugins
remain a separate, startup-only Praxis extension surface: composition
inputs are established before pipeline resolution, while plugin
loading and dispatch follow the plugin proposal and do not participate
in dynamic pipeline reload.

Existing server entry points must remain supported.

[Async Pipeline Activation]: https://github.com/orgs/praxis-proxy/discussions/973

### Goals

- Let downstream distributions extend Praxis without copying its
  server lifecycle.
- Keep listener, watcher, reload, validation, and publication
  behavior inside Praxis.
- Allow downstream pipeline resources to participate in Async
  Pipeline Activation.
- Keep server-managed runtime policies coherent across existing
  consumers and successful reloads.
- Define clear ownership and lifetime semantics for
  pipeline-scoped extensions.
- Preserve failed-reload atomicity.
- Preserve compatibility for existing custom-registry users.
- Keep validation and dump free of activation side effects.

### Non-goals

- A general-purpose framework for Praxis plugins
- Arbitrary downstream mutation of resolved pipelines.
- Redefining Async Pipeline Activation.
- Moving downstream filter or application types into Praxis.
- Making every runtime setting dynamically reloadable.
- Performing asynchronous initialization in filter or extension
  factories.

## Why?

### Motivation

Praxis supports custom filter registries through
`run_server_with_registry`, but it does not provide a supported
way to compose pipeline-scoped extensions and activators through
the same startup, validation, reload, and publication lifecycle.

Downstream distributions that require these capabilities must
copy generic Praxis server code. This creates parallel
implementations of pipeline resolution, configuration watching,
reload, health-check management, validation, and runtime-resource
construction.

These copies make ownership unclear. Generic lifecycle fixes can
be implemented downstream even though they apply to every Praxis
distribution, while upstream fixes may not propagate to copied
implementations.

Reloadable subrequest limits demonstrate the problem.
`SubRequestClient` clones capture their response ceiling at
construction. Reload can construct a client with updated limits
while filters or still-bound pipelines retain clients created at
startup. A downstream workaround can update individual factories,
but it cannot reliably enforce server-wide policy across every
listener and consumer.

Async Pipeline Activation demonstrates the same missing boundary
for pipeline-scoped resources. Praxis owns publication and reload,
so downstream application startup code cannot ensure that
resources are activated before a pipeline begins serving traffic.

Praxis needs one lifecycle boundary that resolves candidate
pipelines, validates them, activates their resources when
appropriate, manages runtime policy, and publishes them safely.
Downstream distributions should describe their composition while
Praxis performs the lifecycle.

This makes the Praxis server a deeper module: consumers receive a
small composition interface while startup, reload, rollback,
activation, and resource coordination remain hidden behind it.

### User Stories

- As a downstream distribution author, I want to register filters,
  pipeline extensions, and activators without copying the Praxis
  server implementation.

- As an operator, I want startup, reload, validation, and dump to
  apply the same composition and validation rules.

- As a filter author, I want server-managed resources and safety
  policies to remain current after a successful reload.

- As a resource author, I want pipeline-scoped resources to
  activate before their pipeline is published.

- As a Praxis maintainer, I want listener, watcher, reload,
  activation, and runtime-policy behavior to have one
  implementation and one test boundary.

- As an existing custom-registry user, I want current server entry
  points to remain compatible while the new composition boundary
  is introduced.

### Related

- [Embeddable Server Composition Discussion]
- [Async Pipeline Activation]
- [Dynamic Configuration Reloading]
- [Hook System and Plugin Support]
- [Pipeline Execution Engine]
- [Praxis AI issue #639]

[Embeddable Server Composition Discussion]: https://github.com/orgs/praxis-proxy/discussions/981
[Dynamic Configuration Reloading]: https://github.com/praxis-proxy/enhancements/blob/main/proposals/00011_dynamic-configuration-reloading.md
[Hook System and Plugin Support]: https://github.com/praxis-proxy/enhancements/blob/main/proposals/00063_plugins.md
[Pipeline Execution Engine]: https://github.com/praxis-proxy/enhancements/blob/main/proposals/00787_pipeline-execution-engine.md
[Praxis AI issue #639]: https://github.com/praxis-proxy/ai/issues/639
