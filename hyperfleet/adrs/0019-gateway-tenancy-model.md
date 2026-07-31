---
Status: Proposed
Owner: HyperFleet Architecture Team
Last Updated: 2026-07-31
---

# 0019 — Gateway-First Multitenancy with a Per-Resource Tenancy Map

## Context

HyperFleet has no tenant isolation: every authenticated API consumer sees every resource. Partners need tenant-scoped visibility, and different deployments have different tenant models (an on-prem deployment might scope by org and project, an Oracle deployment by tenancy OCID and compartment). The HYPERFLEET-1165 spike recommended application-only JWT handling with fixed `tenant_issuer` and `tenant_value` columns. A working POC (2026-07-31, branches `poc/multitenancy-v2` on hyperfleet-api, hyperfleet-api-spec, and hyperfleet-infra) validated a revised design end to end, including the full pipeline running under enforcement, and this ADR records the revised decision.

## Decision

Tenant isolation is enforced in two layers, both configuration-driven so the tenant model is never compiled into code:

1. **Gateway (Envoy + Authorino) owns authentication and tenant extraction.** Envoy strips client-supplied identity headers with an HCM early-header-mutation extension before the ext_authz filter runs, then Authorino validates the caller and injects trusted headers: tenant dimension headers (from JWT claims, mapped per deployment in the AuthConfig) for humans, and `X-HyperFleet-System: true` for machine identities. Sentinel and adapters authenticate with their existing projected ServiceAccount tokens (audience `hyperfleet-api`), validated by Authorino via Kubernetes TokenReview and restricted to an explicit subject allowlist. System identities bypass tenant scoping; they need cross-tenant reads (Sentinel's global poll) and status writes.

2. **The API stores and enforces tenancy as a per-resource key-value map.** A `tenancy JSONB NOT NULL DEFAULT '{}'` column on `resources` (GIN indexed, `jsonb_path_ops`) is populated server-side at creation from the gateway-resolved dimensions, exposed read-only in responses, and immutable through the API. Every resource query is implicitly scoped at the DAO layer with parameterized JSONB containment (`tenancy @> caller_map`); cross-tenant point reads surface as 404. Containment gives hierarchy for free: an org-scoped caller sees all the org's projects. Dimensions (header name, tenancy key, required flag) are Helm configuration, and enforcement is fail-closed: a non-system request that resolves no dimensions, or is missing a required one, is rejected 403, because an empty map would contain-match every row.

This revises two positions of the HYPERFLEET-1165 spike: storage moves from fixed `tenant_issuer`/`tenant_value` columns to the flexible map (the fixed pair cannot express per-deployment models like org+project vs OCID+compartment), and the boundary moves from application-only to gateway-first (per-deployment claim mapping lives in an AuthConfig CR instead of application code, and the trex-inherited in-app JWT stack stops being the long-term authn surface). It keeps the spike's other calls: implicit DAO-layer filtering, a shared system identity for machine clients, write-path immutability, and fail-closed validation.

## Consequences

**Gains:**

- The tenant model is deployment configuration: swapping org+project for tenancy-OCID+compartment is one AuthConfig plus one Helm values change. The POC demonstrated the swap live with zero code changes; old-model tokens fail closed with 403.
- The full pipeline works under enforcement. A tenant-created cluster reached `Reconciled=True` with Sentinel and adapters authenticating through the gateway via TokenReview; isolation held before, during, and after system status writes (23/23 checks across both POC suites).
- Machine identity reuses what already ships: both charts' projected-SA-token support (previously off by default) plus TokenReview, no new credential machinery.
- No new query surface for attackers: tenant filters are parameterized JSONB containment injected at the DAO layer, never concatenated into the user-facing search language, and the tenancy map is never accepted from request bodies (schema-rejected on patch).
- Cross-tenant access denies as 404, so resource existence does not leak.

**Trade-offs:**

- Envoy and Authorino become deployment dependencies, and the API trusts gateway headers, so the deployment must guarantee the API is only reachable through Envoy. Production hardening (below) is documented, not yet built.
- JSONB containment with a GIN index is less obviously indexable than two fixed columns; at HyperFleet volumes this is not a concern, but it is a real difference from the spike's recommendation.
- Two configuration surfaces (AuthConfig response headers, API dimensions) must agree per deployment. Misalignment fails closed (403 or empty lists), not open.
- Resources created before enforcement (or by any unscoped path) carry an empty tenancy map: visible only to system identities until backfilled. Enabling enforcement on an existing deployment requires a tenancy backfill migration.

**Production hardening required before this leaves POC status:**

- NetworkPolicy (or mesh policy) forcing all API ingress through Envoy; today any in-cluster pod could hit the API service directly with forged headers.
- mTLS between Envoy and the API, and TLS on machine-client traffic (both charts default to plain `http://`).
- Re-enable in-app JWT validation as defense in depth; the per-issuer trusted-header mechanism (`identity_header`) already supports a gateway-fronted posture.
- Adapter `AlwaysAck` silently drops events on 403 (the POC hit exactly this while adapters briefly bypassed the gateway); 401/403 need distinct handling and metrics. Sentinel likewise misclassifies HTTP 403 as `fetch_error` rather than an auth error.
- Cross-tenant name collisions: the `kind,name` uniqueness index is global, so a create can 409 against another tenant's name, a small existence leak that needs a per-tenant uniqueness decision.
- Authorino renders missing claims as the literal string `<nil>` in plain response headers; optional tenant headers must always carry `when` conditions (the POC AuthConfigs do).

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| Fixed `tenant_issuer` + `tenant_value` columns (1165 spike recommendation) | Cannot express per-deployment tenant models with different dimension counts and semantics; every new partner shape becomes a schema change. |
| Reuse the user-facing labels system for tenancy | Labels are user-writable and user-searchable; tenancy is a trust boundary. The first POC attempt also showed the label search path rejects namespaced keys and is string-injected, the wrong seam for security filters. |
| Application-only JWT handling (no gateway) | Per-deployment claim mapping would live in application config and code; the gateway centralizes authn for future services, rejects invalid tokens before the API, and the in-app TLS/JWT stack is trex-inherited rather than deliberate. |
| OPA middleware | Cannot enforce row-level isolation without a DB pre-fetch per request; still requires DAO filtering for lists (established by the 1165 spike POC). |
| Per-tenant API or Sentinel deployments | Operational overhead scales linearly with tenants; contradicts the shared-service architecture and Sentinel's label-sharded design. |
| PostgreSQL row-level security as the primary mechanism | Requires per-request session variables through the connection pool; viable later as defense in depth, not as the sole mechanism. |
