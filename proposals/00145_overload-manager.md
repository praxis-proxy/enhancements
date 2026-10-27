---
issue: https://github.com/praxis-proxy/praxis/issues/145
discussion: https://github.com/orgs/praxis-proxy/discussions/1418
status: proposed
repos:
  - praxis
authors:
  - abdallahsamabd
graduation_criteria:
  - How? section with requirements and design
  - Per-resource pressure scale validated, including
    smoothing so a threshold does not flap
  - Graduated actions validated: age out idle
    connections, scale connect/idle/header timeouts,
    disable HTTP/1 keep-alive, purge unused heap,
    refuse before TLS, and /ready 503 at saturation
  - Pressure and action gauges exposed before the
    actions ship
  - Operator can relate pressure to in-flight work and
    the major consumer (connections, buffers, or a filter)
  - Ships behind an experimental flag
stakeholders:
  - shaneutt
related:
  - 00123
  - https://github.com/praxis-proxy/praxis/issues/421
  - https://github.com/praxis-proxy/praxis/issues/1408
epic: https://github.com/praxis-proxy/praxis/issues/163
---

# Overload Manager

## What?

Add an overload manager that samples how hard this
Praxis process is working, turns each resource into a
smoothed pressure value from 0 to 1, and applies
protective actions as that pressure rises.

Today Praxis sheds load only when a hard limit is
already crossed. `runtime.max_memory_bytes` compares a
background RSS sample with one byte threshold.
`runtime.shed_on_fd_pressure` refuses new work when open
file descriptors near the process limit.
`runtime.max_connections` and the per-listener cap
refuse new work once the connection count is reached.
Past any of those lines, a new HTTP request is answered
`503` with `Retry-After`, and a new TCP connection is
closed. The checks run before any filter, and refusals
are counted on `praxis_overload_rejects_total`. The
proxy is fully open, or it is refusing new work.

This proposal adds:

- Resource monitors for cgroup memory, heap, CPU
  saturation, active connection count, and open file
  descriptors. File descriptors are already sampled,
  with a reserve held back. CPU saturation means
  admitted work is waiting. PSI is that signal where
  the kernel provides it. A core at 100% with nothing
  waiting is not pressure
- One memory signal. Cgroup pressure is the working
  set: `memory.current` minus `inactive_file`, divided
  by `memory.max`, so page cache does not inflate the
  ratio. When that limit is finite, cgroup pressure
  wins, because that is the line the kernel enforces.
  Heap is a ratio against a configured maximum, and it
  is the memory signal only when no cgroup limit
  exists. Live heap sits inside RSS, so it cannot
  cross the RSS cap. If `max_memory_bytes` is set
  above a finite `memory.max`, startup warns: the RSS
  cap cannot fire before the kernel kills the process
- A pressure value per resource, from a configurable
  scaling threshold to a saturation threshold, rising
  from 0 to 1 as usage climbs through that band.
  Pressure is smoothed, so a sample sitting on the
  threshold cannot turn an action on and off
- Graduated actions driven by that pressure:
  - Age out idle upstream connections by shortening
    their idle timeout. The pool cannot be cleared
    directly
  - Purge unused heap pages back to the system. This
    is the cheap memory lever. It is not a second
    cliff past RSS
  - Scale connect, idle, and header-read timeouts
    only. A body or stream read, including a long SSE
    or LLM response, keeps its configured deadline
  - Disable downstream keep-alive for HTTP/1. The
    same knob does nothing for HTTP/2
  - Refuse a new connection before TLS, as its own
    action. The accept loop cannot be paused, and
    today's `503` happens only after TLS and header
    parse
  - At saturation, keep that post-parse `503` for
    HTTP and close new TCP connections. Work already
    admitted runs to completion. `/ready` returns
    `503` as well, so Kubernetes stops sending new
    work to this pod
- Logs and metrics for the current pressure of each
  resource, which actions are engaged, and enough
  attribution to relate that pressure to in-flight
  work and the major consumer (connections, buffers,
  or a filter). The gauges ship first. The actions
  follow, behind an experimental flag

The existing hard limits stay in place as the last
backstop. The manager is the graduated layer in front
of them.

### Goals

- Let operators set a pressure band per resource, so
  protection starts before a hard limit is crossed.
- Shed cheaper resources first (idle connections,
  unused heap, HTTP/1 keep-alive, connect and header
  deadlines) while the process can still serve.
- Leave body and stream deadlines alone, so a long
  response is not cut short because the process is
  busy.
- Refuse new work only at saturation, and let work
  already admitted run to completion. Tell Kubernetes
  this pod should receive no new work by failing
  readiness.
- Treat CPU pressure as tasks waiting, not as a busy
  core.
- Use a single memory signal. A finite cgroup limit
  preempts the RSS cap. Warn at startup when the RSS
  cap sits above that limit.
- Keep pressure smooth, so actions do not flap.
- Expose pressure, active actions, and attribution in
  logs and metrics, so operators can alert before
  refusals start and can see what to tune.
- Ship the gauges first, and the actions behind an
  experimental flag.
- Keep today's RSS, file-descriptor, and connection-cap
  refusals as the final backstop.

## Why?

### Motivation

Upstream failure and process failure are different
problems. Health checks, circuit breakers, retries, and
failover cover a sick backend. This proposal covers the
Praxis process itself running out of memory, CPU, or
connections.

If that process is killed for memory, every in-flight
request dies with it, and the traffic moves onto the
remaining replicas until they hit the same wall.
Retries, hedges, and large idle pools all add load on
the proxy, so the features that recover a single
request can be what pushes the process over.

The sheds that exist today are a cliff:

- The RSS cap and the cgroup limit disagree, and live
  heap cannot settle that. Heap is part of RSS, so a
  heap reading cannot cross the RSS cap. Unused heap
  can still be given back. `memory.current` also
  counts file cache the kernel can drop, which makes
  a pod look fuller than the process is. In
  Kubernetes the kernel kills at `memory.max`. An RSS
  cap set above that limit never fires. One memory
  signal should win, and the cgroup working set is
  that signal when a finite limit exists.
- CPU utilization is the wrong saturation signal. A
  core at 100% with no queue means the proxy is
  keeping up. Overload is admitted work waiting, and
  nothing samples that wait today.
- Open file descriptors are already a hard refuse,
  with a reserve held back. They are not yet a
  pressure band, which is where closing HTTP/1
  keep-alives matters most.
- Pressure is not attributable. Request count does not
  track memory or CPU, and AI traffic makes that
  worse: one in-flight request can dominate RSS while
  the request count stays flat. Filters have no shared
  way to report what they retain, so an operator who
  sees pressure cannot tell which buffer, pool, or
  filter to tune.
- Idle upstream connections, downstream keep-alive,
  and timeouts stay at their configured size until the
  moment of refusal. There is no milder action between
  "fully open" and "refuse everyone." Shortening every
  timeout would also cut a long stream off early.
- `praxis_overload_rejects_total` moves when work is
  already being refused. There is no series for
  pressure that is high and still climbing. Readiness
  stays successful until the process is killed, so
  Kubernetes keeps sending new work to a pod that is
  already full.

Shedding the cheaper costs as pressure rises, and
refusing new connections only at saturation, keeps this
process alive so in-flight work can finish. Failing
readiness at that point lets the cluster send new work
elsewhere. Shedding without attribution leaves the
operator unable to tune the consumer that caused the
pressure.

Two open issues sit beside this one. They do not
replace it.

- [Backpressure](https://github.com/praxis-proxy/praxis/issues/421)
  is the wider epic: per-cluster connection limits,
  queue depth, and a degradation story. This proposal
  is the process-wide graduated shed inside that
  story.
- [Capacity signals](https://github.com/praxis-proxy/praxis/issues/1408)
  wants gauges and a score so an autoscaler can add a
  replica before requests fail. The gauges here are
  those signals. That issue consumes them. It does not
  shed load itself.

Connection rate limiting
([00123](00123_connection-rate-limiting.md)) limits how
much one source IP may open. This proposal limits how
hard the process itself may run.

### User Stories

- As a platform operator, I want a configurable
  pressure band per resource so that the proxy sheds
  load gradually as usage climbs, before a hard limit
  refuses every new connection.

- As an SRE running Praxis in Kubernetes, I want
  cgroup working-set pressure to drive protective
  actions before `memory.max` kills the process, and I
  want a startup warning when the RSS cap sits above
  that limit, so an out-of-memory kill does not drop
  in-flight requests and push that traffic onto the
  remaining replicas.

- As an operator, I want CPU pressure to mean tasks
  are waiting, so a fully busy proxy that is still
  draining its queue is left alone.

- As an operator, I want idle upstream connections to
  age out, unused heap returned, and HTTP/1
  keep-alive turned off as pressure rises, so the
  process frees memory and descriptors while it can
  still serve.

- As an operator, I want connect, idle, and header
  deadlines shortened under pressure, and I want a
  long SSE or LLM stream to keep its configured read
  deadline.

- As an operator, I want new connections refused
  before TLS once pressure is saturated, and I want
  `/ready` to fail, so this pod finishes work it
  already accepted and receives no new work.

- As an SRE, I want current pressure and engaged
  actions in logs and metrics so I can alert, and so
  an autoscaler can add a replica, while the proxy is
  shedding load and before the reject counter starts
  to climb.

- As an operator, I want that pressure tied to
  in-flight work and to the major consumer holding the
  resource, so when the proxy sheds load I can see
  what to tune.
