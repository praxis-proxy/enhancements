---
issue: https://github.com/praxis-proxy/policy/issues/104
status: proposed
repos:
  - praxis-operator
  - https://github.com/Kuadrant/kuadrant-operator
authors:
  - maleck13
graduation_criteria:
  - Spike prototype validates PolicyFilter consumption by praxis operator
  - PPE parity gaps identified (host matching, auth methods, CEL limitations, Well Known Attributes)
  - Multi-route fan-out analysis shows acceptable resource cost
stakeholders:
  - kuadrant
  - praxis-ai
  - praxis-ppe
experimental_exempt: true
experimental_exempt_reason: Proposal involves operator changes and control-plane collaboration; prototyping directly in praxis-operator and kuadrant-operator integration is required before feasibility can be validated.
---

# Kuadrant Policy Integration with Praxis Filters

## What?

Kuadrant's policy attachment system needs a new set of APIs to enforce policies via Praxis filters instead of Istio WASM. This proposal introduces a `PolicyFilter` resource that takes the Kuadrant operator's internal concept of effective policy (compiled after inheritance and targeting resolution) and projects them into affected network resource namespaces for visibility, ordering, and consumption by the Praxis operator. Note while RateLimitPolicy is called out, it is not directly covered here yet as it is not currently available as a filter in Praxis. 

> The overall goal of this document is not to say it has to be this way but rather to start capturing the goals plus thoughts and ideas worth investigating further and prompt others to grasp the different pieces and concepts.

### Goals

- Make Kuadrant policies enforceable via Praxis filters particularly PPE (Praxis Policy Engine)
- Improve visibility for network resource owners into which policies affect their requests, what actions execute, and in what order
- Continue to support centralized policy configuration
- Support reordering when needed (e.g., auth before custom parsing)
- Maintain backward compatibility with existing Kuadrant v1 policy APIs
- Establish a control-plane collaboration model between Kuadrant and the Praxis operator

### None Goals

included in the linked epic but not a part of this proposal:
- New policy inference targets
- New rate limiting features
- Solution for a RateLimiting filter or PPE integration of RateLimiting as a plugin

## Why?

### Motivation

Kuadrant currently enforces policies only on Envoy gateways primarily via Istio using WASM filters. With the advent of Praxis as a new data plane that Kuadrant wants to support, we need a way to express and enforce policies in Praxis's filter-native model. Both Gateway API Policy Attachment and Filters have limitations that this proposal goes some way towards addressing:

**Visibility:** Effective policies resulting from hierarchal resolution are opaque. Route owners cannot easily see what policies affect them or what actions they perform without searching across multiple resources and namespaces.

**Ordering:** No mechanism allows specifying execution order for policy (e.g., auth before rate limiting or authN before payload processing and authZ after).

**Cumbersome configuration:** Filters can only attach at the route level, forcing duplicate definitions across 10 routes instead of centralized configuration of cross cutting concerns.

**Targeting tension:** As specified by GWAPI, policies target resources actively (via `targetRef`), while filters are passive (routes pull them in via `ExtensionRef`). 


Praxis's filter-native architecture offers an opportunity: almost everything is a filter including policy (PPE). By translating Kuadrant policies into Praxis policy filters, we hope to solve visibility, ordering, and configuration drift problems.

### User Stories

- As a **platform admin**, I want to define authentication requirements once at the Gateway level as my org has an approved IDP that must be used and have it apply to all routes without duplication, so I reduce configuration drift, auditing and maintenance burden and enforce authentication requirements.

- As a **route owner**, I want to see which policies affect my routes and in what order they execute, so I understand request processing and can understand policy interactions.

- As a **route owner**, I want to define route specific authorization to control access to my applications APIs that build on the existing and known authentication method

- As a **route owner**, I want to reorder my route-specific filters relative to platform policies when needed (e.g., parse request first, then authorize), so I can support request formats that policy evaluation depends on.

- As a **platform admin**, I want to enforce global rate limiting

- As a **route owner**, I want to enforce authenticated rate limiting 

## How?

### Experimental Phase

This proposal is marked `experimental_exempt: true` because it requires operator changes and control-plane collaboration. Prototyping will occur directly in Kuadrant and Praxis operator repositories:

1. **Build transpiler library:** Comprehensive AuthPolicy-to-PPE test coverage and library (existing Rust prototype; consider Go for re-use with the kuadrant operator).
2. **Gap analysis:** Document PPE limitations (host matching, auth method parity, well known attribute parity etc). (in progress)
3. **Spike in Kuadrant operator:** Implement PolicyFilter CRD and reconciler using transpiler library.
4. **Spike in Praxis operator:** Consume PolicyFilter CRs, mount spec to proxy ConfigMap, report status.
5. **Integration test:** Both operators together; AuthPolicy produces working PPE filter in Praxis.


No prototype link yet. Link `experimental_impl` once spike prototype is ready.

### Requirements

- PolicyFilter CRD supports both auth (PPE) and rate limiting filter semantics
- One PolicyFilter per affected HTTPRoute/GRPCRoute; combines all policies targeting that route into a single policy document based on effective policy reconciled by Kuadrant
- PolicyFilter is read-only when created by Kuadrant; enforced via a [VAP](https://gist.github.com/maleck13/5469c0458c0fb73f83f17fdd2edfac31)
- PolicyFilter namespace is the same as the affected route (for visibility and RBAC)
- PolicyFilter spec is in PPE-native format (plugins, global, routes sections)
- Praxis operator watches PolicyFilter CRs, applies spec to listener config, and updates status
- VAP protection prevents accidental modification/deletion of Kuadrant-managed PolicyFilters
- Policy phase defaults to `preRoute` for all policies; developers can override to `preParse` or `ingress` for their own filters
- Within a phase, PolicyFilter carries a `priority` field for ordering (lower numbers first)
- Developers can reorder their own PolicyFilters relative to each other and based on platform policies but cannot modify the platform policy (RBAC controlled)
- Secrets in policies are referenced indirectly (via secret provider config) not exposed in policy text

### Design

#### Why in the Kuadrant Operator

Kuadrant already computes effective policies by resolving the policy hierarchy. Reuse this resolution engine: leverage an AuthPolicy-to-PPE transpiler library in the Kuadrant operator to emit PolicyFilter specs. 

#### PolicyFilter Resource

**Targeting:** PolicyFilter targets are derived from Kuadrant's effective policy resolution. Host scoping is unresolved (Kuadrant uses hosts heavily; PPE does not). Options:

1. Add `host`/`hosts` to PPE's `HttpMatch`
2. Praxis operator applies host scoping via filter conditions:

```
filter_chains:
- name: example-chain
  filters:
  - filter: policy
    config_path: ./policies/api-example-com.yaml
    conditions:
   - when:
       headers:
         host: "api.example.com"


```

**Ownership and Scope**

- Created by Kuadrant controller via the transpiler library when an AuthPolicy or RateLimitPolicy (future) is reconciled into their effective policy
- Ownership: `ownerRef` can't be used reliably here as the PolicyFilter resources will often end up in different namespaces than the policies that formed them. So we will need to use a programmatic approach to clean up
- Namespace: same as the affected HTTPRoute/GRPCRoute (for visibility)
- Read-only: developers cannot modify Kuadrant-created PolicyFilters; VAP enforces this
- Developers can create their own PolicyFilters (e.g., for fine-grained control) if they have permission


**Spec Format (Auth: PreParse Example)**

PolicyFilter spec in PPE-native format. Default phase for simple header-based auth:

```yaml
apiVersion: praxis.sh/v1alpha1
kind: PolicyFilter
metadata:
  name: api-widgets-auth
  namespace: team-a
  labels:
    "kuadrant.io/managed": "true"
spec:
  plugins:
    - name: platform-jwt
      kind: identity/jwt
      config:
        issuer: auth.example.com
        # secret reference, not embedded value
        clientSecret:
          from: secret.oidc_client_secret
  global:
    authentication: [platform-jwt]
    authorization:
      pre_invocation:
        - require(authenticated)
  # Host scoping: proposed, not yet in PPE
  hosts: ["api.example.com","*.other.com"]
  routes:
    - http:
        path_prefix: /api/widgets
        method: [POST]
      authorization:
        pre_invocation:
          - cel:
              expr: "has(role.editor) && role.editor"
  phase: preParse
status:
  conditions:
    - type: Accepted
      status: "True"
      reason: Valid
    - type: Ready
      status: "True"
```

**Spec Format (Auth: PreRoute Example)**

Authorization using parsed request state (tool access control). MCP parser extracts tool metadata into headers; authorization evaluates tool permissions in PreRoute phase:

```yaml
apiVersion: praxis.sh/v1alpha1
kind: PolicyFilter
metadata:
  name: mcp-tool-access
  namespace: mcp-ns
  labels:
    "kuadrant.io/managed": "true"
spec:
  plugins:
    - name: sso-jwt
      kind: identity/jwt
      config:
        issuerUrl: https://keycloak.example.com/realms/mcp
        # OIDC discovery fetches public keys; no client secret needed for JWT validation
  global:
    authentication: [sso-jwt]
  routes:
    - http:
        path_prefix: /mcp
      authorization:
        pre_invocation:
          - cel:
              expr: |
                request.headers.exists(h, h == 'x-mcp-toolname') ?
                  ('tool:' + request.headers['x-mcp-toolname']) in 
                  (has(auth.identity.resource_access) && 
                   auth.identity.resource_access.exists(p, p == request.headers['x-mcp-servername']) ? 
                   auth.identity.resource_access[request.headers['x-mcp-servername']].roles : [])
                : true
  phase: preRoute
status:
  conditions:
    - type: Accepted
      status: "True"
      reason: Valid
    - type: Ready
      status: "True"
```


**Spec Format (Rate Limiting)**

Rate limiting filter spec is TBD; may reuse similar structure or be separate RateLimitFilter CRD.

**Filter Phases and Ordering**

PolicyFilter executes in one of three request phases:

- **Ingress** — Early gates before processing (IP controls, unauthenticated rate limits, request validation). TBD: PPE fit vs. dedicated RateLimitFilter.
- **PreParse** — Before body parsing (header-based auth, early authorization).
- **PreRoute** — After parsing, before routing (auth on parsed content, fine-grained rate limiting).

Ordering:

- Each PolicyFilter declares its `phase` (ingress, preParse, preRoute).
- Within a phase, ordered by `priority` (lower first).
- Platform PolicyFilters run before route-level PolicyFilters in the same phase.
- Developers can create PolicyFilters in any phase and set priority within that phase; cannot change platform policy phases (RBAC).

Example execution order for status in affected route:

```yaml
status:
  executionOrder:
    - phase: ingress
      filters: [platform-global-ratelimit]
      kind: PolicyFilter
    - phase: preParse
      filters: [platform-auth]
      kind: PolicyFilter      
    - phase: preRoute
      filters: [platform-authz, route-ratelimit]
      kind: PolicyFilter      
```

#### Control Plane Collaboration Model

**Praxis operator watches PolicyFilter CRs directly**

- Kuadrant creates PolicyFilter CRs in network resource namespaces via transpiler layer
- Praxis operator watches PolicyFilter CRs across all namespaces
- Praxis operator reconciles PolicyFilter spec into proxy config (ConfigMap)
- Status reflects acceptance/rejection by praxis operator


#### Visibility Mechanisms

Two layers of visibility:

1. **Route status:** Add a condition listing affected PolicyFilters and their execution phases (see above) (e.g., `preParse: [platform-auth], preRoute: [platform-authz, route-ratelimit]`)

2. **PolicyFilter in application namespace:** Reading PolicyFilter CR directly shows the effective policy, secret references (not secrets themselves), and affected resources

#### Secret Management

- Secrets referenced in policy (e.g., OIDC client secret) must not be embedded in PolicyFilter spec and become readable
- Use PPE secret provider config: reference secrets via `secret.<name>` (e.g., `secret.oidc_client_secret`)
- Praxis operator projects Kubernetes Secrets as environment variables or mounted files into the proxy
- PolicyFilter spec only contains the reference, not the value

#### Fan-Out Analysis

- Gateway-level AuthPolicy targeting a Gateway with N routes produces N PolicyFilter CRs (one per route)
- For large topologies (e.g., 200 routes), this produces 200 CRs
- Alternative: one PolicyFilter per Gateway with broad PPE conditions (reduces CRs, complexity in PPE)
- Spike should measure and recommend threshold/strategy

#### Host Matching Gap

- PPE `HttpMatch` currently lacks a `host` field (only `path`, `path_prefix`, `method`)
- Routes on the same path with different hosts cannot be separate entries in PPE
- Praxis operator uses filter `conditions` to scope by host, Kuadrant puts host-specific logic in CEL predicates (workaround)
- Spike should confirm which is feasible

#### Rate Limiting Filter

- Limitador will becomes available to Praxis via some form of filter: separate RateLimitFilter CRD or reuse PolicyFilter with `kind: rate-limit`?
- If limitador is a PPE plugin: include rate limiting config in PolicyFilter spec alongside auth
- Spike should clarify limitador architecture and recommend structure


### Next Steps - Spike Deliverables

1. **PolicyFilter CRD** 
   - Spec: plugins, global, routes (PPE format) or rate-limit-native equivalent
   - Status: Accepted/Ready conditions
   - Priority field for ordering

2. **Kuadrant reconciler** 
   - Watches limited AuthPolicy definitions, produces PolicyFilter CRs in affected route namespaces
   - Validates inheritance resolution, namespace projection, ownership
   - Updates source AuthPolicy status with reference to generated PolicyFilter

3. **Praxis operator reconciler** 
   - Watches PolicyFilter CRs
   - Translates spec into proxy config (ConfigMap or equivalent)
   - Updates PolicyFilter status (Accepted/Ready/Error)

4. **Integration test**
   - Deploy both operators
   - Apply AuthPolicy to HTTPRoute
   - Assert PolicyFilter is created in route namespace
   - Assert praxis proxy has correct PPE filter with expected config
   - Assert route request is authenticated

5. **Gap analysis document (some of this is in progress https://github.com/praxis-proxy/policy/issues/100)**
   - PPE limitations (host matching, auth methods, CEL coverage)
   - Control plane collaboration model recommendation
   - Fan-out cost and strategy
   - Rate limiting approach recommendation
   - Blockers and workarounds

### Open Questions

- **Host matching:** PPE enhancement vs filter conditions?
- **Rate limiting:** Separate filter type or PPE plugin? 
- **Fan-out:** Is N PolicyFilters per N routes acceptable, or needed optimization?
