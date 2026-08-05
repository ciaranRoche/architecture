---
Status: Active
Owner: HyperFleet Architecture Team
Last Updated: 2026-08-05
---

# 0019 — Envoy + Authorino Gateway as the Authentication Boundary

## Context

HyperFleet needs a single, trusted point where caller identity is established: human users arrive from different identity providers per deployment (with different token claims and tenant models), and internal machine clients (Sentinel, adapters) need service identity for the same API. Validating credentials inside each application couples identity-provider configuration to application code and gives human and machine traffic divergent auth paths.

## Decision

HyperFleet adopts **Envoy** as the API ingress proxy and **Authorino** (Kuadrant) as its external authorization service, and **all API traffic, external and internal, goes through this gateway**. No client, human or machine, reaches the API by any other route.

1. **Envoy strips client-supplied identity headers before authorization runs**, using an HTTP connection manager early-header-mutation extension. Route-level `request_headers_to_remove` must not be used for this: it is applied by the router filter after ext_authz, which also deletes the headers Authorino injects, and it fails silently.
2. **Authorino authenticates every caller.** Human callers present OIDC JWTs; issuers, claim mappings, and per-deployment identity models live in `AuthConfig` custom resources, not application code. Machine callers present their projected ServiceAccount tokens (audience `hyperfleet-api`), validated via Kubernetes TokenReview and restricted to an explicit subject allowlist.
3. **Authorino injects trusted identity headers** for downstream consumption (caller identity, tenant dimensions, a system marker for machine identities). Optional headers carry claim-presence conditions, since a missing claim otherwise renders as a literal `<nil>` value.
4. **The API trusts the injected headers and performs no credential validation of its own on this path.** The gateway being the only route to the API is therefore a security requirement, enforced with network policy and TLS between gateway and API, not an optional hardening step. In-app JWT validation may be retained as defense in depth, but it is not the boundary.

## Consequences

**Gains:**

- One authentication boundary for all traffic. Human and machine callers are authenticated by the same component with the same audit point, and invalid credentials are rejected before they reach any application.
- Identity configuration is deployment configuration. Adding or changing an identity provider, claim mapping, or tenant model is an `AuthConfig` change, with no application rebuild.
- Machine identity needs no new machinery: Sentinel and adapter charts already support projected ServiceAccount tokens, and TokenReview plus a subject allowlist means an unlisted in-cluster ServiceAccount with the correct audience is still denied.
- Future services behind the gateway inherit authentication instead of reimplementing it.

**Trade-offs:**

- Envoy and Authorino become required deployment dependencies on the availability path of every API call, including Sentinel and adapter traffic.
- The trust model depends on the network restriction that makes the gateway the only route in; this must be enforced and tested per deployment, not assumed.
- Identity configuration spans two surfaces (AuthConfig response headers and application header mapping) that must stay aligned. Misalignment fails closed, not open.
- Client 401/403 handling in internal services becomes load-bearing: an authorization misconfiguration must surface loudly in Sentinel and adapter metrics rather than being silently absorbed.

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| Application-only middleware | Couples identity-provider and tenant-model configuration to application code; every future service revalidates credentials; human and machine traffic take divergent auth paths. |
| Service-mesh sidecar per service | Same trust mechanics as the gateway but requires adopting a mesh first, with per-pod overhead and its own identity-propagation design. The gateway is the mesh-less form of the same boundary and can migrate to sidecars later without changing the trust model. |
| OPA (ext_authz or in-process) | A policy engine, not an authentication service: it cannot enforce row-level data isolation without a database pre-fetch per request, and it still needs something else to validate credentials and extract identity. |
| No shared boundary | Leaves every service to solve authentication independently and blocks any tenant-isolation work on a per-service basis. |
