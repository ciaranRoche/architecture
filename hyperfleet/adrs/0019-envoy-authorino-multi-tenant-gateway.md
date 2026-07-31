---
Status: Proposed
Owner: HyperFleet Team
Last Updated: 2026-07-31
---

# 0019 --- Envoy + Authorino as Multi-Tenant API Gateway

## Context

HyperFleet has no tenant isolation. All authenticated API consumers see all
resources regardless of organizational boundary. Partners require tenant-scoped
resource visibility, but each deployment has different tenant dimensions (org,
project, team, user, etc.) depending on the partner's identity provider and
organizational model.

A prior spike ([multi-tenant-identity-authz-design.md](../docs/multi-tenant-identity-authz-design.md))
evaluated OPA middleware and DAO-layer filtering with dedicated `tenant_issuer` /
`tenant_value` columns. This ADR evaluates an alternative: a gateway-first
approach using Envoy as API proxy with Authorino as the external authorization
service, combined with label-based tenant filtering at the application layer.

## Decision

Use **Envoy** as the API ingress proxy and **Authorino** (Kuadrant) as the
external authorization service. Tenant isolation is enforced at two layers:

1. **Gateway layer (Envoy + Authorino):** Authenticates requests (JWT/API key),
   extracts tenant identity from token claims, and injects tenant context as
   HTTP headers (e.g., `X-Tenant-Org`). Envoy strips these headers from
   incoming requests to prevent spoofing.

2. **Application layer (tenant middleware):** Reads gateway-injected headers,
   injects tenant labels on resource creation, and appends label-based search
   filters on all queries so each tenant sees only their own resources.

Both layers are **config-driven**. Tenant dimensions are declared in:

- **Authorino `AuthConfig`**: Maps JWT claims to response headers. Different
  deployments use different `AuthConfig` CRs for different tenant models.
- **API `tenant` config (Helm values)**: Maps HTTP headers to resource label
  keys. The middleware uses this to inject and filter labels.

No schema migration is required. Tenant ownership is stored in the existing
resource labels system (`resource_labels` table) using a namespaced key
convention (e.g., `hyperfleet.io/org`).

## Consequences

**Gains:**

- Tenant model is not compiled into code. Switching from org+project to
  team+user requires changing two YAML files, not rebuilding the API.
- Gateway handles AuthN centrally. Invalid tokens are rejected before reaching
  the API, saving compute.
- Authorino provides a standard Envoy ext_authz interface that works with
  service mesh (Istio) if adopted later.
- No database migration. Labels are already indexed and searchable via the
  existing TSL search infrastructure.
- Defense-in-depth: even if a code path misses tenant filtering, the gateway
  still enforces authentication and tenant identity extraction.

**Trade-offs:**

- Requires Envoy and Authorino as infrastructure dependencies. Adds operational
  complexity versus the application-only approach.
- Label-based filtering uses `EXISTS` subqueries, not a simple column `WHERE`
  clause. For very high tenant counts with large resource tables, dedicated
  columns with composite indexes may perform better.
- Two configuration surfaces (AuthConfig + Helm values) must stay aligned.
  Misconfiguring one without the other can lead to missing tenant headers or
  unfiltered queries.
- Local development and testing require running Envoy + Authorino (or
  simulating the headers they inject).

## Alternatives Considered

| Alternative | Why Not Chosen |
|-------------|---------------|
| Application-only with dedicated `tenant_issuer` + `tenant_value` columns | Requires schema migration, couples tenant model to DB schema. Less flexible for partners with different tenant dimensions. Cannot reject bad tokens before they reach the API. |
| OPA middleware (ext_authz or in-process) | OPA cannot enforce row-level tenant isolation without a DB pre-fetch on every request. Adds Rego learning curve. Still needs DAO-layer filtering for LIST operations. |
| Istio service mesh with AuthorizationPolicy | Heavier infrastructure dependency. Istio's authorization policies are less expressive than Authorino's AuthConfig for multi-source identity and custom response injection. |
| Per-tenant API deployments | Operational overhead scales linearly with tenants. Contradicts the shared-service architecture. |
