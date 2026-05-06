# Multi-Tenant Admin Dashboard

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> A super-admin console for SaaS operators that unifies tenant management, quotas, billing, and safe impersonation in one place.

Multi-Tenant Admin Dashboard is an AI-native, open-source super-admin platform for SaaS companies. It gives operations, support, and revenue teams a single pane of glass for managing tenants across identity, billing, quotas, and operational health — concerns currently scattered across WorkOS, Stripe, Qrvey, and bespoke internal tools.

---

## Why Multi-Tenant Admin Dashboard?

- The "super admin" category is unowned. Identity (WorkOS, Logto), billing (Stripe), and analytics (Qrvey, Embeddable) are each well-served, but no vendor unifies them into one operator-facing dashboard.
- Assembling a capable super-admin stack from best-of-breed tools is expensive: embedded analytics platforms charge $5K–$50K+/year, and Stripe takes a percentage of revenue, while admin UI work is still custom-built per company.
- WorkOS does not cover billing or operational metrics. Logto requires engineering to build admin UI on top. Keycloak and Ory require deep IAM and Kubernetes expertise.
- Open-source admin templates such as AdminLTE and React Admin have no multi-tenancy logic; framework-specific starters (SaaSykit, Bullet Train) are tied to PHP or Rails.
- SOC 2-aligned buyers increasingly require impersonation with full audit trails, and usage-based pricing is driving demand for per-tenant quota and entitlement management — neither is well covered by existing tools.

---

## Key Features

### Super-Admin Console

- Tenant list, search, and detail view across the entire estate
- Tenant activation and deactivation controls
- Custom fields and metadata for tenants
- Bulk operations such as bulk activation, deactivation, and user export

### Identity, Access & Impersonation

- RBAC with super-admin, tenant-admin, and end-user role definitions
- SSO / OIDC support for tenant authentication
- Safe impersonation with full audit trail and session isolation (no data modification)
- Directory Sync (SCIM) for automated user provisioning
- Delegated admin portal so tenant IT teams can manage their own users
- MFA support and single sign-out across user sessions

### Tenant Isolation & Authorization

- Row-level security or schema-per-tenant architecture
- Fine-grained authorization (FGA) for complex dynamic policies
- Audit log viewer covering all administrative actions

### Billing, Quotas & Usage

- Webhook event stream for billing and subscription changes (Stripe integration)
- Quota enforcement per tenant: seat count, API call limits, feature entitlements
- Tenant usage dashboard: API calls, active users, feature adoption
- Quota usage trending and alerts

### Operational Intelligence

- Impersonation context brief summarising recent activity, open issues, and billing status
- AI-powered tenant health scoring combining billing, adoption, and support signals
- Anomaly detection for unusual quota consumption, permission changes, and API spikes
- Natural-language tenant search and querying
- Automated remediation suggestions for common issues

---

## AI-Native Advantage

AI is used to close gaps no incumbent addresses. A composite tenant health score blends billing signals, feature adoption, support ticket volume, and usage trends into a single risk number per tenant. Natural-language querying lets ops teams ask questions like "show me all Pro tenants inactive for 30 days" without SQL, and anomaly detection flags unusual quota consumption, API spikes, and unexpected permission changes in real time. Before a support agent impersonates a tenant, an AI-generated brief surfaces recent activity, open issues, and billing status so the session starts with full context.

---

## Tech Stack & Deployment

The project targets both self-hosted (Docker Compose for a turnkey setup, contrasting with the Kubernetes complexity of Keycloak and Ory) and managed cloud deployment. It builds on established standards: OIDC / OAuth 2.0 for tenant authentication, SCIM for directory sync, RBAC with optional FGA for authorization, and PostgreSQL row-level security for tenant data isolation. Stripe is the integration point for billing webhooks and entitlements. Audit logging is designed to meet SOC 2 and ISO 27001 expectations for impersonation traceability.

---

## Market Context

Multi-tenant architecture underpins virtually all SaaS companies, but the tooling market is fragmented across identity, billing, and analytics with no funded vendor owning the super-admin category. WorkOS has raised $80M+ as the enterprise-readiness layer; Auth0 was acquired by Okta for $6.5 billion; embedded analytics tools charge $5K–$50K+/year. Primary buyers are founding engineers at Series A/B SaaS companies, support escalation teams needing safe impersonation, RevOps teams tracking quota and billing per tenant, and infrastructure leads managing per-tenant resource limits.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
