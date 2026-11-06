---
issue: https://github.com/praxis-proxy/praxis/issues/807
status: proposed
repos:
  - praxis
  - ai
authors:
  - cdoern
graduation_criteria:
  - Lifetime boundary against ENH #1042 agreed
  - How? section listing the implementing PRs
stakeholders:
  - shaneutt
  - leseb
  - aslakknutsen
experimental_exempt: true
experimental_exempt_reason: "Server and filter-framework infrastructure, with no filter to prototype in the experimental repo."
related:
  - 01042
  - https://github.com/praxis-proxy/praxis/issues/1042
---

# Background Jobs

## What?

A bounded job queue in core: a worker pool, a
per-job timeout, and cancellation. A filter
enqueues work that outlives the request that
created it, and a worker later runs that work
through a configured filter chain.

Probes, the other half of #807, are not in scope.
"Long-running programs with a message-passing
interface" shares nothing with jobs beyond
supervision, and bundling the two would couple
unrelated designs. They deserve their own proposal.

### What already exists

Most of the execution problem is already solved.
`FilteredSubrequestExecutor` runs a request through
an `Arc<FilterPipeline>` with no client connection
attached: the request phase, `router`,
`load_balancer`, the upstream exchange, then the
response phase, bounded by a deadline and a
response-size ceiling. An existing test in
`crates/filter/src/filtered_subrequest/tests.rs`
drives exactly that against a real backend today,
and `IrrStepRunner` is a working consumer in about
a hundred lines.

So this is not a proposal for an execution engine.
It is a proposal for the parts missing around one:
a queue, workers, and a lifetime.

### Goals

- Enqueue work that outlives its originating
  request, bounded by queue depth, payload bytes,
  worker count, and a per-job timeout.
- Run each job through an operator-configured
  chain, so routing, guardrails, persistence and
  observability apply as configured.
- Survive a config reload. A background job that
  dies because an operator edited an unrelated
  listener is a bug.
- Drain on shutdown, or move jobs to a documented
  terminal state.

### Non-Goals

- Probes.
- Replacing Pingora `BackgroundService` for
  process-lifetime work such as listeners and
  admin HTTP.
- Replacing ENH #1042, the pipeline-scoped filter
  task supervisor, which solves an adjacent problem
  described below.
- Durable jobs. Core has no persistence layer, and
  a queue is the wrong place to introduce one.
- Distributed or multi-process scheduling.

## Why?

### Motivation

Core runs no queued work, and every consumer that
has needed background work has grown its own
mechanism:

| Owner | Lifetime | Mechanism | Shutdown |
| --- | --- | --- | --- |
| Pingora services | process | `BackgroundService` | `ShutdownWatch` |
| Config and cert watchers | process | thread plus private runtime | token, or none |
| Health checks | process | thread plus private runtime | token, best effort |
| Samplers and circuit eviction | process | shared housekeeping thread | none |
| `intelligent_route` overlay | pipeline | thread plus private runtime | `Drop` |
| `cloud_events` delivery | request | detached `tokio::spawn` | none |

Six mechanisms, four different shutdown stories,
and no queue anywhere. ENH #1042 addresses the
pipeline row. Nothing in the rest of the table is a
queue, and nothing in it is reusable as one.

The concrete consumer is OpenAI Responses
background mode (praxis-proxy/ai#32): accept a
request, return it as queued immediately, run it
later through the configured path, and let the
client poll or cancel it. The AI gateway cannot
build that on its own without either spawning a
private runtime, which is the pattern 01042 exists
to stop, or reaching into server internals it has
no access to.

### Relationship to ENH #1042

The two are complementary, and the boundary is
lifetime.

ENH #1042 gives a filter background work that lives
as long as the pipeline that built it, and cancels
it when that pipeline drops. That is what a token
refresher or a file watcher wants.

A queue wants the opposite. Cancelling on pipeline
drop means a config reload kills every in-flight
job, and a five-minute background inference should
not die because an operator edited an unrelated
listener. Jobs therefore want process lifetime and
a reload-durable registry, which is the pattern
`KvStoreRegistry`, `SessionStoreRegistry` and
`SlowStartRegistry` already follow.

This is the part of the proposal most worth
arguing with, and the reason aslakknutsen is listed
as a stakeholder.

### User Stories

- As a filter author, I want to enqueue work that
  outlives the request, so that an API can accept
  now and execute later.
- As a filter author, I want that work to traverse
  a configured chain, so that routing, guardrails
  and persistence still apply to it.
- As an operator, I want queue depth, worker count
  and timeout to be bounded and configurable, so
  that background work cannot exhaust the proxy.
- As an operator, I want a config reload or a
  shutdown to have a defined effect on in-flight
  jobs.
