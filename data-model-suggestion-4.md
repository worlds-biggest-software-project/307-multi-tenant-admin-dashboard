# Data Model Suggestion 4: Graph-Relational

> Project: Multi-Tenant Admin Dashboard · Created: 2026-05-24

## Philosophy

Multi-tenant admin dashboards manage a dense web of relationships: users belong to tenants, users have roles, roles grant permissions on resources, tenants own quotas, tenants connect to SSO providers, support agents impersonate tenants, feature flags target tenant cohorts. The question "who can access what, and through which path?" is fundamentally a graph traversal — not a JOIN chain through 4 junction tables.

This model uses a property graph layer (`graph_nodes` and `graph_edges`) alongside relational operational tables. The graph answers relationship-heavy queries that are the core of admin operations: "show me everything this super-admin has access to," "which tenants are affected by this feature flag?", "what's the complete permission chain from user → role → permission → resource?", and "what did this support agent have access to during their impersonation session?"

The graph is particularly powerful for Fine-Grained Authorization (FGA), which WorkOS and Ory Keto implement. Instead of flat role-permission tables, FGA models authorization as a graph: user → has_role → role → grants → permission → on_resource. The graph can express complex policies like "user X can impersonate tenant Y only if X has the support_agent role AND tenant Y is on the enterprise plan AND X's IP is in the allowlist."

**Best for:** Platforms implementing Fine-Grained Authorization (FGA), where permission chain visualization is a feature, and where impact analysis ("who is affected if we change this role?") is important.

**Trade-offs:**
- **Pro:** FGA is a natural graph — permission chains are traversals
- **Pro:** Impact analysis ("who loses access if we remove this role?") is trivial
- **Pro:** Cross-tenant access patterns visible as graph paths
- **Pro:** Permission chain visualization is a direct graph render
- **Con:** Two data access patterns (relational for CRUD, graph for authorization)
- **Con:** Graph consistency with operational tables requires careful maintenance
- **Con:** Graph traversal queries need careful depth limits
- **Con:** More complex application code

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenFGA / Zanzibar | Graph edge model inspired by Google Zanzibar / OpenFGA tuple structure |
| OAuth 2.0 / OIDC | SSO connections as graph edges from tenant to IdP nodes |
| SCIM 2.0 | Directory sync relationships as graph edges |
| SOC 2 TSC CC6.1 | Permission graph provides complete access chain for auditors |
| ISO 27001 A.5.15 | Access control enforcement via graph traversal |
| RBAC + FGA | RBAC as graph edges; FGA conditions as edge properties |

---

## Graph Layer

```sql
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type TEXT NOT NULL CHECK (node_type IN (
        'tenant', 'user', 'role', 'permission', 'resource',
        'sso_provider', 'quota', 'feature_flag', 'impersonation_session'
    )),
    ref_id UUID NOT NULL,
    label TEXT NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_type, ref_id)
);

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type TEXT NOT NULL CHECK (edge_type IN (
        'belongs_to',       -- user → tenant
        'has_role',          -- user → role
        'grants',            -- role → permission
        'on_resource',       -- permission → resource (e.g., specific tenant, all tenants)
        'authenticates_via', -- tenant → sso_provider
        'has_quota',         -- tenant → quota
        'has_feature',       -- tenant → feature_flag
        'impersonates',      -- user → tenant (via impersonation session)
        'manages',           -- user → tenant (ownership/CSM)
        'parent_of'          -- role → role (role hierarchy)
    )),
    properties JSONB NOT NULL DEFAULT '{}',
    -- has_role: {"granted_by": "uuid", "granted_at": "2026-05-01"}
    -- grants: {"actions": ["read", "update"], "constraints": {"read_only": true}}
    -- impersonates: {"session_id": "uuid", "reason": "support ticket #123", "started_at": "..."}
    -- has_quota: {"limit": 50, "used": 32, "enforcement": "hard", "unit": "seats"}
    -- authenticates_via: {"provider": "oidc", "domain_hint": "acme.com", "status": "active"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_id, target_id, edge_type)
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_ref ON graph_nodes(ref_id);
CREATE INDEX idx_graph_edges_source ON graph_edges(source_id, edge_type);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id, edge_type);
CREATE INDEX idx_graph_edges_properties ON graph_edges USING GIN (properties);
```

---

## Operational Tables

### Core

```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    plan TEXT NOT NULL DEFAULT 'free',
    status TEXT NOT NULL DEFAULT 'active',
    domain TEXT,
    data_residency TEXT NOT NULL DEFAULT 'us',
    metadata JSONB NOT NULL DEFAULT '{}',
    onboarded_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    email TEXT NOT NULL,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    external_id TEXT,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_users_email_tenant ON users(email, tenant_id);
CREATE INDEX idx_users_tenant ON users(tenant_id);
```

### RBAC

```sql
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    description TEXT,
    is_system BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_required BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX idx_roles_slug ON roles(tenant_id, slug);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);
```

### SSO & SCIM

```sql
CREATE TABLE sso_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider_type TEXT NOT NULL,
    display_name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, provider_type)
);

CREATE TABLE scim_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    endpoint_url TEXT NOT NULL,
    token_hash TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    last_sync_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Quotas

```sql
CREATE TABLE quota_definitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    unit TEXT NOT NULL,
    default_limit BIGINT NOT NULL,
    enforcement TEXT NOT NULL DEFAULT 'hard',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_quotas (
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    quota_id UUID NOT NULL REFERENCES quota_definitions(id),
    allocated_limit BIGINT NOT NULL,
    current_usage BIGINT NOT NULL DEFAULT 0,
    period_start TIMESTAMPTZ,
    period_end TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, quota_id)
);
```

### Billing

```sql
CREATE TABLE billing_accounts (
    tenant_id UUID PRIMARY KEY REFERENCES tenants(id) ON DELETE CASCADE,
    stripe_customer_id TEXT NOT NULL UNIQUE,
    stripe_subscription_id TEXT,
    plan TEXT NOT NULL DEFAULT 'free',
    status TEXT NOT NULL DEFAULT 'active',
    mrr_cents BIGINT NOT NULL DEFAULT 0,
    current_period_end TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Impersonation

```sql
CREATE TABLE impersonation_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id UUID NOT NULL REFERENCES users(id),
    target_tenant_id UUID NOT NULL REFERENCES tenants(id),
    target_user_id UUID REFERENCES users(id),
    reason TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    read_only BOOLEAN NOT NULL DEFAULT TRUE,
    ip_address INET,
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ended_at TIMESTAMPTZ
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
```

### Feature Flags

```sql
CREATE TABLE feature_flags (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'disabled',
    rollout_config JSONB NOT NULL DEFAULT '{}',
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

### Health & Alerts

```sql
CREATE TABLE tenant_health_scores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    overall_score REAL NOT NULL,
    scores JSONB NOT NULL DEFAULT '{}',
    signals JSONB NOT NULL DEFAULT '{}',
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE anomaly_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    alert_type TEXT NOT NULL,
    severity TEXT NOT NULL DEFAULT 'warning',
    description TEXT NOT NULL,
    details JSONB NOT NULL DEFAULT '{}',
    status TEXT NOT NULL DEFAULT 'open',
    detected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_health_tenant ON tenant_health_scores(tenant_id, computed_at DESC);
CREATE INDEX idx_alerts_tenant ON anomaly_alerts(tenant_id, status);
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID,
    actor_id UUID NOT NULL,
    impersonation_session_id UUID REFERENCES impersonation_sessions(id),
    action TEXT NOT NULL,
    resource_type TEXT NOT NULL,
    resource_id UUID,
    changes JSONB NOT NULL DEFAULT '{}',
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_actor ON audit_log(actor_id, created_at DESC);
```

### API Keys

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    name TEXT NOT NULL,
    key_hash TEXT NOT NULL UNIQUE,
    key_prefix TEXT NOT NULL,
    scopes TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Graph Queries

### Complete permission chain for a user

```sql
-- "What can user X do?" — traverse user → has_role → role → grants → permission
WITH RECURSIVE access_chain AS (
    SELECT gn.id AS node_id, gn.node_type, gn.label, 0 AS depth,
           ARRAY[gn.label] AS path
    FROM graph_nodes gn
    WHERE gn.node_type = 'user' AND gn.ref_id = 'user-uuid'

    UNION ALL

    SELECT gn2.id, gn2.node_type, gn2.label, ac.depth + 1,
           ac.path || gn2.label
    FROM access_chain ac
    JOIN graph_edges ge ON ge.source_id = ac.node_id
    JOIN graph_nodes gn2 ON gn2.id = ge.target_id
    WHERE ac.depth < 4
      AND ge.edge_type IN ('has_role', 'grants', 'on_resource', 'parent_of')
)
SELECT node_type, label, path
FROM access_chain
WHERE node_type IN ('permission', 'resource')
ORDER BY path;
```

### Impact analysis: "Who loses access if we delete this role?"

```sql
SELECT gn_user.label AS user_name, gn_user.ref_id AS user_id
FROM graph_nodes gn_role
JOIN graph_edges ge ON ge.target_id = gn_role.id AND ge.edge_type = 'has_role'
JOIN graph_nodes gn_user ON gn_user.id = ge.source_id
WHERE gn_role.node_type = 'role' AND gn_role.ref_id = 'role-uuid';
```

### Cross-tenant access map for SOC 2 audit

```sql
-- All users who can access tenants they don't belong to
SELECT DISTINCT
    gn_user.label AS user_name,
    u.email,
    gn_tenant.label AS accessible_tenant,
    ge.edge_type AS access_type,
    ge.properties AS access_details
FROM graph_nodes gn_user
JOIN users u ON u.id = gn_user.ref_id
JOIN graph_edges ge ON ge.source_id = gn_user.id
    AND ge.edge_type IN ('impersonates', 'manages', 'belongs_to')
JOIN graph_nodes gn_tenant ON gn_tenant.id = ge.target_id AND gn_tenant.node_type = 'tenant'
WHERE gn_user.node_type = 'user'
  AND (u.tenant_id IS NULL OR u.tenant_id != gn_tenant.ref_id)
ORDER BY gn_user.label, gn_tenant.label;
```

### Feature flag blast radius

```sql
-- Which tenants are affected by this feature flag?
SELECT gn_tenant.label AS tenant_name, gn_tenant.ref_id AS tenant_id,
       ge.properties->>'enabled_at' AS enabled_at
FROM graph_nodes gn_flag
JOIN graph_edges ge ON ge.target_id = gn_flag.id AND ge.edge_type = 'has_feature'
JOIN graph_nodes gn_tenant ON gn_tenant.id = ge.source_id AND gn_tenant.node_type = 'tenant'
WHERE gn_flag.node_type = 'feature_flag' AND gn_flag.ref_id = 'flag-uuid';
```

### Tenant relationship map (for visualization)

```sql
-- Build complete graph for a tenant (for D3/force-directed visualization)
SELECT
    gn_src.node_type AS source_type,
    gn_src.label AS source_label,
    ge.edge_type,
    gn_tgt.node_type AS target_type,
    gn_tgt.label AS target_label,
    ge.properties
FROM graph_edges ge
JOIN graph_nodes gn_src ON gn_src.id = ge.source_id
JOIN graph_nodes gn_tgt ON gn_tgt.id = ge.target_id
WHERE gn_src.ref_id = 'tenant-uuid' OR gn_tgt.ref_id = 'tenant-uuid'
ORDER BY ge.edge_type, gn_src.label;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Core | 2 | tenants, users |
| RBAC | 2 | roles, user_roles |
| SSO & Identity | 2 | sso_connections, scim_connections |
| Quotas | 2 | quota_definitions, tenant_quotas |
| Billing | 1 | billing_accounts |
| Impersonation | 2 | impersonation_sessions, impersonation_actions |
| Feature Flags | 2 | feature_flags, tenant_features |
| Health & Alerts | 2 | tenant_health_scores, anomaly_alerts |
| Infrastructure | 2 | audit_log, api_keys |
| **Total** | **19** | |

---

## Key Design Decisions

1. **Authorization as graph traversal** — The path `user → has_role → role → grants → permission → on_resource → resource` is a graph traversal. This directly implements the Zanzibar/OpenFGA authorization model, enabling both "can user X do Y?" checks and "show me everything user X can access" queries.

2. **Role hierarchy via `parent_of` edges** — `super_admin → parent_of → tenant_admin → parent_of → viewer` enables inherited permissions. When checking if a user can perform an action, the graph traversal follows `has_role` and `parent_of` edges to find all inherited permissions.

3. **Impersonation as graph edge** — Active impersonation sessions are `impersonates` edges from user nodes to tenant nodes. The edge `properties` carry session context. Querying "who is currently impersonating any tenant?" is a single edge-type filter.

4. **Cross-tenant access visualization** — The graph query for SOC 2 auditors shows all users who can access tenants they don't belong to, with the access type (impersonation, management, direct membership). This is the key audit artifact for access reviews.

5. **Feature flag blast radius** — `has_feature` edges from tenants to feature flag nodes enable "which tenants are affected?" queries. Before toggling a flag, the team can see exactly which tenants are in scope.

6. **Dual access patterns** — CRUD operations (creating a tenant, updating a user) use relational tables. Authorization checks and impact analysis use the graph. The application layer maintains consistency by writing to both when changes occur.

7. **Edge properties for FGA conditions** — `grants` edges carry `constraints` in their properties JSONB (e.g., `{"read_only": true}`, `{"ip_whitelist": ["10.0.0.0/8"]}`). This enables conditional authorization beyond flat RBAC.

8. **Tenant relationship visualization** — The complete graph for a tenant (users, roles, SSO, quotas, features, impersonation sessions) can be rendered as a force-directed diagram, giving admins a visual map of the tenant's access and configuration state.
