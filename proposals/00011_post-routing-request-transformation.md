---
discussion: https://github.com/orgs/praxis-proxy/discussions/1071
issue: https://github.com/praxis-proxy/enhancements/issues/11
status: proposed
repos:
  - praxis
  - ai
authors:
  - leseb
graduation_criteria:
  - Post-routing transformation lifecycle and ownership semantics defined
  - Cluster application-protocol configuration and context propagation reviewed
  - Canonical buffered-body mutation, framing, and body-limit behavior defined
  - Standard HTTP and iterative_request_router lifecycle parity agreed
  - Compatibility and no-buffering behavior for existing filters defined
  - Praxis AI integration with the Responses-to-Chat adapter agreed
stakeholders:
  - shaneutt
  - alexsnaps
experimental_exempt: true
experimental_exempt_reason: Core lifecycle changes cannot be prototyped as an external filter.
related:
  - https://github.com/praxis-proxy/ai/issues/35
  - https://github.com/orgs/praxis-proxy/discussions/1070
  - https://github.com/orgs/praxis-proxy/discussions/840
  - https://github.com/orgs/praxis-proxy/discussions/777
---

# Post-routing Request Transformation

## What?

Allow a filter to transform a buffered HTTP request after
Praxis has selected its upstream cluster, but before the
request is sent upstream.

Upstream clusters may expose an optional application-level
protocol identifier. This identifier describes the API
protocol shared by every endpoint in the cluster, rather
than the HTTP transport protocol used to reach it.

Examples include:

- `openai_responses`
- `openai_chat_completions`
- `anthropic_messages`

Praxis core treats these values as opaque identifiers.
Application-specific filters define which identifiers they
understand and how they affect request or response
processing.

Once routing and load balancing select a cluster, its
application protocol must be available through the filter
context for the remainder of the HTTP exchange.

Filters that require it must also be able to opt into
bounded, mutable access to the canonical buffered request
body after upstream selection and before transport. Their
changes must be the bytes forwarded upstream.

The capability must have equivalent semantics in the
standard HTTP proxy lifecycle and in each internal exchange
executed by `iterative_request_router`.

Filters that do not opt into post-routing transformation
must retain their existing lifecycle and must not cause
additional request buffering.

The initial consumer is the OpenAI Responses pipeline in
Praxis AI. A single pipeline should be able to send a
Responses request unchanged to a native Responses backend,
or translate it when routing selects a backend that only
supports Chat Completions.

The shared Responses lifecycle, including validation,
rehydration, storage, and agentic-loop state, must remain
independent of the selected backend protocol.

### Goals

- Represent the application protocol shared by a cluster's
  endpoints.
- Make the selected application protocol available to
  filters for the complete upstream exchange.
- Support bounded request transformation after upstream
  selection and before transport.
- Preserve one stateful application pipeline across
  heterogeneous upstream protocols.
- Provide the same lifecycle contract for normal requests
  and iterative request-router exchanges.
- Preserve existing behavior and fast paths for filters
  that do not use the capability.

### Non-goals

- Implement OpenAI or provider-specific translation in
  Praxis core.
- Automatically detect protocols by probing upstream
  endpoints.
- Allow endpoints within one cluster to use different
  application protocols.
- Replace or introduce a routing policy.
- Reorder the existing request-body pre-read phase.
- Define cross-protocol retry or fallback after an upstream
  attempt has started.
- Require every Praxis cluster to declare an application
  protocol.

## Why?

### Motivation

`StreamBuffer` request-body filters run before normal
request filters select a cluster and endpoint. Body-derived
facts can therefore influence routing, but body transforms
cannot depend on the routing result.

Praxis AI exposes this gap when one Responses pipeline can
route to either a native Responses backend or a Chat
Completions-only backend. Translation is required only in
the latter case, after the cluster is known.

Two protocol-specific pipelines work for static mappings,
but duplicate stateful Responses configuration and cannot
cleanly support request-time selection between protocols.
Putting the decision inside an AI-specific router or adapter
would instead couple translation to one routing
implementation.

A generic post-routing boundary preserves the existing
separation of responsibilities: routing selects the
upstream, cluster configuration describes its application
protocol, and application filters adapt the request before
transport. The same contract is required for normal proxy
requests and `iterative_request_router` exchanges.

### User Stories

- As an operator, I want one Responses pipeline for native
  and Chat Completions-only backends without duplicating its
  stateful configuration.

- As a filter author, I want to adapt the upstream body from
  the selected cluster's application protocol.

- As an agentic-loop user, I want internal inference
  exchanges to use the same routing and translation contract
  as normal proxy requests.

### Related

- [Post-routing request transformation discussion]
- [Responses to Chat Completions filter]
- [API translation for non-Anthropic providers]
- [Provider fallback for same-format inference endpoints]
- [Pipeline Continuations]

[Post-routing request transformation discussion]:
  https://github.com/orgs/praxis-proxy/discussions/1071
[Responses to Chat Completions filter]:
  https://github.com/praxis-proxy/ai/issues/35
[API translation for non-Anthropic providers]:
  https://github.com/orgs/praxis-proxy/discussions/1070
[Provider fallback for same-format inference endpoints]:
  https://github.com/orgs/praxis-proxy/discussions/840
[Pipeline Continuations]:
  https://github.com/orgs/praxis-proxy/discussions/777
