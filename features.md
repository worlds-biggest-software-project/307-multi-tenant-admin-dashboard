# Multi-Tenant Admin Dashboard — Feature & Functionality Survey

> Candidate #307 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| WorkOS | Commercial SaaS | Proprietary SaaS | https://workos.com/ |
| Logto | Open Source | ISC (self-hosted); Managed Cloud | https://logto.io/ |
| LoginRadius | Commercial SaaS | Proprietary SaaS | https://www.loginradius.com/ |
| Qrvey | Commercial SaaS | Proprietary SaaS | https://qrvey.com/ |
| Stripe | Commercial SaaS | Proprietary SaaS (% revenue-based) | https://stripe.com/ |
| Keycloak | Open Source | Apache 2.0 | https://www.keycloak.org/ |
| Ory | Open Source / Commercial | Hybrid (open-source + cloud) | https://www.ory.sh/ |

## Feature Analysis by Solution

### WorkOS

**Core features**
- Enterprise-ready SSO (SAML, OIDC, OAuth)
- Directory Sync (SCIM) for automated user provisioning
- Role-Based Access Control (RBAC) with custom role definitions
- Fine-Grained Authorization (FGA) for complex permission policies
- Admin Portal: embeddable or standalone for customer IT teams
- Multi-tenant user and role management with organization contexts
- Audit logs with comprehensive event tracking
- Flexible authorization models with role assignment from IdP groups

**Differentiating features**
- Purpose-built for SaaS enterprise readiness; admin portal included (vs generic auth providers)
- FGA extends RBAC for complex authorization scenarios
- Admin Portal can be self-serve link generation for customer IT setup
- Audit logs integrated for SOC 2 compliance
- Native multi-tenant JWT claims (organization_id in token payload)

**UX patterns**
- Embeddable authorization UI component
- No-code Admin Portal for customer IT teams
- Dashboard for tracking SSO adoption and directory sync status

**Integration points**
- SAML, OIDC, OAuth 2.0 identity providers
- SCIM directory sync
- SDKs for major frameworks (Node, Python, Ruby, Go)
- Audit log webhooks
- API for custom admin integrations

**Known gaps**
- Does not cover billing or usage-based quota management
- No built-in operational metrics (tenant health scoring, anomaly detection)
- Limited analytics beyond SSO adoption
- Admin Portal is read-only for many tenant operations; requires custom UI for impersonation

**Licence / IP notes**
- Proprietary SaaS; no open-source option

---

### Logto

**Core features**
- Open-source identity infrastructure (ISC licence)
- Built-in multi-tenancy with Organizations feature
- OIDC-native and OAuth 2.1 compliant
- Role-Based Access Control (RBAC) with role templates
- Management API for programmatic control of all identity operations
- Multi-platform support: Web, iOS, Android, native apps
- Comprehensive Management, Experience, and Account APIs
- User and organization management dashboards
- Self-hosted or managed cloud deployment

**Differentiating features**
- First-class multi-tenancy: organizations are core to data model
- Fully open-source with permissive ISC licence
- Self-hosting eliminates vendor lock-in
- Management API enables complete automation of identity lifecycle
- Fast setup optimized for TypeScript developers
- Organizations feature implements workspace model (like Slack workspaces)

**UX patterns**
- Dashboard for user and organization management
- Self-service org creation and member invitations
- Role management at org level
- JWT payload includes organization context

**Integration points**
- OIDC and OAuth 2.1 identity provider support
- Management API for custom admin UI and automation
- SDKs for JavaScript, Python, Java, Go, etc.
- Webhooks for identity events
- Self-hosted via Docker or managed cloud

**Known gaps**
- Requires engineering to build custom admin dashboard UI
- No built-in billing or quota management
- Tenant health scoring or anomaly detection not included
- Limited commercial support options vs proprietary competitors

**Licence / IP notes**
- ISC licence (permissive open-source, similar to MIT)
- Self-hosting is fully supported with no licensing restrictions
- Managed cloud tier available for teams preferring SaaS

---

### LoginRadius

**Core features**
- Federated identity and access management
- RBAC with delegated administration
- Multi-tenant support with per-tenant configuration
- SSO and MFA
- User lifecycle management
- Directory integrations (Active Directory, LDAP, Okta sync)
- Audit logging and compliance reporting

**Differentiating features**
- Delegated admin: allow tenants to manage their own roles and permissions
- Multi-tenant RBAC at organization and role level
- Directory integration automation

**UX patterns**
- Admin console for super-admin operations
- Delegated admin interface for tenant IT teams
- User self-service portal

**Integration points**
- SAML, OpenID Connect, OAuth 2.0
- Active Directory, LDAP, Google Workspace sync
- SCIM directory sync
- Custom API for integrations

**Known gaps**
- IAM layer only; no billing, quota, or operational admin features
- Limited to identity and access concerns
- No tenant analytics or health scoring
- Less modern API design vs newer entrants

**Licence / IP notes**
- Proprietary SaaS; no open-source option

---

### Qrvey

**Core features**
- Embedded analytics platform with multi-tenant security
- Row-level security (RLS) for per-tenant data isolation
- Dashboard and reporting tools
- Real-time analytics
- Custom analytics widget library
- Tenant-specific data filtering and access control

**Differentiating features**
- Purpose-built multi-tenant analytics with native RLS
- Embeddable in SaaS products
- Tenant context automatically applied to all queries

**UX patterns**
- Pre-built dashboard templates
- Self-service analytics for end users within SaaS app
- Admin reporting for operational oversight

**Integration points**
- Analytics API for programmatic dashboard creation
- Data connector API for connecting external sources
- Embeddable component library

**Known gaps**
- Analytics-only platform; does not cover identity, billing, or quotas
- No IAM or access control features
- Limited to reporting and analytics use cases

**Licence / IP notes**
- Proprietary SaaS; analytics layer with custom pricing

---

### Stripe (Billing + Customer Portal)

**Core features**
- Subscription and usage-based billing
- Per-customer payment and billing portal
- Invoice management
- Webhook events for billing changes
- Customer metadata and custom fields
- Payment method management
- Billing analytics

**Differentiating features**
- Authoritative source for billing; handles compliance and tax
- Customer portal is free and customizable
- Webhooks enable tight integration with SaaS product

**UX patterns**
- Delegated to Stripe-hosted Customer Portal or white-labeled via API
- Admin dashboard for subscription and invoice management

**Integration points**
- Webhooks for subscription events
- Billing API for custom integrations
- Customer portal embedding

**Known gaps**
- Billing only; no identity, access control, or operational metrics
- No tenant admin features beyond billing management
- Does not cover quota or entitlement enforcement

**Licence / IP notes**
- Proprietary SaaS; payment processor, revenue-percentage model

---

### Keycloak

**Core features**
- Open-source IAM server (Apache 2.0)
- SAML, OIDC, OAuth 2.0 support
- RBAC and fine-grained access control
- User federation (LDAP, Active Directory, Kerberos)
- Multi-tenancy with realm separation
- Built-in user console and admin console
- Themes and custom UI support
- Audit logging

**Differentiating features**
- Fully open-source with permissive Apache 2.0 licence
- Mature project with large community
- Theme support for UI customization
- User federation for directory-less setups
- Realms enable multi-tenancy with complete isolation

**UX patterns**
- Customizable login flows via policy-driven authentication
- Admin console for realm and user management
- User self-service portal

**Integration points**
- SAML, OIDC, OAuth 2.0 identity provider
- LDAP/Active Directory user federation
- Webhooks for event streaming
- Custom extensions via SPI (Service Provider Interface)

**Known gaps**
- Steep learning curve; requires deep IAM expertise to operate
- Admin UI is basic; customization requires developer work
- No built-in billing or quota management
- Limited commercial support compared to proprietary solutions

**Licence / IP notes**
- Apache 2.0 (permissive open-source)
- Self-hosting fully supported
- Community edition; commercial support available from Red Hat

---

### Ory

**Core features**
- Open-source identity and access management (Apache 2.0)
- Modern API-first architecture
- OIDC and OAuth 2.0
- RBAC and fine-grained authorization
- Passwordless authentication (Magic Links, WebAuthn)
- Account recovery and fraud detection
- Self-hosted or managed cloud
- Audit logging and compliance reporting

**Differentiating features**
- Cloud-native and Kubernetes-native design
- Modern, RESTful API with excellent developer experience
- Passwordless-first approach
- Zero-knowledge architecture for privacy
- Stateless design enables horizontal scaling

**UX patterns**
- API-driven administration
- Self-service account portal
- Modern authentication flows (WebAuthn, passkeys)

**Integration points**
- OIDC, OAuth 2.0, custom integrations
- Management APIs for all identity operations
- Webhooks for event-driven workflows
- SDKs for multiple languages

**Known gaps**
- Smaller community than Keycloak
- Open-source model means limited commercial support
- No built-in billing or quota management
- Admin dashboard is minimal; custom UI required

**Licence / IP notes**
- Apache 2.0 (permissive open-source)
- Self-hosting fully supported
- Managed cloud available with commercial support

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

Any multi-tenant admin dashboard must provide:

- **Super-admin console** for viewing and managing all tenants
- **Tenant isolation** at the database level (row-level security or schema separation)
- **RBAC (Role-Based Access Control)** with role definitions and permission assignment
- **SSO / Multi-IdP support** for enterprise tenant onboarding
- **Directory Sync (SCIM)** for automated user provisioning
- **Audit logging** of all administrative actions and impersonations
- **Safe impersonation** with full audit trail and session isolation
- **Quota and entitlements** enforcement per tenant (API calls, seats, features)
- **Billing and usage tracking** per tenant with webhook events
- **User and organization management** dashboard

### Differentiating Features

Features present in some solutions providing competitive advantage:

- **Fine-Grained Authorization (FGA)** — Extends RBAC for complex dynamic policies (WorkOS, Ory)
- **Delegated admin** — Allow tenant IT teams to manage their own roles without super-admin involvement (LoginRadius)
- **First-class multi-tenancy** — Organizations as core data model, native JWT claims (Logto)
- **Embedded analytics** — Tenant usage, health metrics, quota consumption dashboards (Qrvey)
- **Passwordless authentication** — WebAuthn, magic links, passkeys (Ory)
- **Admin portal self-serve** — Reduce customer onboarding friction (WorkOS)
- **Open-source with self-hosting** — Eliminate vendor lock-in and control costs (Logto, Keycloak, Ory)

### Underserved Areas / Opportunities

Gaps across multiple solutions representing genuine opportunities:

1. **Unified admin dashboard combining all concerns** — Currently fragmented across identity (WorkOS, Logto), billing (Stripe), and analytics (Qrvey). No single vendor provides super-admin view of identity + billing + usage + health.

2. **AI-powered tenant health scoring** — No solution combines billing signals, feature adoption, support ticket volume, and usage trends into a risk score. Opportunity: ML-driven dashboards identifying at-risk tenants.

3. **Anomaly detection and alerting** — No platform auto-surfaces unusual quota consumption, API spikes, or permission changes. Opportunity: real-time anomaly detection with auto-remediation suggestions.

4. **Natural-language tenant querying** — Ops teams cannot ask "show me all Pro tenants inactive for 30 days" without SQL. Opportunity: LLM-powered natural-language query interface.

5. **Impersonation context and briefs** — Before impersonation, ops teams lack quick context on tenant's recent activity, open issues, and billing status. Opportunity: auto-generated impersonation brief summarizing key signals.

6. **Cross-tenant feature rollout and canary** — No solution supports gradual feature rollouts to tenant cohorts. Opportunity: feature flag integration for canary deployments per tenant segment.

7. **Simplified self-hosting** — Keycloak and Ory require Kubernetes expertise. Opportunity: Docker Compose deployment with turnkey setup.

### AI-Augmentation Candidates

Features existing tools implement manually but where AI could dramatically improve outcomes:

- **Tenant health scoring** — Combine billing health, product adoption, support ticket sentiment, API latency, and churn signals into a composite risk score.
- **Anomaly detection** — Monitor quota consumption, API call patterns, and access permission changes; alert on statistical deviations.
- **Impersonation summarization** — Before support agent impersonates, auto-generate brief: recent activity, open issues, churn risk, billing status.
- **Natural-language tenant search** — LLM-powered query interface allowing "show all enterprise customers on annual plan who haven't logged in 60+ days."
- **Auto-remediation suggestions** — Flag common issues (failed webhook delivery, stuck payment, inactive integrations) with suggested fixes.

---

## Legal & IP Summary

No copyright, trademark, or patent conflicts identified. Logto (ISC), Keycloak (Apache 2.0), and Ory (Apache 2.0) are all permissive open-source licences with no restrictions on self-hosting or modification. Proprietary tools (WorkOS, LoginRadius, Qrvey, Stripe) are commercial products with standard SaaS terms. No material was omitted due to IP uncertainty.

---

## Recommended Feature Scope

### Must-have (MVP)

- Super-admin console with tenant list, search, and detail view
- Tenant isolation: row-level security or schema-per-tenant architecture
- RBAC: admin, tenant-admin, end-user role definitions
- SSO/OIDC support for tenant authentication
- Safe impersonation with full audit trail (no data modification allowed)
- Basic quota enforcement: seat count, API call limits per tenant
- Audit log viewer showing all administrative actions
- Tenant activation/deactivation controls
- User and org management dashboard
- Webhook event stream for billing/subscription changes (Stripe integration)

### Should-have (v1.1)

- Fine-grained authorization (FGA) for complex policies
- Directory Sync (SCIM) for automated user provisioning
- Delegated admin portal: allow tenant IT teams to manage their own users
- Tenant usage dashboard: API calls, active users, feature adoption
- Impersonation context brief: recent activity, open issues, billing status
- Quota usage trending and alerts
- Custom fields and metadata for tenants
- Bulk operations: activate/deactivate multiple tenants, bulk user export
- Two-factor authentication (MFA) support
- Single sign-out across all user sessions

### Nice-to-have (backlog)

- AI-powered tenant health scoring combining billing, adoption, and support signals
- Anomaly detection: unusual quota consumption, permission changes, API spikes
- Natural-language tenant search and querying
- Feature flag integration for gradual tenant rollout
- Passwordless authentication (WebAuthn, magic links)
- Custom admin portal branding and theming
- Advanced audit filtering and compliance reporting
- User activity heatmaps and session replay
- Automated remediation suggestions for common issues
- Open-source self-hosted option (Docker Compose)
- Advanced RBAC: attribute-based access control (ABAC)
- Cross-tenant reporting and benchmarking
