# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Multi-Tenant Admin Dashboard · Created: 2026-05-24

## Philosophy

A multi-tenant admin dashboard is a meta-platform: it manages other platforms' tenants, users, roles, quotas, and billing. The core data model must precisely represent the relationships between tenants, their users, their role assignments, their quota allocations, their billing subscriptions, and the audit trail of every administrative action. A normalized relational model makes each of these relationships explicit and enforceable via foreign keys.

The key design challenge is the three-level hierarchy: super-admin → tenant-admin → end-user, where each level has different visibility and permission scopes. Normalized tables for roles, permissions, and role assignments make this hierarchy queryable and auditable. Quota definitions are separated from quota usage tracking, enabling both "what is this tenant entitled to?" and "how much have they consumed?" queries.

Impersonation — a critical feature for support teams — gets its own table with full session tracking, ensuring SOC 2-compliant audit trails that prove exactly who accessed which tenant, when, and what they did during the session.

**Best for:** Platforms where RBAC precision, quota enforcement accuracy, and audit trail completeness are non-negotiable, and where the admin dashboard must itself pass SOC 2 Type 2 audits.

**Trade-offs:**
- **Pro:** Foreign key relationships enforce data integrity across the tenant hierarchy
- **Pro:** RBAC model is explicit and auditable — roles, permissions, assignments are all queryable
- **Pro:** Quota definitions separated from usage enables flexible entitlement management
- **Pro:** Impersonation sessions fully traceable with action-level audit
- **Con:** 25+ tables — significant schema surface
- **Con:** Adding custom tenant metadata requires schema changes
- **Con:** Per-tenant SSO/SCIM configuration adds multiple config tables
- **Con:** Complex JOINs needed for permission checks across the hierarchy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OAuth 2.0 (RFC 6749) | Per-tenant OAuth configuration in `sso_connections`; token exchange for impersonation (RFC 8693) |
| OpenID Connect | Per-tenant OIDC issuer and client config in `sso_connections` |
| SCIM 2.0 (RFC 7643/7644) | Directory sync configuration in `scim_connections`; sync events in `scim_sync_logs` |
| JWT (RFC 7519 / RFC 9068) | Tenant context (`org_id`) embedded in JWT claims; impersonation tokens carry `act` claim |
| SAML 2.0 | Enterprise IdP federation config in `sso_connections` |
| SOC 2 TSC | Audit log design meets CC6.1 (logical access) and CC7.2 (system monitoring) criteria |
| ISO 27001 A.5.15 | Access control enforcement via normalized RBAC tables |
| NIST SP 800-63-4 | AAL2 (MFA) requirement for super-admin roles enforced via `mfa_required` on roles |
| PostgreSQL RLS | Row-level security policies reference `current_setting('app.current_tenant_id')` |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free',
    status TEXT NOT NULL DEFAULT 'active',
    -- active, suspended, deactivated, pending_setup
    logo_url TEXT,
    domain TEXT,
    data_residency TEXT NOT NULL DEFAULT 'us',
    -- us, eu, ap — for GDPR data residency
    onboarded_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_metadata (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    key TEXT NOT NULL,
    value TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, key)
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    -- NULL for super-admins (platform-level users)
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    avatar_url TEXT,
    status TEXT NOT NULL DEFAULT 'active',
    -- active, invited, suspended, deactivated
    external_id TEXT,
    -- from IdP via SCIM/OIDC
    last_login_at TIMESTAMPTZ,
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_users_email_tenant ON users(email, tenant_id);
CREATE UNIQUE INDEX idx_users_email_global ON users(email) WHERE tenant_id IS NULL;
CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_external ON users(tenant_id, external_id);
```

---

## RBAC: Roles & Permissions

```sql
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    -- NULL = platform-level role (super_admin)
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    -- system roles cannot be deleted
    mfa_required BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_roles_slug ON roles(tenant_id, slug);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource TEXT NOT NULL,
    -- tenants, users, roles, quotas, billing, audit_logs, impersonation
    action TEXT NOT NULL,
    -- create, read, update, delete, impersonate, export
    description TEXT,
    UNIQUE (resource, action)
);

CREATE TABLE role_permissions (
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

-- Seed system roles
INSERT INTO roles (slug, name, is_system, mfa_required) VALUES
('super_admin', 'Super Admin', TRUE, TRUE),
('support_agent', 'Support Agent', TRUE, TRUE),
('viewer', 'Viewer', TRUE, FALSE);
```

---

## SSO & Identity Configuration

```sql
CREATE TABLE sso_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider_type TEXT NOT NULL,
    -- oidc, saml, google_workspace, azure_ad, okta
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    -- pending, active, error, disabled
    client_id TEXT,
    client_secret_vault_ref TEXT,
    issuer_url TEXT,
    -- OIDC issuer
    metadata_url TEXT,
    -- SAML metadata URL
    saml_certificate TEXT,
    domain_hint TEXT,
    -- auto-redirect users with this email domain
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider_type)
);

CREATE TABLE scim_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    endpoint_url TEXT NOT NULL,
    bearer_token_hash TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    last_sync_at TIMESTAMPTZ,
    last_sync_status TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE scim_sync_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scim_connection_id UUID NOT NULL REFERENCES scim_connections(id) ON DELETE CASCADE,
    operation TEXT NOT NULL,
    -- user_created, user_updated, user_deactivated, group_created, group_updated
    resource_type TEXT NOT NULL,
    resource_id TEXT,
    status TEXT NOT NULL,
    -- success, error
    error_message TEXT,
    synced_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scim_sync_connection ON scim_sync_logs(scim_connection_id, synced_at DESC);
```

---

## Quotas & Entitlements

```sql
CREATE TABLE quota_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    unit TEXT NOT NULL,
    -- seats, api_calls, storage_bytes, projects, environments
    default_limit BIGINT NOT NULL,
    enforcement TEXT NOT NULL DEFAULT 'hard',
    -- hard (block), soft (warn), advisory (log only)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_quotas (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    quota_id UUID NOT NULL REFERENCES quota_definitions(id),
    allocated_limit BIGINT NOT NULL,
    override_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, quota_id)
);

CREATE TABLE quota_usage (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    quota_id UUID NOT NULL REFERENCES quota_definitions(id),
    current_usage BIGINT NOT NULL DEFAULT 0,
    period_start TIMESTAMPTZ NOT NULL,
    period_end TIMESTAMPTZ NOT NULL,
    last_updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, quota_id, period_start)
);

CREATE TABLE quota_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    quota_id UUID NOT NULL REFERENCES quota_definitions(id),
    threshold_pct INT NOT NULL,
    -- 80, 90, 100
    triggered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    acknowledged_at TIMESTAMPTZ,
    acknowledged_by UUID REFERENCES users(id)
);

CREATE INDEX idx_quota_usage_tenant ON quota_usage(tenant_id, period_start DESC);
CREATE INDEX idx_quota_alerts_tenant ON quota_alerts(tenant_id) WHERE acknowledged_at IS NULL;

INSERT INTO quota_definitions (slug, name, unit, default_limit) VALUES
('seats', 'User Seats', 'seats', 10),
('api_calls_monthly', 'API Calls per Month', 'api_calls', 100000),
('storage', 'Storage', 'storage_bytes', 5368709120),
('environments', 'Environments', 'environments', 3);
```

---

## Billing Integration

```sql
CREATE TABLE billing_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE UNIQUE,
    stripe_customer_id TEXT NOT NULL UNIQUE,
    stripe_subscription_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    billing_email TEXT,
    status TEXT NOT NULL DEFAULT 'active',
    -- active, past_due, canceled, trialing
    trial_ends_at TIMESTAMPTZ,
    current_period_start TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    mrr_cents BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE billing_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    stripe_event_id TEXT NOT NULL UNIQUE,
    event_type TEXT NOT NULL,
    -- invoice.paid, invoice.payment_failed, customer.subscription.updated, etc.
    payload JSONB NOT NULL DEFAULT '{}',
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_billing_events_tenant ON billing_events(tenant_id, processed_at DESC);
```

---

## Impersonation

```sql
CREATE TABLE impersonation_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id UUID NOT NULL REFERENCES users(id),
    -- the super-admin or support agent
    target_tenant_id UUID NOT NULL REFERENCES tenants(id),
    target_user_id UUID REFERENCES users(id),
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    -- active, ended, expired
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    ip_address INET,
    user_agent TEXT,
    read_only BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE impersonation_actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES impersonation_sessions(id) ON DELETE CASCADE,
    action TEXT NOT NULL,
    resource_type TEXT,
    resource_id TEXT,
    details JSONB NOT NULL DEFAULT '{}',
    performed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_impersonation_actor ON impersonation_sessions(actor_id, started_at DESC);
CREATE INDEX idx_impersonation_tenant ON impersonation_sessions(target_tenant_id, started_at DESC);
CREATE INDEX idx_impersonation_actions ON impersonation_actions(session_id, performed_at);
```

---

## Feature Flags & Entitlements

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'disabled',
    -- disabled, enabled_globally, percentage_rollout, tenant_list
    rollout_percentage INT DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_features (
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    feature_flag_id UUID NOT NULL REFERENCES feature_flags(id) ON DELETE CASCADE,
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    enabled_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, feature_flag_id)
);
```

---

## Tenant Health & AI

```sql
CREATE TABLE tenant_health_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    overall_score REAL NOT NULL,
    billing_score REAL NOT NULL,
    adoption_score REAL NOT NULL,
    support_score REAL NOT NULL,
    engagement_score REAL NOT NULL,
    signals JSONB NOT NULL DEFAULT '{}',
    -- {"active_users_30d": 45, "api_calls_trend": "declining", "open_tickets": 3,
    --  "days_since_last_login": 2, "mrr_cents": 9900, "payment_failures": 0}
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    alert_type TEXT NOT NULL,
    -- quota_spike, api_spike, permission_change, login_anomaly, billing_failure
    severity TEXT NOT NULL DEFAULT 'warning',
    -- info, warning, critical
    description TEXT NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'open',
    -- open, acknowledged, resolved, dismissed
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_health_scores_tenant ON tenant_health_scores(tenant_id, computed_at DESC);
CREATE INDEX idx_anomaly_alerts_tenant ON anomaly_alerts(tenant_id, status);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID,
    -- NULL for platform-level actions
    actor_id UUID NOT NULL,
    actor_type TEXT NOT NULL DEFAULT 'user',
    -- user, system, scim, api_key
    impersonation_session_id UUID REFERENCES impersonation_sessions(id),
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id, created_at DESC);
CREATE INDEX idx_audit_log_impersonation ON audit_log(impersonation_session_id)
    WHERE impersonation_session_id IS NOT NULL;
```

---

## API Keys

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    -- NULL for platform-level keys
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
```

---

## Row-Level Security

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID
           OR tenant_id IS NULL);

ALTER TABLE tenant_quotas ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON tenant_quotas
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

ALTER TABLE quota_usage ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON quota_usage
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 3 | tenants, tenant_metadata, users |
| RBAC | 4 | roles, permissions, role_permissions, user_roles |
| SSO & Identity | 3 | sso_connections, scim_connections, scim_sync_logs |
| Quotas | 4 | quota_definitions, tenant_quotas, quota_usage, quota_alerts |
| Billing | 2 | billing_accounts, billing_events |
| Impersonation | 2 | impersonation_sessions, impersonation_actions |
| Feature Flags | 2 | feature_flags, tenant_features |
| AI & Health | 2 | tenant_health_scores, anomaly_alerts |
| Audit & API | 2 | audit_log, api_keys |
| **Total** | **24** | |

---

## Key Design Decisions

1. **Three-level user hierarchy** — `users.tenant_id` is nullable: NULL for platform-level super-admins, set for tenant-scoped users. This avoids a separate `admins` table while clearly distinguishing platform operators from tenant members.

2. **Normalized RBAC** — Separate `roles`, `permissions`, `role_permissions`, and `user_roles` tables enable precise permission queries: "can user X perform action Y on resource Z?" is a single JOIN chain, and the complete permission set for any role is queryable.

3. **Quota three-table pattern** — `quota_definitions` (what quotas exist), `tenant_quotas` (what each tenant is allocated), `quota_usage` (current consumption per period). This separation enables plan-based defaults overridden per tenant, with period-based usage tracking for metered billing.

4. **Impersonation with action log** — `impersonation_sessions` tracks who impersonated which tenant and when. `impersonation_actions` logs every action taken during the session. `audit_log` references the session ID, creating a complete chain from action to impersonator.

5. **Per-tenant SSO configuration** — `sso_connections` stores per-tenant OIDC/SAML configuration, supporting the enterprise requirement that each tenant brings their own IdP. The `domain_hint` column enables auto-redirect for email domains.

6. **Billing as integration, not source** — `billing_accounts` mirrors Stripe subscription state; `billing_events` stores webhook payloads. Stripe remains the source of truth for billing; the admin dashboard reads from these tables for display.

7. **Tenant health as computed snapshots** — `tenant_health_scores` stores periodic AI-computed scores, not raw signals. The `signals` JSONB column preserves the input data for each computation, enabling score explainability.

8. **Partitioned audit log** — `audit_log` is range-partitioned by `created_at` for retention management. The `impersonation_session_id` column links any audit entry back to an impersonation session when applicable.
