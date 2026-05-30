# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Multi-Tenant Admin Dashboard · Created: 2026-05-24

## Philosophy

Multi-tenant admin dashboards must accommodate enormous variation across tenants: different SSO providers (OIDC, SAML, Google Workspace, Azure AD), different billing configurations, different quota needs, and different custom metadata fields. A JSONB hybrid places the invariant structure (every tenant has an ID, a name, a plan, a status) in relational columns while tenant-specific configuration (SSO settings, SCIM config, custom metadata, quota overrides) lives in JSONB columns.

This approach dramatically reduces table count. Instead of separate `sso_connections`, `scim_connections`, `tenant_metadata`, and `quota_definitions` tables, the tenant row itself carries `sso_config`, `scim_config`, `metadata`, and `quotas` as JSONB. GIN indexes enable containment queries across these fields.

The RBAC model uses a pragmatic hybrid: relational `roles` and `user_roles` tables for the core permission hierarchy, but `permissions` as a JSONB array on the role (since permission sets are read-heavy and rarely change individually). This eliminates the `permissions` and `role_permissions` junction tables.

**Best for:** Rapid MVP development, platforms where tenant configuration varies widely, and teams comfortable with PostgreSQL JSONB patterns who want fewer tables and simpler migrations.

**Trade-offs:**
- **Pro:** Far fewer tables — faster development, simpler schema
- **Pro:** New SSO providers, quota types, and metadata fields added without migration
- **Pro:** Tenant configuration is self-contained in one row
- **Pro:** GIN-indexed JSONB queries competitive for containment
- **Con:** JSONB fields lack database-level constraints
- **Con:** Application must validate JSONB structure
- **Con:** Complex cross-tenant JSONB queries harder to optimize
- **Con:** JSONB permission arrays less precise than relational RBAC

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OAuth 2.0 / OIDC | SSO configuration stored in `sso_config` JSONB on tenants |
| SCIM 2.0 | Directory sync config in `scim_config` JSONB on tenants |
| JWT (RFC 7519) | Tenant context in JWT claims; role permissions from `permissions` JSONB |
| SOC 2 | Audit log with JSONB `changes` column for flexible change tracking |
| PostgreSQL RLS | Row-level security with `current_setting('app.current_tenant_id')` |
| NIST SP 800-63-4 | MFA requirements stored in role `config` JSONB |

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
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"industry": "fintech", "company_size": "50-200", "csm": "jane@company.com",
    --  "custom_field_1": "value", "tags": ["enterprise", "annual"]}
    sso_config JSONB NOT NULL DEFAULT '{}',
    -- {"provider": "oidc", "issuer_url": "https://accounts.google.com",
    --  "client_id": "xxx", "client_secret_ref": "vault://sso/tenant-123",
    --  "domain_hint": "acme.com", "status": "active"}
    -- SAML: {"provider": "saml", "metadata_url": "https://...", "certificate": "...", "status": "active"}
    scim_config JSONB NOT NULL DEFAULT '{}',
    -- {"endpoint_url": "/scim/v2/tenant-123", "token_hash": "sha256:...",
    --  "status": "active", "last_sync_at": "2026-05-24T10:00:00Z", "users_synced": 150}
    quotas JSONB NOT NULL DEFAULT '{}',
    -- {"seats": {"limit": 50, "used": 32, "enforcement": "hard"},
    --  "api_calls_monthly": {"limit": 500000, "used": 123456, "period_start": "2026-05-01"},
    --  "storage_bytes": {"limit": 10737418240, "used": 2147483648, "enforcement": "soft"},
    --  "environments": {"limit": 5, "used": 3, "enforcement": "hard"}}
    billing JSONB NOT NULL DEFAULT '{}',
    -- {"stripe_customer_id": "cus_xxx", "stripe_subscription_id": "sub_xxx",
    --  "billing_email": "billing@acme.com", "status": "active", "mrr_cents": 9900,
    --  "trial_ends_at": null, "current_period_end": "2026-06-01T00:00:00Z"}
    health JSONB NOT NULL DEFAULT '{}',
    -- {"overall_score": 85.2, "billing_score": 95, "adoption_score": 72,
    --  "support_score": 88, "engagement_score": 86,
    --  "signals": {"active_users_30d": 28, "api_calls_trend": "stable", "open_tickets": 1}}
    onboarded_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_status ON tenants(status);
CREATE INDEX idx_tenants_plan ON tenants(plan);
CREATE INDEX idx_tenants_metadata ON tenants USING GIN (metadata);
CREATE INDEX idx_tenants_quotas ON tenants USING GIN (quotas);
CREATE INDEX idx_tenants_billing ON tenants USING GIN (billing);
```

---

## Users

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    -- NULL for super-admins
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    avatar_url TEXT,
    status TEXT NOT NULL DEFAULT 'active',
    external_id TEXT,
    profile JSONB NOT NULL DEFAULT '{}',
    -- {"mfa_enabled": true, "mfa_method": "totp", "timezone": "America/New_York",
    --  "locale": "en-US", "last_ip": "203.0.113.1"}
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_users_email_tenant ON users(email, tenant_id);
CREATE UNIQUE INDEX idx_users_email_global ON users(email) WHERE tenant_id IS NULL;
CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_external ON users(tenant_id, external_id);
CREATE INDEX idx_users_profile ON users USING GIN (profile);
```

---

## RBAC

```sql
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    -- NULL = platform-level role
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    config JSONB NOT NULL DEFAULT '{}',
    -- {"mfa_required": true, "max_session_hours": 8, "ip_whitelist": ["10.0.0.0/8"]}
    permissions JSONB NOT NULL DEFAULT '[]',
    -- [{"resource": "tenants", "actions": ["read", "update"]},
    --  {"resource": "users", "actions": ["read", "create", "update", "delete"]},
    --  {"resource": "impersonation", "actions": ["start"], "constraints": {"read_only": true}},
    --  {"resource": "billing", "actions": ["read"]}]
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_roles_slug ON roles(tenant_id, slug);
CREATE INDEX idx_roles_permissions ON roles USING GIN (permissions);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

---

## Impersonation

```sql
CREATE TABLE impersonation_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id UUID NOT NULL REFERENCES users(id),
    target_tenant_id UUID NOT NULL REFERENCES tenants(id),
    target_user_id UUID REFERENCES users(id),
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    context JSONB NOT NULL DEFAULT '{}',
    -- {"tenant_name": "Acme Corp", "tenant_plan": "pro", "mrr_cents": 9900,
    --  "open_tickets": 2, "last_login_days_ago": 3, "health_score": 85.2}
    -- ^ AI-generated impersonation brief
    actions JSONB NOT NULL DEFAULT '[]',
    -- [{"action": "page.viewed", "resource": "/settings", "at": "2026-05-24T10:05:00Z"},
    --  {"action": "setting.read", "resource": "billing_config", "at": "2026-05-24T10:06:00Z"}]
    read_only BOOLEAN NOT NULL DEFAULT TRUE,
    ip_address INET,
    user_agent TEXT,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ,
    duration_seconds INT
);

CREATE INDEX idx_impersonation_actor ON impersonation_sessions(actor_id, started_at DESC);
CREATE INDEX idx_impersonation_tenant ON impersonation_sessions(target_tenant_id, started_at DESC);
```

---

## Feature Flags

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    status TEXT NOT NULL DEFAULT 'disabled',
    config JSONB NOT NULL DEFAULT '{}',
    -- {"rollout_percentage": 25, "tenant_ids": ["uuid1", "uuid2"],
    --  "plan_filter": ["pro", "enterprise"], "created_by": "uuid"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
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

## Anomaly Alerts

```sql
CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    alert_type TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'warning',
    description TEXT NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    -- {"metric": "api_calls", "expected_range": [1000, 5000], "actual": 28000,
    --  "deviation_pct": 460, "detection_method": "z_score"}
    status TEXT NOT NULL DEFAULT 'open',
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at TIMESTAMPTZ
);

CREATE INDEX idx_anomaly_alerts_tenant ON anomaly_alerts(tenant_id, status);
```

---

## Billing Events

```sql
CREATE TABLE billing_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    stripe_event_id TEXT NOT NULL UNIQUE,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL DEFAULT '{}',
    processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_billing_events_tenant ON billing_events(tenant_id, processed_at DESC);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID,
    actor_id UUID NOT NULL,
    actor_type TEXT NOT NULL DEFAULT 'user',
    impersonation_session_id UUID REFERENCES impersonation_sessions(id),
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    -- {"field": "status", "old": "active", "new": "suspended"}
    -- {"field": "quotas.seats.limit", "old": 10, "new": 50}
    context JSONB NOT NULL DEFAULT '{}',
    -- {"ip_address": "203.0.113.1", "user_agent": "...", "mfa_active": true}
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
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes JSONB NOT NULL DEFAULT '[]',
    -- [{"resource": "users", "actions": ["read"]}, {"resource": "quotas", "actions": ["read"]}]
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);
```

---

## Example Queries

### Find all enterprise tenants with SSO configured but SCIM not active

```sql
SELECT name, slug, plan,
       sso_config->>'provider' AS sso_provider,
       sso_config->>'status' AS sso_status,
       scim_config->>'status' AS scim_status
FROM tenants
WHERE plan = 'enterprise'
  AND sso_config->>'status' = 'active'
  AND (scim_config->>'status' IS NULL OR scim_config->>'status' != 'active')
ORDER BY name;
```

### Quota usage dashboard

```sql
SELECT name, slug,
       (quotas->'seats'->>'used')::INT AS seats_used,
       (quotas->'seats'->>'limit')::INT AS seats_limit,
       ROUND(100.0 * (quotas->'seats'->>'used')::INT / 
             NULLIF((quotas->'seats'->>'limit')::INT, 0), 1) AS seats_pct,
       (quotas->'api_calls_monthly'->>'used')::INT AS api_calls_used,
       (quotas->'api_calls_monthly'->>'limit')::INT AS api_calls_limit
FROM tenants
WHERE status = 'active'
ORDER BY (quotas->'seats'->>'used')::REAL / NULLIF((quotas->'seats'->>'limit')::REAL, 0) DESC;
```

### Natural-language-style query: "Pro tenants inactive 30+ days"

```sql
SELECT name, slug, plan,
       (health->'signals'->>'active_users_30d')::INT AS active_users,
       (health->>'overall_score')::REAL AS health_score,
       billing->>'mrr_cents' AS mrr_cents
FROM tenants
WHERE plan = 'pro'
  AND (health->'signals'->>'active_users_30d')::INT = 0
  AND status = 'active'
ORDER BY (billing->>'mrr_cents')::BIGINT DESC;
```

### Permission check: can user X impersonate?

```sql
SELECT r.slug AS role, r.permissions
FROM user_roles ur
JOIN roles r ON r.id = ur.role_id
WHERE ur.user_id = 'user-uuid'
  AND r.permissions @> '[{"resource": "impersonation", "actions": ["start"]}]';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core | 1 | tenants (SSO, SCIM, quotas, billing, health all inline JSONB) |
| Users | 1 | users (profile as JSONB) |
| RBAC | 2 | roles (permissions as JSONB), user_roles |
| Impersonation | 1 | impersonation_sessions (actions as JSONB array) |
| Feature Flags | 2 | feature_flags, tenant_features |
| Alerts | 1 | anomaly_alerts |
| Billing | 1 | billing_events |
| Infrastructure | 2 | audit_log, api_keys |
| **Total** | **11** | |

---

## Key Design Decisions

1. **Tenant as self-contained document** — SSO config, SCIM config, quotas, billing state, and health scores are all JSONB columns on the `tenants` table. A single `SELECT * FROM tenants WHERE id = ?` returns everything needed for the tenant detail page.

2. **Permissions as JSONB array on roles** — Instead of `permissions` and `role_permissions` junction tables, the permission set is a JSONB array on each role. Permission checks use `@>` containment queries. Trade-off: no database-level enforcement of valid resources/actions, but dramatically simpler schema.

3. **Impersonation actions as JSONB array** — Low-volume action logs (typically 5-20 actions per session) stored inline. For high-volume impersonation platforms, this should be normalized to a separate table.

4. **Quota tracking inline** — `quotas` JSONB on tenants stores both limits and current usage. The application layer updates `used` counts atomically. This avoids separate `quota_definitions`, `tenant_quotas`, and `quota_usage` tables at the cost of application-level validation.

5. **Health scores inline** — `health` JSONB stores the AI-computed composite score and contributing signals directly on the tenant row. No separate health table means the tenant list query includes health data without a JOIN.

6. **Feature flag config as JSONB** — Rollout rules (percentage, tenant list, plan filter) stored as JSONB on `feature_flags`. The `tenant_features` junction table handles explicit per-tenant overrides.

7. **Audit log context as JSONB** — IP address, user agent, and MFA status stored in a `context` JSONB column rather than dedicated columns. This allows adding new context fields (e.g., geographic location) without schema changes.

8. **11 tables total** — Compared to 24 in the normalized model. The JSONB approach trades constraint enforcement for development speed and schema flexibility.
