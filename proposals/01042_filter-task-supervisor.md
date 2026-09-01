---
discussion: https://github.com/praxis-proxy/ai/pull/815/changes#r3854446287
issue: https://github.com/praxis-proxy/praxis/issues/1042
status: proposed
repos:
  - praxis
  - ai
  - extproc
authors:
  - aslakknutsen
graduation_criteria:
  - How? section with requirements and design
  - Lifecycle (construct / start / cancel-on-pipeline-drop) agreed by stakeholders
  - Fail-construction vs fail-closed spawn modes agreed
  - Decision on whether intelligent_route overlay reload migrates in the same change
stakeholders:
  - shaneutt
  - alexsnaps
  - szedan-rh
related:
  - https://github.com/praxis-proxy/praxis/issues/1043
---

# Pipeline-Scoped Filter Task Supervisor

## What?

`HttpFilter` / `TcpFilter` have no lifecycle for work
that must outlive a single request and die with the
pipeline. Filter construction is synchronous, often
runs with no Tokio runtime, and pipeline hot-reload
drops the old chain.

Filters that need background work today mint a
private OS thread and a private current-thread
runtime, then cancel it in `Drop`. That is an
accident of missing API, not a design.

This proposal adds a **pipeline-owned task
supervisor**: filters may start named background
tasks during pipeline build; those tasks are
cancelled when that pipeline is dropped (reload or
shutdown). Filters do not spawn threads or create
runtimes.

Pingora `BackgroundService` stays for
**process-lifetime** work registered before
`server.run()` (listeners, admin HTTP). This
supervisor is **pipeline-lifetime**. Those are
different scopes. Config watchers, health-check
loops, and TLS cert watchers stay server-owned.


### Goals

- Give filters a supported way to run background
  work whose lifetime is the pipeline, not the
  process and not the request.
- Keep `from_config` as YAML parse + validation.
  Background work starts in a later pipeline-build
  hook, so most filters stay unaware of the
  supervisor.
- Cancel all tasks for a pipeline when that
  pipeline is dropped, so reload cannot leak
  refreshers or watchers.
- Distinguish **fail construction** (background
  work never became ready; overlay watcher) from
  **fail closed** (requests reject until work
  succeeds; token refresh).
- Make spawn/runtime failure a pipeline-build
  error, not a live filter that 503s forever.
- Stop new filters from copying the overlay-watcher
  thread pattern.

### Non-Goals

- Replacing Pingora `BackgroundService` or
  `server.add_service`.
- Moving server-owned loops (config watcher, health
  checks, TLS cert watcher) onto this API. Those
  are process/listener lifetime, not filter
  lifetime.
- Making `from_config` async or requiring a Tokio
  runtime at config parse time.
- Changing request-path `HttpFilter` methods.
- Specifying thread count, shared vs per-pipeline
  runtime, or the exact trait shape. That is How?
- Shipping Azure AD / Entra token refresh. That
  feature may *use* this API; it is not this API.

## Why?

### Motivation

Praxis swaps filter pipelines at runtime. Anything
a filter starts in `from_config` must stop when
that pipeline is dropped, or reload leaks work and
credentials.

The filter trait does not say that. So authors copy
the one working example: `intelligent_route`
overlay reload, which spawns
`routing-overlay-watcher` (OS thread + private
Tokio runtime + `CancellationToken` + bounded
join). That shape exists because overlay reload is
an inotify loop that must not block pipeline build.
It is also the *only* filter on `main` (core, ai,
extproc) that owns a thread.

The next consumer does not look like inotify. An
Entra client-credentials filter is a timer plus one
HTTP POST, then inject `Authorization`. It still
cannot `tokio::spawn` (`from_config` is sync, often
off-runtime) and cannot register a Pingora
background service (process-lifetime, not torn down
on pipeline swap). Without a pipeline-scoped API,
it copies the overlay thread anyway — including the
weaker variant that logs spawn failure and still
constructs a filter that fail-closes forever. That
copy is already in flight as
[praxis-proxy/ai#815](https://github.com/praxis-proxy/ai/pull/815).

Inventory today:

| Owner | Lifetime | Mechanism |
| --- | --- | --- |
| `intelligent_route` (overlay + reload) | pipeline | private thread + runtime |
| TLS cert watcher | listener / process | server-owned thread |
| Config watcher, health checks | process | server-owned thread |
| Pingora services | process | `BackgroundService` |
| Every other filter | request | none |

extproc does not add a fourth pattern. It builds an
AI filter pipeline; overlay `intelligent_route` in
that pipeline is the same thread.

Filter-owned threads are the wrong lifetime, not
just an ugly implementation:

- **Drop is not a supervisor.** The only stop
  signal is `Drop` of the filter, which is
  synchronous. Overlay and the Entra draft both
  cancel, wait a couple of seconds, then detach.
  The thread can outlive the pipeline with
  whatever it captured (client secret, token,
  HTTP client). There is no process-level
  shutdown join.
- **Reload multiplies them.** Each listener
  pipeline that constructs the filter gets a
  thread. A config reload constructs the new
  pipeline before the old `Drop` runs, so two
  copies overlap. Repeated reloads plus a timed-out
  join accumulate detached threads.
- **A private runtime is a second scheduler.**
  Pingora already runs worker threads. Each filter
  thread brings its own current-thread Tokio
  runtime (timer wheel, I/O driver) for work that
  is usually a sleep and one HTTP POST. That cost
  is paid per filter instance, not per process.
- **Spawn failure is invisible.** `from_config`
  can return `Ok` after the thread or runtime
  fails to start. The filter is live; the cache
  stays empty; every request 503s (or worse, if
  someone fail-opens). Overlay at least fails
  construction when the watcher never comes up.
  Copies do not have to.
- **There is no shared budget or signal.** Each
  filter invents its own `CancellationToken`, join
  timeout, and `warn!`. Operators cannot list,
  cap, or drain “background work for this
  pipeline.” Tests have to stub a fake handle so
  construction does not spawn a real thread.


### User Stories

- As a filter author, I want to start
  pipeline-scoped background work without creating
  a thread or runtime so that credential refresh
  and file watch stay in-tree without copying
  overlay internals.
- As a filter author, I want spawn failure to fail
  pipeline build so that a dead refresher cannot
  ship as a live 503-forever filter.
- As an operator, I want hot reload to stop the old
  pipeline's background work so that refreshers and
  watchers do not accumulate across config swaps.
- As a maintainer, I want one place that owns
  filter background lifetime so that AI, core, and
  extproc do not grow independent thread+runtime
  copies.
