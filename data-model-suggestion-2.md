# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Multi-Tenant Admin Dashboard · Created: 2026-05-24

## Philosophy

A multi-tenant admin dashboard is, at its core, an audit machine. Every tenant activation, role assignment, impersonation session, quota change, and billing event must be traceable, timestamped, and attributable. SOC 2 Type 2 auditors don't ask "what are the current permissions?" — they ask "who granted this permission, when, and was MFA active at the time?" Event sourcing makes the audit trail the source of truth rather than an afterthought.

Every administrative action is an immutable event: tenant created, user invited, role assigned, quota increased, impersonation started, SSO configured. The current state of any entity — a tenant's active status, a user's role set, a quota's usage level — is a materialised read model derived by projecting events forward. This means the system can reconstruct any point-in-time state: "what were tenant X's quotas on March 15th?" is a direct event query.

For impersonation specifically, the event stream eliminates the need for a separate impersonation audit table. The sequence of events `impersonation.started → page.viewed → setting.read → impersonation.ended` provides a complete session record, and the `metadata` field on each event carries the impersonation context.

**Best for:** Platforms where audit trail completeness is the primary requirement, where SOC 2 / ISO 27001 auditors need to verify the entire history of access control changes, and where compliance-as-evidence is built into the data model.

**Trade-offs:**
- **Pro:** Complete audit trail — every admin action is an immutable event
- **Pro:** Point-in-time state reconstruction for any entity
- **Pro:** Impersonation audit is automatic, not a separate feature
- **Pro:** Event stream feeds AI-powered anomaly detection
- **Con:** Read model eventual consistency
- **Con:** Higher storage from append-only events
- **Con:** Event schema versioning required as the platform evolves
- **Con:** More complex application code for projections

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents 1.0 | Event envelope follows CloudEvents spec (type, source, time, data) |
| SOC 2 TSC CC6.1 | Access control history is the event stream itself |
| SOC 2 TSC CC7.2 | System monitoring via event-driven anomaly detection |
| ISO 27001 A.5.15 | Access control changes recorded as events with actor attribution |
| OAuth 2.0 (RFC 6749) | SSO configuration changes as events; token exchange for impersonation |
| SCIM 2.0 (RFC 7643) | Directory sync operations recorded as events |
| NIST SP 800-63-4 | AAL context (MFA active) recorded in event metadata |

---

## Event Store

```sql
CREATE TABLE event_store (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID,
    -- NULL for platform-level events
    stream_type TEXT NOT NULL CHECK (stream_type IN (
        'tenant', 'user', 'role', 'quota', 'billing',
        'impersonation', 'sso', 'scim', 'feature_flag'
    )),
    stream_id UUID NOT NULL,
    sequence_number BIGINT NOT NULL,
    event_type TEXT NOT NULL,
    ce_source TEXT NOT NULL DEFAULT '/admin-dashboard',
    ce_specversion TEXT NOT NULL DEFAULT '1.0',
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}',
    -- {"actor_id": "uuid", "actor_type": "user", "ip_address": "...",
    --  "mfa_active": true, "impersonation_session_id": null}
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_events_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, occurred_at DESC);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at DESC);
```

### Event Type Registry

```sql
CREATE TABLE event_type_registry (
    event_type TEXT PRIMARY KEY,
    stream_type TEXT NOT NULL,
    description TEXT NOT NULL,
    schema_version INT NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

INSERT INTO event_type_registry (event_type, stream_type, description) VALUES
-- Tenant lifecycle
('tenant.created',              'tenant',         'Tenant created'),
('tenant.activated',            'tenant',         'Tenant activated'),
('tenant.suspended',            'tenant',         'Tenant suspended'),
('tenant.deactivated',          'tenant',         'Tenant deactivated'),
('tenant.plan_changed',         'tenant',         'Tenant plan upgraded or downgraded'),
('tenant.metadata_updated',     'tenant',         'Tenant custom metadata changed'),
('tenant.data_exported',        'tenant',         'Tenant data exported (GDPR)'),
('tenant.data_deleted',         'tenant',         'Tenant data deleted (GDPR)'),
-- User lifecycle
('user.invited',                'user',           'User invited to tenant'),
('user.activated',              'user',           'User account activated'),
('user.suspended',              'user',           'User suspended'),
('user.deactivated',            'user',           'User deactivated'),
('user.mfa_enabled',            'user',           'User enabled MFA'),
('user.mfa_disabled',           'user',           'User disabled MFA'),
('user.login',                  'user',           'User logged in'),
('user.login_failed',           'user',           'Failed login attempt'),
-- Role & permission lifecycle
('role.created',                'role',           'Role created'),
('role.updated',                'role',           'Role definition updated'),
('role.deleted',                'role',           'Role deleted'),
('role.assigned',               'role',           'Role assigned to user'),
('role.revoked',                'role',           'Role revoked from user'),
('role.permission_added',       'role',           'Permission added to role'),
('role.permission_removed',     'role',           'Permission removed from role'),
-- Quota lifecycle
('quota.allocated',             'quota',          'Quota allocated to tenant'),
('quota.updated',               'quota',          'Quota limit changed'),
('quota.usage_recorded',        'quota',          'Quota usage snapshot recorded'),
('quota.threshold_reached',     'quota',          'Quota threshold alert triggered'),
('quota.exceeded',              'quota',          'Quota hard limit exceeded'),
-- Billing lifecycle
('billing.subscription_created','billing',        'Stripe subscription created'),
('billing.subscription_updated','billing',        'Subscription plan or status changed'),
('billing.invoice_paid',        'billing',        'Invoice paid'),
('billing.payment_failed',      'billing',        'Payment failed'),
('billing.subscription_canceled','billing',       'Subscription canceled'),
-- Impersonation lifecycle
('impersonation.started',       'impersonation',  'Support agent started impersonation'),
('impersonation.action',        'impersonation',  'Action performed during impersonation'),
('impersonation.ended',         'impersonation',  'Impersonation session ended'),
-- SSO lifecycle
('sso.connection_created',      'sso',            'SSO connection configured'),
('sso.connection_activated',    'sso',            'SSO connection activated'),
('sso.connection_failed',       'sso',            'SSO connection test failed'),
('sso.connection_disabled',     'sso',            'SSO connection disabled'),
-- SCIM lifecycle
('scim.connection_created',     'scim',           'SCIM endpoint configured'),
('scim.sync_completed',         'scim',           'SCIM sync completed'),
('scim.sync_failed',            'scim',           'SCIM sync failed'),
('scim.user_provisioned',       'scim',           'User provisioned via SCIM'),
('scim.user_deprovisioned',     'scim',           'User deprovisioned via SCIM'),
-- Feature flag lifecycle
('feature.enabled',             'feature_flag',   'Feature flag enabled for tenant'),
('feature.disabled',            'feature_flag',   'Feature flag disabled for tenant'),
('feature.rollout_changed',     'feature_flag',   'Feature flag rollout percentage changed');
```

---

## Snapshots & Projections

```sql
CREATE TABLE stream_snapshots (
    stream_id UUID NOT NULL,
    stream_type TEXT NOT NULL,
    sequence_number BIGINT NOT NULL,
    snapshot_data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT PRIMARY KEY,
    last_event_id UUID NOT NULL,
    last_sequence BIGINT NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status TEXT NOT NULL DEFAULT 'running'
);
```

---

## Materialised Read Models

### Tenant Status

```sql
CREATE TABLE rm_tenants (
    id UUID PRIMARY KEY,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL,
    status TEXT NOT NULL,
    data_residency TEXT NOT NULL DEFAULT 'us',
    user_count INT NOT NULL DEFAULT 0,
    active_users_30d INT NOT NULL DEFAULT 0,
    mrr_cents BIGINT NOT NULL DEFAULT 0,
    health_score REAL,
    onboarded_at TIMESTAMPTZ,
    metadata JSONB NOT NULL DEFAULT '{}',
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_tenants_status ON rm_tenants(status);
CREATE INDEX idx_rm_tenants_plan ON rm_tenants(plan);
```

### User & Role Status

```sql
CREATE TABLE rm_users (
    id UUID PRIMARY KEY,
    tenant_id UUID,
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    status TEXT NOT NULL,
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    roles TEXT[] NOT NULL DEFAULT '{}',
    last_login_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_users_tenant ON rm_users(tenant_id);
CREATE INDEX idx_rm_users_email ON rm_users(email);

CREATE TABLE rm_role_assignments (
    user_id UUID NOT NULL,
    role_id UUID NOT NULL,
    role_slug TEXT NOT NULL,
    tenant_id UUID,
    granted_by UUID,
    granted_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (user_id, role_id)
);
```

### Quota Status

```sql
CREATE TABLE rm_quota_status (
    tenant_id UUID NOT NULL,
    quota_slug TEXT NOT NULL,
    allocated_limit BIGINT NOT NULL,
    current_usage BIGINT NOT NULL DEFAULT 0,
    usage_pct REAL NOT NULL DEFAULT 0,
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    last_alert_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, quota_slug)
);
```

### Billing Status

```sql
CREATE TABLE rm_billing_status (
    tenant_id UUID PRIMARY KEY,
    stripe_customer_id TEXT NOT NULL,
    plan TEXT NOT NULL,
    billing_status TEXT NOT NULL,
    mrr_cents BIGINT NOT NULL DEFAULT 0,
    trial_ends_at TIMESTAMPTZ,
    current_period_end TIMESTAMPTZ,
    last_payment_at TIMESTAMPTZ,
    last_payment_status TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Impersonation Log

```sql
CREATE TABLE rm_impersonation_log (
    session_id UUID PRIMARY KEY,
    actor_id UUID NOT NULL,
    actor_name TEXT NOT NULL,
    target_tenant_id UUID NOT NULL,
    target_tenant_name TEXT NOT NULL,
    reason TEXT NOT NULL,
    status TEXT NOT NULL,
    action_count INT NOT NULL DEFAULT 0,
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    duration_seconds INT
);

CREATE INDEX idx_rm_impersonation_tenant ON rm_impersonation_log(target_tenant_id, started_at DESC);
CREATE INDEX idx_rm_impersonation_actor ON rm_impersonation_log(actor_id, started_at DESC);
```

### SSO & SCIM Status

```sql
CREATE TABLE rm_sso_status (
    tenant_id UUID NOT NULL,
    connection_id UUID NOT NULL,
    provider_type TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL,
    domain_hint TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, connection_id)
);

CREATE TABLE rm_scim_status (
    tenant_id UUID NOT NULL PRIMARY KEY,
    connection_id UUID NOT NULL,
    status TEXT NOT NULL,
    last_sync_at TIMESTAMPTZ,
    last_sync_status TEXT,
    users_synced INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Reference Data

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    email TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    slug TEXT NOT NULL,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource TEXT NOT NULL,
    action TEXT NOT NULL,
    UNIQUE (resource, action)
);

CREATE TABLE quota_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    unit TEXT NOT NULL,
    default_limit BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'disabled',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
```

---

## Example Queries

### Reconstruct tenant state at a point in time

```sql
SELECT event_type, event_data, metadata, occurred_at
FROM event_store
WHERE stream_type = 'tenant'
  AND stream_id = 'tenant-uuid'
  AND occurred_at <= '2026-03-15T00:00:00Z'
ORDER BY sequence_number;
```

### Impersonation session timeline

```sql
SELECT event_type,
       event_data->>'action' AS action,
       event_data->>'resource_type' AS resource,
       metadata->>'actor_id' AS actor,
       occurred_at
FROM event_store
WHERE stream_type = 'impersonation'
  AND stream_id = 'session-uuid'
ORDER BY sequence_number;
```

### Permission change history for a user

```sql
SELECT event_type,
       event_data->>'role_slug' AS role,
       metadata->>'actor_id' AS granted_by,
       occurred_at
FROM event_store
WHERE stream_type = 'role'
  AND event_type IN ('role.assigned', 'role.revoked')
  AND event_data->>'user_id' = 'user-uuid'
ORDER BY occurred_at;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store, event_type_registry, stream_snapshots |
| Projections | 1 | projection_checkpoints |
| Read Models | 8 | rm_tenants, rm_users, rm_role_assignments, rm_quota_status, rm_billing_status, rm_impersonation_log, rm_sso_status, rm_scim_status |
| Reference Data | 7 | tenants, users, roles, permissions, quota_definitions, feature_flags, api_keys |
| **Total** | **19** | |

---

## Key Design Decisions

1. **Admin actions as events** — Every admin operation (tenant activation, role assignment, quota change, impersonation start) is an immutable event. The audit trail IS the data model, not a side effect of it.

2. **Impersonation as event stream** — `impersonation.started`, `impersonation.action`, `impersonation.ended` events provide a complete session record. The `metadata.impersonation_session_id` on subsequent events links any action back to the impersonation context.

3. **MFA context in event metadata** — Every event's `metadata` includes `mfa_active: true/false`, satisfying NIST SP 800-63-4 AAL2 verification. Auditors can query for admin actions performed without MFA.

4. **Permission history without a separate audit table** — `role.assigned`, `role.revoked`, `role.permission_added` events provide the complete history of who had which permissions and when. No separate `permission_change_log` needed.

5. **Quota lifecycle as events** — `quota.allocated`, `quota.updated`, `quota.threshold_reached`, `quota.exceeded` events track the full lifecycle. The `rm_quota_status` read model provides current usage; the event stream shows how it got there.

6. **Billing events from Stripe** — `billing.*` events mirror Stripe webhook events as first-class events in the store. The `rm_billing_status` read model provides current subscription state.

7. **Nullable tenant_id for platform events** — Platform-level events (super-admin role changes, feature flag rollouts) have `tenant_id = NULL`, distinguishing them from tenant-scoped events.

8. **CloudEvents compliance** — `ce_source` and `ce_specversion` on every event enable forwarding to external SIEM platforms. The event stream can feed Splunk, Datadog, or any CloudEvents-compatible system for SOC 2 monitoring.
