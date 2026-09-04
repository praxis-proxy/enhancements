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
  - Cluster application-protocol and provider metadata propagation reviewed
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
  - https://github.com/praxis-proxy/ai/issues/924
  - https://github.com/orgs/praxis-proxy/discussions/1070
  - https://github.com/orgs/praxis-proxy/discussions/840
  - https://github.com/orgs/praxis-proxy/discussions/777
---

# Post-routing Request Transformation

## What?

Enable one HTTP filter pipeline to route requests to upstream
clusters with different application protocols while ensuring
each selected upstream receives the representation it
supports.

Clusters may declare optional, opaque application-protocol
and provider identifiers, such as `openai_responses` and
`openai`. Every endpoint in a cluster shares these values.
Application filters can use the selected upstream metadata to
preserve or adapt the outbound request before transport.
Existing pipelines that do not require adaptation retain
their current behavior.

Within `iterative_request_router`, adaptation is isolated to
each exchange and starts from that iteration's canonical
body. Adapted bytes from one exchange do not become the
source for another. Retaining the selected inference backend
across model rounds is tracked separately in
[Inference backend binding across IRR model rounds].

The initial consumer is the OpenAI Responses pipeline in
Praxis AI. A single pipeline should be able to send a
Responses request unchanged to a native Responses backend,
or translate it when routing selects a backend that only
supports Chat Completions.

Provider metadata also distinguishes native OpenAI from a
compatible backend using the same protocol. For native
OpenAI, provider-owned fields such as `conversation` and
`previous_response_id` must pass through unchanged so that
OpenAI remains responsible for validating conflicts and
existence.

This keeps validation, rehydration, storage, and
agentic-loop state independent of the backend protocol.

### Goals

- Represent and expose the selected cluster's application
  protocol and provider identity.
- Support routing-dependent adaptation only for request
  bodies already buffered within the effective request-body
  limit, and reject transformed bodies that exceed that limit
  before upstream transport.
- Provide the same selection and adaptation ordering,
  selected-cluster metadata, body-limit enforcement, HTTP
  framing, and failure behavior for normal requests and every
  iterative exchange.
- Preserve existing behavior and fast paths for filters
  that do not opt in.

### Non-goals

- Implement OpenAI or provider-specific translation in
  Praxis core.
- Infer provider identity from an endpoint address or detect
  protocols by probing upstream endpoints. Addresses are not
  stable provider identifiers, and probing would add network
  I/O, inference token costs, and failure modes for facts that
  operators already know when configuring a cluster.
- Allow endpoints within one cluster to use different
  application protocols. A transport retry may select another
  endpoint without restoring the canonical request or rerunning
  application-level transformation. Mixed-protocol endpoints
  could therefore receive a path and body encoded for another
  protocol. Each application protocol must be configured as a
  separate cluster.
- Introduce a routing policy or cross-protocol retry. Routing
  remains responsible for upstream selection; retrying across
  protocols would also require restoring the original body
  and applying a different transformation.
- Require every Praxis cluster to declare an application
  protocol. The metadata remains optional for unrelated
  clusters, while a pipeline containing a filter that depends
  on it must fail configuration validation when a selectable
  cluster omits it.

### Options under consideration

- A dedicated request-transformation callback after upstream
  selection.
- Transformation integrated into routing or upstream
  selection.
- Another lifecycle mechanism that provides the same
  metadata, isolation, and transport guarantees.

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

Protocol alone is not enough for provider-specific behavior:
native OpenAI and compatible backends may both implement the
Responses protocol. Explicit provider metadata lets filters
preserve provider-owned fields for native OpenAI without
assuming that its endpoint address is `api.openai.com`.

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
  the selected cluster's application protocol and provider.

- As an agentic-loop user, I want internal inference
  exchanges to use the same routing and translation contract
  as normal proxy requests.

### Related

- [Post-routing request transformation discussion]
- [Responses to Chat Completions filter]
- [Inference backend binding across IRR model rounds]
- [API translation for non-Anthropic providers]
- [Provider fallback for same-format inference endpoints]
- [Pipeline Continuations]

[Post-routing request transformation discussion]:
  https://github.com/orgs/praxis-proxy/discussions/1071
[Responses to Chat Completions filter]:
  https://github.com/praxis-proxy/ai/issues/35
[Inference backend binding across IRR model rounds]:
  https://github.com/praxis-proxy/ai/issues/924
[API translation for non-Anthropic providers]:
  https://github.com/orgs/praxis-proxy/discussions/1070
[Provider fallback for same-format inference endpoints]:
  https://github.com/orgs/praxis-proxy/discussions/840
[Pipeline Continuations]:
  https://github.com/orgs/praxis-proxy/discussions/777
