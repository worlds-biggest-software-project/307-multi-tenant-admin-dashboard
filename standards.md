# Standards & API Reference

> Project: Multi-Tenant Admin Dashboard · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Annex A.5.15 defines access control requirements mandating the principle of least privilege and need-to-know access. Multi-tenant admin dashboards must enforce these controls for both super-admin and tenant-admin roles, with documented and regularly reviewed access grants.

**ISO/IEC 27017:2015 — Cloud Service Security Controls**
- URL: https://www.iso.org/standard/43757.html
- Extends ISO 27001 with cloud-specific controls covering multi-tenancy, data isolation, and shared responsibility. Cloud-hosted admin dashboards must address tenant isolation controls outlined here, particularly around virtual separation and monitoring.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749.html
- Defines the foundational OAuth 2.0 protocol for delegated access. Multi-tenant admin dashboards must support per-tenant OAuth flows, enabling tenants to connect their own identity providers (IdPs) via Authorization Code flow. Mandates token isolation per tenant context.

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://www.rfc-editor.org/rfc/rfc6750.html
- Defines how Bearer tokens must be transmitted. Admin dashboard APIs must enforce Bearer token authentication on all endpoints, with tenant context embedded in the token payload.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519.html
- Specifies the JWT format used for conveying identity and authorization claims between identity providers and applications. Multi-tenant dashboards embed organization or tenant identifiers as custom claims within JWTs to carry tenant context across service boundaries without additional database lookups.

**RFC 9068 — JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens**
- URL: https://www.rfc-editor.org/rfc/rfc9068.html
- Standardises the structure of OAuth 2.0 access tokens as JWTs. Relevant for multi-tenant systems requiring interoperable, structured access tokens carrying `org_id` or `tenant_id` claims.

**RFC 7643 — System for Cross-domain Identity Management (SCIM): Core Schema**
- URL: https://datatracker.ietf.org/doc/html/rfc7643
- Defines the SCIM 2.0 User and Group object schemas. Multi-tenant dashboards targeting enterprise buyers must support SCIM to enable automated user provisioning and deprovisioning by enterprise identity providers such as Okta, Azure AD, and Google Workspace.

**RFC 7644 — SCIM: Protocol**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Defines the SCIM 2.0 HTTP REST protocol for creating, reading, updating, and deleting user and group resources. The multi-tenancy section of RFC 7644 specifies three mechanisms for explicit tenant specification in SCIM endpoints.

**RFC 8693 — OAuth 2.0 Token Exchange**
- URL: https://www.rfc-editor.org/rfc/rfc8693.html
- Enables service-to-service delegation flows. Relevant for admin impersonation features where a super-admin acquires a token scoped to a specific tenant's context without knowing that tenant's credentials.

---

### OpenID Connect & Identity Specifications

**OpenID Connect Core 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- The primary standard for federated identity. Multi-tenant admin dashboards must support per-tenant OIDC configuration, allowing each tenant to register their own IdP (e.g., Okta, Azure AD, Google) for SSO. Tenant isolation in OIDC requires per-tenant issuer URLs or per-tenant client credentials.

**OpenID Connect Enterprise Extensions 1.0**
- URL: https://openid.github.io/connect-enterprise-extensions/main.html
- Draft specification extending OIDC with multi-tenant organization identity features. Defines the `tenant` claim for distinguishing personal vs organisation-managed accounts and specifies tenant identifiers within OIDC tokens. Directly relevant to representing tenant context in JWT tokens.

**SAML 2.0**
- URL: https://www.oasis-open.org/standards#samlv2.0
- Enterprise federation standard required by large corporate buyers using legacy IdPs (ADFS, Shibboleth). Multi-tenant admin dashboards must support per-tenant SAML 2.0 metadata configuration for SSO alongside OIDC.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The standard for describing HTTP REST APIs. All admin dashboard management APIs should be documented with OpenAPI 3.1 to enable SDK generation, client tooling, and AI agent integration. OAS 3.1 is now the baseline expectation for enterprise SaaS APIs.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used alongside OpenAPI for defining and validating request/response payloads. Tenant configuration objects, quota definitions, and audit log event schemas should be expressed as JSON Schema for interoperability.

**PostgreSQL Row-Level Security (RLS)**
- URL: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
- De-facto database standard for enforcing tenant data isolation at the database layer (available since PostgreSQL 9.5). RLS policies attach to tables and enforce an invisible `WHERE tenant_id = current_setting('app.current_tenant')` on every query, providing defense-in-depth isolation even when application code has bugs.

---

### Security & Authentication Standards

**NIST SP 800-63-4 — Digital Identity Guidelines**
- URL: https://pages.nist.gov/800-63-4/
- Published July 2025. Defines risk-based Identity Assurance Levels (IAL), Authenticator Assurance Levels (AAL), and Federation Assurance Levels (FAL). Admin dashboards handling sensitive tenant data and impersonation should target AAL2 (MFA required) for all super-admin operations. The revision introduces support for passkeys and digital wallets.

**OWASP Multi-Tenant Security Cheat Sheet**
- URL: https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html
- Canonical practical guidance for securing multi-tenant applications. Covers tenant context derivation from authenticated tokens, database-level isolation strategies (RLS, schema isolation), cache isolation, secure onboarding/offboarding, and cross-tenant attack prevention (IDOR, tenant impersonation, privilege escalation).

**OWASP Authorization Cheat Sheet**
- URL: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- Best practices for implementing authorization in web applications, including RBAC and attribute-based access control patterns used in admin dashboards.

**SOC 2 Trust Services Criteria (2017 with 2022 revisions)**
- URL: https://www.aicpa-cima.com/resources/download/2017-trust-services-criteria
- Governs the audit logging and access control requirements that enterprise buyers require from their SaaS vendors. Admin dashboards must implement cryptographically signed, tamper-evident audit logs covering all administrative actions (impersonations, role assignments, tenant activation/deactivation) and retain them for a minimum of 12 months to satisfy SOC 2 Type II audits.

**GDPR (EU) 2016/679 — General Data Protection Regulation**
- URL: https://gdpr.eu/
- Requires multi-tenant SaaS platforms to support per-tenant data residency preferences, data processing agreements (DPAs), and the ability to delete or export all personal data for a given tenant on request. Admin dashboards must expose tenant data management controls that enable GDPR compliance operations (right to erasure, data export).

---

## Similar Products — Developer Documentation & APIs

### WorkOS

- **Description:** Enterprise-readiness platform providing SSO, Directory Sync (SCIM), RBAC, Fine-Grained Authorization, audit logs, and an embeddable Admin Portal for SaaS products.
- **API Documentation:** https://workos.com/docs/reference
- **Admin Portal API:** https://workos.com/docs/reference/admin-portal
- **SDKs/Libraries:** Node.js, Python, Ruby, Go, Java, PHP — https://workos.com/docs
- **Developer Guide:** https://workos.com/docs/admin-portal
- **Standards:** REST/JSON; OpenAPI-documented; SAML 2.0, OIDC, SCIM 2.0
- **Authentication:** API Key (Bearer token); per-organisation portal link tokens (5-minute TTL)

---

### Logto

- **Description:** Open-source (ISC licence) identity and multi-tenant access management platform with Organizations as a first-class data model, OIDC/OAuth 2.1 compliance, and a comprehensive Management API.
- **API Documentation:** https://openapi.logto.io/
- **Management API Docs:** https://docs.logto.io/integrate-logto/interact-with-management-api
- **SDKs/Libraries:** `@logto/api` (Node.js); SDKs for JavaScript, Python, Go, Java, Swift, Kotlin — https://docs.logto.io/developers/sdk-conventions
- **Developer Guide:** https://docs.logto.io/introduction
- **Standards:** REST/JSON; OpenAPI 3.x; OIDC Core 1.0; OAuth 2.1; SCIM 2.0
- **Authentication:** Client credentials (machine-to-machine) OAuth 2.0 Bearer tokens; Management API tokens auto-refreshed by `@logto/api` SDK

---

### Frontegg

- **Description:** B2B SaaS user management and multi-tenant admin platform providing tenant (account) management APIs, self-service admin portal components, audit logs, webhooks, and API token management.
- **API Documentation:** https://developers.frontegg.com/ciam/api/overview
- **Tenant (Account) API:** https://developers.frontegg.com/api/tenants
- **SDKs/Libraries:** TypeScript/JavaScript SDK generated from OpenAPI spec; React, Next.js, Vue, Angular SDKs — https://developers.frontegg.com
- **Developer Guide:** https://developers.frontegg.com
- **Standards:** REST/JSON; OpenAPI spec available; OIDC; SAML 2.0; SCIM 2.0
- **Authentication:** Bearer token (environment-level API key); tenant-scoped tokens via `app-<tenant-id>.frontegg.com`

---

### Keycloak

- **Description:** Open-source (Apache 2.0) IAM server supporting OIDC, SAML, OAuth 2.0, RBAC, user federation (LDAP/Active Directory), multi-tenancy via realm separation, and a full Admin REST API.
- **API Documentation:** https://www.keycloak.org/docs-api/latest/rest-api/index.html
- **Admin REST API Guide:** https://www.keycloak.org/documentation
- **SDKs/Libraries:** Java Admin Client (official); community clients for Python, Go, JavaScript, .NET
- **Developer Guide:** https://www.keycloak.org/docs/latest/server_development/index.html
- **Standards:** REST/JSON; OIDC Core 1.0; SAML 2.0; OAuth 2.0; SCIM 2.0 (via extension)
- **Authentication:** Bearer token obtained via client credentials or password grant against `/realms/<realm>/protocol/openid-connect/token`

---

### Ory (Kratos + Hydra + Keto)

- **Description:** Cloud-native open-source (Apache 2.0) identity stack. Ory Kratos handles user identity and self-service flows; Ory Hydra provides OAuth 2.0/OIDC; Ory Keto provides fine-grained relationship-based access control (ReBAC).
- **API Documentation:** https://www.ory.com/docs/reference/api
- **Kratos HTTP API:** https://www.ory.com/docs/kratos/reference/api
- **SDKs/Libraries:** Go, JavaScript/TypeScript, Python, PHP, Java, .NET — https://www.ory.com/docs/ecosystem/projects
- **Developer Guide:** https://www.ory.com/docs/kratos/quickstart
- **Standards:** REST/JSON; OpenAPI 3.x (spec is source of truth); OIDC Core 1.0; OAuth 2.0/2.1; PKCE-mandatory
- **Authentication:** Admin APIs require Bearer token; Public APIs are unauthenticated (for self-service flows)

---

### Stripe (Billing & Customer Portal)

- **Description:** Payment processing and subscription billing platform. The Customer Portal provides hosted subscription and invoice management. Billing APIs enable per-tenant subscription lifecycle management, quota-driven metered billing, and webhook-driven event integration.
- **API Documentation:** https://docs.stripe.com/api
- **Billing Documentation:** https://docs.stripe.com/billing
- **Customer Portal:** https://docs.stripe.com/customer-management
- **Customer Portal API:** https://docs.stripe.com/customer-management/integrate-customer-portal
- **SDKs/Libraries:** Node.js, Python, Ruby, PHP, Go, Java, .NET — https://docs.stripe.com/libraries
- **Standards:** REST/JSON; OpenAPI spec available; webhook event schema documented
- **Authentication:** API Key (Bearer token); Restricted API Keys for per-tenant scoped access

---

### LoginRadius

- **Description:** Commercial SaaS identity and access management platform with multi-tenant RBAC, delegated administration, SSO, MFA, SCIM directory sync, and compliance audit logging.
- **API Documentation:** https://www.loginradius.com/docs/api/v2/getting-started/introduction/
- **SDKs/Libraries:** JavaScript, Node.js, Python, Java, PHP, .NET, Ruby — https://www.loginradius.com/docs/libraries/
- **Developer Guide:** https://www.loginradius.com/docs/
- **Standards:** REST/JSON; OIDC; SAML 2.0; SCIM 2.0; OAuth 2.0
- **Authentication:** API Key + API Secret; OAuth 2.0 access tokens for delegated flows

---

## Notes

**Emerging: OAuth 2.1**
OAuth 2.1 consolidates RFC 6749 with the OAuth 2.0 Security Best Current Practices and mandates PKCE for all flows while removing the implicit and resource owner password credential grants. Though not yet an official RFC, most modern identity providers (Logto, Ory) have adopted OAuth 2.1 behavior. Admin dashboard implementations should treat PKCE as mandatory.

**Emerging: Model Context Protocol (MCP)**
As AI-powered administrative features (natural-language querying, anomaly detection, impersonation briefs) are added to admin dashboards, MCP compatibility — allowing AI agents to call admin APIs as tools — will become increasingly relevant. Admin APIs documented with OpenAPI 3.1 can be exposed to MCP clients with minimal adaptation.

**Gap: No Unified Multi-Tenant Admin API Standard**
No single IETF or ISO standard defines the REST API contract for a super-admin dashboard (tenant CRUD, impersonation, quota enforcement). Each vendor defines their own schema. This creates an opportunity to publish an open OpenAPI specification for multi-tenant admin operations that the open-source community can rally around.
