# Multi-Tenant Admin Dashboard — Phased Development Plan

> Project: 307-multi-tenant-admin-dashboard · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four
`data-model-suggestion-*.md` files. The persisted schema follows **Data Model Suggestion 1
(Entity-Centric Normalized Relational)** because RBAC precision, quota accuracy, and SOC 2 audit
completeness are non-negotiable for this product. Where tenant configuration varies widely
(custom metadata, per-tenant SSO config), the design borrows the JSONB-hybrid pattern from
Suggestion 3. The audit log is designed to be exportable as a CloudEvents stream (Suggestion 2)
without making the whole system event-sourced.

---

## Product Summary

**What it does.** A super-admin console for SaaS operators that unifies, in one pane of glass,
the concerns currently scattered across WorkOS (identity), Stripe (billing), and Qrvey
(analytics): tenant management, RBAC, per-tenant SSO/SCIM, quota and entitlement enforcement,
billing mirroring, safe audited impersonation, and AI-native operational intelligence (health
scoring, anomaly detection, natural-language tenant search, impersonation briefs).

**Who uses it.** Founding engineers at Series A/B SaaS companies, support escalation teams
(safe impersonation), RevOps teams (quota/billing per tenant), and infrastructure leads
(per-tenant resource limits).

**Key differentiators.** (1) One dashboard spanning identity + billing + quota + health;
(2) AI-native tenant health, anomaly detection, NL search, and impersonation briefs that no
incumbent offers; (3) turnkey Docker Compose self-hosting versus the Kubernetes complexity of
Keycloak/Ory; (4) an open, OpenAPI-3.1-documented super-admin API surface that doubles as an
MCP tool surface.

**Deployment model.** Self-hosted via Docker Compose (primary) and managed cloud. PostgreSQL +
Redis are the only hard infrastructure dependencies. Stripe is an external integration; LLM
calls go through a pluggable provider abstraction.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node 22 LTS) | The competitive surface (Logto, Frontegg) is TypeScript-native; the product is API + web-dashboard heavy, not ML-heavy. One language across API, workers, and frontend reduces context-switching and enables shared types. |
| Runtime / monorepo | pnpm workspaces + Turborepo | Shared `@app/db`, `@app/types`, `@app/sdk` packages consumed by API, workers, and web. Turborepo caches builds/tests across the monorepo. |
| API framework | Fastify + `@fastify/swagger` (Zod via `fastify-type-provider-zod`) | Fastify is fast, has first-class JSON Schema, and auto-emits **OpenAPI 3.1** (standards.md requirement) directly from Zod route schemas — no hand-written spec drift. |
| Validation / schema | Zod | Single source of truth for request/response types, OpenAPI generation, and DB-boundary parsing. JSON Schema (Draft 2020-12) is derived from Zod. |
| Database | PostgreSQL 16 | Required for **Row-Level Security** (standards.md: tenant isolation), partitioned `audit_log`, JSONB metadata, and `pgvector` for NL-search embeddings. |
| ORM / query layer | Drizzle ORM | Type-safe, thin, SQL-first (lets us hand-write RLS policies and partitioning that heavyweight ORMs fight). Generates migrations. |
| Migrations | drizzle-kit + hand-authored SQL for RLS/partitions | Drizzle handles table DDL; RLS policies, partition setup, and seed data are explicit SQL migrations. |
| Cache / queue | Redis 7 + BullMQ | Async workloads: Stripe/SCIM webhook processing, quota aggregation, health-score computation, anomaly scans, email. Redis also backs rate limiting and session/impersonation token storage. |
| AuthN (operators) | OIDC/OAuth 2.1 with mandatory PKCE; sessions as signed JWT (RFC 9068 profile) | Operators log into the dashboard via OIDC. Super-admin actions require **AAL2 / MFA** (NIST SP 800-63-4). |
| AuthN (tenant SSO) | `openid-client` (OIDC) + `@node-saml/node-saml` (SAML 2.0) | Per-tenant IdP config (standards.md: OIDC Core 1.0, SAML 2.0). Each tenant brings its own IdP. |
| Impersonation tokens | OAuth 2.0 Token Exchange (RFC 8693) with `act` claim | Super-admin acquires a tenant-scoped, read-only-by-default token carrying the actor (`act`) claim. |
| SCIM | Custom SCIM 2.0 server (RFC 7643/7644) | Inbound provisioning endpoint per tenant; no maintained turnkey Node SCIM server covers multi-tenant routing. |
| Billing | Stripe Node SDK + signed webhooks | Stripe is the source of truth; we mirror subscription/quota state and store raw events idempotently. |
| LLM provider | Provider abstraction over Anthropic + OpenAI (env-selectable); embeddings via provider | Health scoring, anomaly explanation, NL→SQL, impersonation briefs. Abstraction avoids lock-in and enables a deterministic mock in tests. |
| NL search safety | LLM emits a constrained Drizzle query AST against a whitelisted view, never raw SQL | Prevents injection / cross-tenant leakage from the NL interface (OWASP Multi-Tenant + Authorization cheat sheets). |
| Frontend | Next.js 16 (App Router) + React 19 + shadcn/ui + TanStack Query | Server Components for the tenant list/detail dashboard; consumes the generated TypeScript SDK. |
| SDK generation | `openapi-typescript` + `openapi-fetch` from the emitted OAS 3.1 | The web app and external consumers share one generated client; dogfoods the public API. |
| Auth/session crypto | `jose` (JWT), `argon2` (API key + SCIM token hashing) | Standards-aligned token handling and tamper-resistant secret storage. |
| Secrets | Pluggable `SecretStore` (env-backed default, Vault adapter) | `sso_connections.client_secret_vault_ref` indirection per Suggestion 1. |
| Testing | Vitest (unit/integration) + Testcontainers (real Postgres/Redis) + Playwright (E2E) | Fast unit layer; real-dependency integration via ephemeral containers; browser E2E for the console. |
| Code quality | Biome (lint+format) + `tsc --noEmit` (strict) | One fast tool for lint/format; strict TypeScript across the monorepo. |
| Containerisation | Docker + Docker Compose | Turnkey self-host (README requirement): `postgres`, `redis`, `api`, `worker`, `web` services. |
| CI | GitHub Actions | Lint, typecheck, unit, integration (Testcontainers), build, Docker image publish. |
| Observability | `pino` structured logs + OpenTelemetry traces | Operator-facing tooling must itself be debuggable and SOC 2 auditable. |

### Project Structure

```
multi-tenant-admin-dashboard/
├── package.json                 # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── biome.json
├── tsconfig.base.json
├── docker-compose.yml           # postgres, redis, api, worker, web
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── types/                   # shared Zod schemas + inferred TS types (tenant, user, role, quota, audit, events)
│   ├── db/
│   │   ├── src/
│   │   │   ├── schema/          # Drizzle table defs (tenants, users, rbac, sso, quotas, billing, impersonation, audit, ai)
│   │   │   ├── client.ts        # pool + withTenant() RLS context helper
│   │   │   └── seed.ts          # system roles, permissions, quota_definitions
│   │   ├── migrations/          # drizzle-kit SQL + hand-authored RLS/partition/seed migrations
│   │   └── drizzle.config.ts
│   ├── core/                    # framework-agnostic business logic (services), no HTTP
│   │   └── src/
│   │       ├── tenants/  rbac/  sso/  scim/  quotas/  billing/
│   │       ├── impersonation/  audit/  ai/  events/  secrets/
│   ├── llm/                     # provider abstraction + prompts + deterministic mock
│   └── sdk/                     # generated openapi-typescript client (+ thin wrapper)
├── apps/
│   ├── api/                     # Fastify server
│   │   └── src/
│   │       ├── server.ts  app.ts  openapi.ts
│   │       ├── plugins/         # auth, rls-context, rate-limit, error-handler, audit-hook
│   │       ├── routes/          # tenants, users, roles, sso, scim, quotas, billing, impersonation, audit, ai, health
│   │       └── webhooks/        # stripe, scim
│   ├── worker/                  # BullMQ processors
│   │   └── src/queues/          # webhook-ingest, quota-aggregate, health-score, anomaly-scan, email
│   └── web/                     # Next.js 16 dashboard
│       └── src/app/             # (auth), tenants/, tenants/[id]/, audit/, quotas/, impersonate/, search/
├── tests/
│   ├── integration/             # Vitest + Testcontainers
│   ├── e2e/                     # Playwright
│   └── fixtures/                # stripe events, SCIM payloads, SAML metadata, sample tenants
└── docs/
    └── openapi.json             # emitted artefact (checked in by CI)
```

---

## Phase 1: Foundation — Monorepo, Database, RLS, Migrations

### Purpose
Establish the monorepo, the PostgreSQL schema with tenant-isolation Row-Level Security, the
migration/seed pipeline, and the `withTenant()` connection helper that every later phase relies
on. After this phase the database can be brought up via Docker Compose, migrated, seeded with
system roles/permissions/quota definitions, and RLS provably isolates tenant rows.

### Tasks

#### 1.1 — Monorepo & tooling skeleton
**What**: pnpm + Turborepo workspace with Biome, strict TypeScript, and the package/app directories above.

**Design**:
- `pnpm-workspace.yaml` includes `packages/*` and `apps/*`.
- `tsconfig.base.json`: `"strict": true`, `"noUncheckedIndexedAccess": true`, `"moduleResolution": "bundler"`, path aliases `@app/*`.
- `turbo.json` pipelines: `build` (depends on `^build`), `lint`, `typecheck`, `test`, `test:integration`.
- `.env.example` documents: `DATABASE_URL`, `REDIS_URL`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `JWT_SIGNING_KEY` (or JWKS path), `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `LLM_PROVIDER`, `LLM_API_KEY`, `SECRET_STORE` (`env`|`vault`), `APP_BASE_URL`.

**Testing**:
- `Unit: pnpm -r typecheck → exits 0 on empty skeleton`.
- `Unit: biome check → 0 errors on committed files`.
- `CI: turbo build with no packages → succeeds (smoke)`.

#### 1.2 — Database schema (`@app/db`)
**What**: Drizzle table definitions for all 24 tables of Suggestion 1, plus a JSONB `config` column on `sso_connections` and `settings JSONB` on `tenants` (Suggestion 3 hybrid).

**Design**: Implement the DDL from Suggestion 1 verbatim for: `tenants`, `tenant_metadata`,
`users`, `roles`, `permissions`, `role_permissions`, `user_roles`, `sso_connections`,
`scim_connections`, `scim_sync_logs`, `quota_definitions`, `tenant_quotas`, `quota_usage`,
`quota_alerts`, `billing_accounts`, `billing_events`, `impersonation_sessions`,
`impersonation_actions`, `feature_flags`, `tenant_features`, `tenant_health_scores`,
`anomaly_alerts`, `audit_log` (range-partitioned by `created_at`), `api_keys`. Key shared status
enums (TypeScript union types exported from `@app/types`):

```ts
export type TenantStatus = 'active' | 'suspended' | 'deactivated' | 'pending_setup';
export type UserStatus   = 'active' | 'invited' | 'suspended' | 'deactivated';
export type QuotaEnforcement = 'hard' | 'soft' | 'advisory';
export type ImpersonationStatus = 'active' | 'ended' | 'expired';
export type AnomalyStatus = 'open' | 'acknowledged' | 'resolved' | 'dismissed';
```

**Testing**:
- `Integration (Testcontainers): apply all migrations → all 24 tables + indexes exist (query information_schema)`.
- `Integration: audit_log is partitioned by created_at → inserting rows in two months lands in two partitions`.
- `Unit: Drizzle schema types compile and match @app/types unions`.

#### 1.3 — RLS policies & `withTenant()` helper
**What**: Hand-authored SQL migration enabling RLS on tenant-scoped tables, plus a DB-client wrapper that sets `app.current_tenant_id` per transaction.

**Design**:
- Enable RLS + `tenant_isolation` policy on `users`, `tenant_quotas`, `quota_usage`, `tenant_metadata`, `sso_connections`, `scim_connections`, `billing_accounts`, `impersonation_sessions`, `tenant_features`, `anomaly_alerts`, `tenant_health_scores`. Policy: `tenant_id = current_setting('app.current_tenant_id', true)::UUID OR tenant_id IS NULL` (NULL allows platform rows).
- App connects as a **non-superuser, non-BYPASSRLS** role (`app_user`).
- Helper:
```ts
// runs cb inside a transaction with SET LOCAL app.current_tenant_id
async function withTenant<T>(tenantId: string | null, cb: (tx: Tx) => Promise<T>): Promise<T>
// platform/super-admin context: withPlatform(cb) sets the GUC to '' so only IS NULL rows + explicit cross-tenant queries via a separate elevated role
```
- A separate elevated path (`asSuperAdmin`) uses a role with `BYPASSRLS` for legitimate cross-tenant listing, gated by RBAC at the service layer.

**Testing**:
- `Integration: insert users for tenant A and B; withTenant(A) SELECT → only A's rows`.
- `Integration: withTenant(A) attempt to UPDATE a tenant B row → 0 rows affected`.
- `Integration: asSuperAdmin SELECT across tenants → sees both (BYPASSRLS path)`.
- `Integration: missing GUC (app.current_tenant_id unset) → tenant-scoped SELECT returns 0 rows (fails closed)`.

#### 1.4 — Seed data & migration CLI
**What**: Seed system roles (`super_admin`, `support_agent`, `viewer`), the `permissions` matrix, default `quota_definitions`, and a `db:migrate`/`db:seed` CLI.

**Design**: Seed exactly the rows from Suggestion 1's `INSERT` blocks. Permission matrix:
resources `{tenants,users,roles,quotas,billing,audit_logs,impersonation}` × actions
`{create,read,update,delete,impersonate,export}`. `super_admin` gets all; `support_agent` gets
read + impersonate; `viewer` gets read. CLI commands: `pnpm db:migrate`, `pnpm db:seed`,
`pnpm db:reset`.

**Testing**:
- `Integration: db:seed twice → idempotent (ON CONFLICT DO NOTHING), no duplicate roles`.
- `Unit: permission matrix generator → 7×6 expected rows, super_admin role_permissions count == permissions count`.

---

## Phase 2: API Skeleton, Operator Auth & MFA

### Purpose
Stand up the Fastify server with OpenAPI 3.1 emission, OIDC operator login, JWT sessions,
the RLS-context request plugin, RBAC enforcement, and the AAL2/MFA gate for super-admin
operations. After this phase the API authenticates operators, enforces roles, and every route
runs in the correct tenant/RLS context.

### Tasks

#### 2.1 — Fastify app + OpenAPI 3.1 emission
**What**: Bootstrapped Fastify server with Zod type provider, Swagger UI, and a CI step writing `docs/openapi.json`.

**Design**:
- `app.ts` registers `fastify-type-provider-zod`, `@fastify/swagger` (`openapi: { openapi: '3.1.0' }`), `@fastify/swagger-ui` at `/docs`, `@fastify/helmet`, `@fastify/cors`.
- Global error handler maps domain errors → RFC 9457 `application/problem+json`.
- `GET /healthz` (liveness), `GET /readyz` (checks Postgres + Redis).
- `pnpm openapi:dump` boots the app and writes the spec.

**Testing**:
- `Integration: GET /healthz → 200 {status:'ok'}`.
- `Integration: GET /docs/json → valid OpenAPI 3.1 (validate with @apidevtools/swagger-parser)`.
- `Unit: error handler maps NotFoundError → 404 problem+json with type/title/detail`.

#### 2.2 — OIDC operator login & JWT sessions
**What**: Operator authentication via the platform OIDC issuer, issuing signed session JWTs.

**Design**:
- Routes: `GET /auth/login` (PKCE Authorization Code redirect), `GET /auth/callback` (exchange code → upsert platform `user` with `tenant_id NULL` → issue session JWT), `POST /auth/logout`.
- Session JWT (RFC 9068 profile) claims: `sub` (user id), `roles: string[]`, `amr` (auth methods, includes `mfa` when satisfied), `aal` (`1`|`2`), `exp` (1h), refresh via short rotation.
- `authPlugin` decorates `request.actor = { userId, roles, aal, amr }`; rejects missing/expired tokens with 401.

**Testing**:
- `Integration (mocked IdP): callback with valid code → 200, Set-Cookie session, user upserted`.
- `Integration: request to protected route without token → 401`.
- `Integration: expired JWT → 401; tampered signature → 401`.

#### 2.3 — RBAC enforcement & RLS-context plugin
**What**: A `requirePermission(resource, action)` guard and a plugin that establishes RLS context from the route's tenant scope.

**Design**:
- `requirePermission('tenants','update')` preHandler: loads effective permissions for `request.actor.roles` (cached in Redis 60s), 403 on miss.
- `rlsContextPlugin`: for routes under `/tenants/:tenantId/*`, runs the handler's DB work inside `withTenant(tenantId)`; platform routes use `asSuperAdmin` only when the actor holds the platform `super_admin` role.
- IDOR defence (OWASP): tenant-scoped routes never trust a `tenant_id` from the body — only the path param, validated against actor scope.

**Testing**:
- `Integration: viewer calls DELETE /tenants/:id → 403`.
- `Integration: super_admin calls same → passes guard`.
- `Integration: support_agent reads /tenants/:id/users → only that tenant's users (RLS verified)`.
- `Integration: body containing foreign tenant_id is ignored; path param wins`.

#### 2.4 — AAL2 / MFA gate
**What**: Enforce MFA for operations on roles whose `mfa_required = true` and for all impersonation.

**Design**:
- `requireAAL2` preHandler: if `request.actor.aal < 2` → 403 `mfa_required` problem with a `step_up_url`.
- TOTP enrolment/verify endpoints (`POST /auth/mfa/enroll`, `POST /auth/mfa/verify`) using `otplib`; on success the next session token carries `aal:2, amr:['pwd','otp']`.

**Testing**:
- `Integration: super_admin without MFA hits POST /tenants/:id/impersonation → 403 mfa_required`.
- `Integration: after /auth/mfa/verify → token aal=2 → same call passes the gate`.
- `Unit: TOTP verify rejects codes outside the time window`.

---

## Phase 3: Tenant & User Management (Core Value, Part 1)

### Purpose
Deliver the heart of the super-admin console: tenant CRUD, lifecycle (activate/suspend/
deactivate), search, custom metadata, and tenant-scoped user management. This is the first
externally valuable increment — operators can manage the whole tenant estate.

### Tasks

#### 3.1 — Tenant CRUD, lifecycle & metadata
**What**: REST endpoints for tenant create/read/update, status transitions, and custom metadata.

**Design**:
- Endpoints (all OpenAPI-documented, Zod-validated):
  - `POST /tenants` → create (`name`, `slug`, `plan`, `data_residency`).
  - `GET /tenants` → paginated list (cursor pagination, `?status=&plan=&q=`).
  - `GET /tenants/:id` → detail (joins user_count, billing status, latest health score).
  - `PATCH /tenants/:id` → update mutable fields + `settings` JSONB.
  - `POST /tenants/:id/status` → `{ status, reason }`; validated state machine: `pending_setup → active → suspended ↔ active → deactivated`.
  - `PUT /tenants/:id/metadata/:key`, `DELETE /tenants/:id/metadata/:key`.
- `TenantService` in `@app/core` is HTTP-agnostic; every mutating method emits an audit entry (Phase 6 hook) and returns the updated read model.

```ts
interface TenantService {
  create(input: CreateTenant, actor: Actor): Promise<Tenant>;
  list(filter: TenantFilter): Promise<Page<TenantSummary>>;
  get(id: string): Promise<TenantDetail>;
  setStatus(id: string, status: TenantStatus, reason: string, actor: Actor): Promise<Tenant>;
}
```

**Testing**:
- `Unit: setStatus from deactivated → active → throws InvalidTransition`.
- `Integration: POST /tenants duplicate slug → 409`.
- `Integration: GET /tenants?status=suspended → only suspended tenants`.
- `Integration: PATCH unknown id → 404 problem+json`.

#### 3.2 — Tenant-scoped user management
**What**: CRUD + lifecycle for users within a tenant, plus platform super-admin user listing.

**Design**:
- `POST /tenants/:id/users` (invite: status `invited`), `GET /tenants/:id/users`, `PATCH /tenants/:id/users/:userId` (status, name), with RLS enforced via `rlsContextPlugin`.
- Unique email per tenant (partial unique index from Suggestion 1).
- `last_login_at` updated by the auth layer (Phase 2 callback / tenant SSO Phase 5).

**Testing**:
- `Integration: invite user with email already in tenant → 409`.
- `Integration: same email in two different tenants → both succeed (composite uniqueness)`.
- `Integration: list users for tenant A as support_agent → cannot see tenant B users`.

#### 3.3 — Bulk operations
**What**: Bulk activate/deactivate tenants and bulk user export (CSV).

**Design**:
- `POST /tenants/bulk/status` `{ tenant_ids: string[], status, reason }` → per-item result array `{ id, ok, error? }` (partial success, 207-style payload).
- `POST /tenants/:id/users/export` → enqueues a job; `GET /exports/:jobId` returns CSV when ready (RFC 4180). Export is itself an audited, GDPR-relevant action.

**Testing**:
- `Integration: bulk status with one invalid id → other ids succeed, invalid reported in results`.
- `Integration (mocked queue): export enqueues job, status endpoint reflects pending→done`.
- `Unit: CSV serialiser escapes commas/quotes/newlines per RFC 4180`.

---

## Phase 4: RBAC, Quotas & Entitlements (Core Value, Part 2)

### Purpose
Add the authorization and resource-governance core: custom roles/permissions management,
per-tenant quota allocation, usage tracking, enforcement (hard/soft/advisory), and feature
flags/entitlements. After this phase operators can govern what tenants are entitled to and what
they have consumed.

### Tasks

#### 4.1 — Roles & permissions management
**What**: CRUD for tenant-scoped and platform roles, permission assignment, and user-role grants.

**Design**:
- `POST /tenants/:id/roles`, `PATCH /roles/:roleId`, `DELETE /roles/:roleId` (blocked when `is_system`).
- `PUT /roles/:roleId/permissions` (set permission ids), `POST /users/:userId/roles` (grant, records `granted_by`), `DELETE /users/:userId/roles/:roleId`.
- `canAct(actor, resource, action): boolean` — the single authorization predicate used by `requirePermission`, computed from `user_roles → role_permissions → permissions`, cached.

**Testing**:
- `Integration: DELETE a system role → 409`.
- `Integration: grant role then canAct(user,'quotas','update') → true; revoke → false`.
- `Integration: role created in tenant A invisible to tenant B (RLS)`.

#### 4.2 — Quota definitions, allocation & usage
**What**: Manage quota definitions, per-tenant allocations/overrides, and period-based usage snapshots.

**Design**:
- `GET /quota-definitions`, `POST /quota-definitions` (platform-only).
- `PUT /tenants/:id/quotas/:quotaSlug` `{ allocated_limit, override_reason }`.
- `GET /tenants/:id/quotas` → per-quota `{ allocated_limit, current_usage, usage_pct, period }`.
- `POST /tenants/:id/usage` (internal/SDK ingestion) increments `quota_usage` for the current period; worker aggregates raw events into `quota_usage` rows.

```ts
interface QuotaService {
  allocate(tenantId: string, slug: string, limit: number, reason: string, actor: Actor): Promise<TenantQuota>;
  record(tenantId: string, slug: string, delta: number): Promise<QuotaUsage>;
  check(tenantId: string, slug: string): Promise<{ allowed: boolean; usage: number; limit: number; enforcement: QuotaEnforcement }>;
}
```

**Testing**:
- `Integration: allocate then record beyond limit → check() returns allowed:false for 'hard'`.
- `Integration: 'soft' enforcement over limit → allowed:true but breach flagged`.
- `Integration: usage rolls into correct period_start window`.
- `Unit: override without reason → ValidationError`.

#### 4.3 — Quota enforcement middleware & alerts
**What**: An enforcement helper plus threshold alert generation (80/90/100%).

**Design**:
- `enforceQuota(slug)` preHandler usable by downstream products via the SDK: `hard` → 429 `quota_exceeded` with `Retry-After`; `soft`/`advisory` → allow, emit alert/log.
- Worker job evaluates usage after each aggregation; inserts `quota_alerts` rows crossing thresholds (dedup per period); unacknowledged alerts surface on tenant detail.

**Testing**:
- `Integration: cross 80% → one quota_alert at 80 created; re-cross same period → no duplicate`.
- `Integration (mocked): hard quota at 100% → 429 with Retry-After header`.
- `Integration: acknowledge alert → disappears from open-alerts query`.

#### 4.4 — Feature flags & entitlements
**What**: Global feature flags with per-tenant enablement and percentage rollout.

**Design**:
- `POST /feature-flags`, `PUT /tenants/:id/features/:flagSlug` `{ enabled }`.
- `isEnabled(tenantId, slug)`: explicit `tenant_features` row wins; else `percentage_rollout` via stable hash of `tenantId` vs `rollout_percentage`; else `enabled_globally`/`disabled`.

**Testing**:
- `Unit: percentage rollout is deterministic per tenant id and monotonic as % increases`.
- `Integration: explicit tenant override beats global setting`.

---

## Phase 5: Per-Tenant SSO, SCIM & Billing Integration

### Purpose
Connect the platform to external identity and billing systems: per-tenant OIDC/SAML SSO, a SCIM
2.0 provisioning server, and Stripe webhook ingestion mirroring subscription/quota state. These
are integration phases that depend on the core (Phases 3–4) but are independent of each other
and can be built in parallel.

### Tasks

#### 5.1 — Per-tenant SSO configuration & login (OIDC + SAML)
**What**: Configure per-tenant IdPs and authenticate tenant users through them.

**Design**:
- `POST /tenants/:id/sso` stores an `sso_connections` row; `client_secret` written via `SecretStore`, only `client_secret_vault_ref` persisted.
- `config JSONB` holds provider-specific fields (Suggestion 3 hybrid): OIDC `{ issuer_url, client_id, scopes }`; SAML `{ metadata_url, certificate }`.
- `POST /tenants/:id/sso/:connId/test` validates discovery/metadata reachability → sets status `active|error`.
- Tenant login: `GET /sso/:tenantSlug/login` (OIDC PKCE or SAML AuthnRequest) → callback upserts the tenant user, updates `last_login_at`. `domain_hint` enables email-domain auto-routing.
- OAuth 2.1: PKCE mandatory for OIDC flows.

**Testing**:
- `Integration (mocked OIDC discovery): create + test OIDC connection → status active`.
- `Integration: unreachable issuer → status error, descriptive message stored`.
- `Integration: secret never returned in GET /sso responses (only vault_ref)`.
- `Unit (fixture): SAML metadata XML parsed → cert + SSO URL extracted`.

#### 5.2 — SCIM 2.0 provisioning server
**What**: Inbound SCIM endpoints (RFC 7643/7644) for IdP-driven user provisioning per tenant.

**Design**:
- Routes under `/scim/v2/:tenantId/`: `POST/GET/PATCH/DELETE Users`, `GET Groups`, `GET ServiceProviderConfig`.
- Bearer auth against `scim_connections.bearer_token_hash` (argon2-verified); the `:tenantId` plus token jointly scope the connection.
- Each operation upserts `users` (mapping SCIM `userName`/`emails`/`active` → user fields), writes a `scim_sync_logs` row, and emits an audit/event entry.
- Responses conform to SCIM core schema (`schemas`, `id`, `meta`, list `Resources`/`totalResults`).

**Testing**:
- `Integration: POST /scim/v2/:tid/Users valid → user created, scim_sync_log success row`.
- `Integration: bad bearer token → 401, no user created`.
- `Integration: PATCH active:false → user.status deactivated`.
- `Integration: token for tenant A used on tenant B path → 401 (token scope mismatch)`.
- `Fixture: Okta-shaped SCIM payload deserialises into the user mapping`.

#### 5.3 — Stripe billing integration & webhooks
**What**: Mirror Stripe subscription/quota state and ingest webhooks idempotently.

**Design**:
- `POST /tenants/:id/billing/link` creates/attaches a Stripe customer → `billing_accounts` row.
- `POST /webhooks/stripe`: verify signature with `STRIPE_WEBHOOK_SECRET`; insert into `billing_events` keyed on `stripe_event_id` (unique → idempotent); enqueue `webhook-ingest` job.
- Worker projects events → `billing_accounts` (`plan`, `status`, `current_period_*`, `mrr_cents`) and triggers quota re-allocation on `customer.subscription.updated`.
- `GET /tenants/:id/billing` → current subscription snapshot.

**Testing**:
- `Integration: webhook with invalid signature → 400, nothing stored`.
- `Integration: same stripe_event_id delivered twice → one billing_events row (idempotent)`.
- `Integration (fixture): invoice.payment_failed → billing_accounts.status past_due`.
- `Integration: subscription.updated to Pro → tenant quotas re-allocated to Pro defaults`.

---

## Phase 6: Audit Logging & Safe Impersonation

### Purpose
Deliver the SOC 2 / ISO 27001 backbone: a tamper-evident, queryable audit log capturing every
administrative action, and safe impersonation with full session and action tracking. Audit is
retro-wired into the services from Phases 3–5 via a hook so no action goes unlogged.

### Tasks

#### 6.1 — Audit logging subsystem
**What**: Centralised, append-only, hash-chained audit log with a query/export API.

**Design**:
- `AuditService.record(entry)` writes `audit_log` rows; each row stores `prev_hash` and `row_hash = sha256(prev_hash + canonical(entry))` for tamper-evidence (per-tenant chain + platform chain).
- A Fastify `onResponse` hook + service-layer calls capture: actor, `actor_type`, `impersonation_session_id` (if active), action, resource, `changes` (before/after JSONB), `ip_address`, `user_agent`.
- `GET /audit` (platform) and `GET /tenants/:id/audit` (tenant-scoped) — filter by actor, action, resource, date range; cursor-paginated over the partitioned table.
- `GET /audit/export?format=cloudevents` emits the stream as CloudEvents 1.0 JSON (Suggestion 2 alignment) for SIEM forwarding. Retention ≥ 12 months (partition drop policy documented).

```ts
interface AuditEntry {
  tenantId: string | null;
  actorId: string; actorType: 'user'|'system'|'scim'|'api_key';
  impersonationSessionId?: string;
  action: string; resourceType: string; resourceId?: string;
  changes: Record<string, { from: unknown; to: unknown }>;
  ipAddress?: string; userAgent?: string;
}
```

**Testing**:
- `Integration: tenant status change writes an audit row with before/after changes`.
- `Integration: hash chain verifies; mutating any row breaks verification of subsequent rows`.
- `Integration: GET /tenants/:id/audit as support_agent → only that tenant's entries`.
- `Integration: export?format=cloudevents → valid CloudEvents envelopes (type/source/time/data)`.

#### 6.2 — Safe impersonation (RFC 8693 token exchange)
**What**: Start/end impersonation sessions issuing tenant-scoped, read-only-by-default tokens with full action logging.

**Design**:
- `POST /tenants/:id/impersonation` (requires AAL2, `impersonation` permission, `{ reason, target_user_id?, read_only=true }`) → creates `impersonation_sessions`, issues an RFC 8693-style token: `sub=target`, `act={ sub: actorId }`, `org_id=tenantId`, `scope` limited to read when `read_only`, short TTL (15m). Emits `impersonation.started`.
- Every request made with an impersonation token is recorded in `impersonation_actions` and linked from `audit_log.impersonation_session_id`. Write attempts under `read_only` → 403.
- `POST /impersonation/:sessionId/end` → status `ended`; idle/TTL expiry → `expired` via worker.

**Testing**:
- `Integration: start impersonation without AAL2 → 403 mfa_required`.
- `Integration: read_only token attempts a write → 403; action still recorded`.
- `Integration: actions during session appear in impersonation_actions and reference session in audit_log`.
- `Integration: end session → token rejected afterwards (401)`.

---

## Phase 7: Web Dashboard (Operator Console)

### Purpose
Build the operator-facing Next.js console over the generated SDK: tenant list/search/detail,
user and role management, quota and billing views, audit viewer, and impersonation controls.
After this phase the product is usable end-to-end without curling the API.

### Tasks

#### 7.1 — SDK generation, auth shell & app layout
**What**: Generate the typed client, wire operator OIDC login, and build the dashboard shell.

**Design**:
- `pnpm sdk:generate` runs `openapi-typescript` over `docs/openapi.json` into `packages/sdk`.
- Next.js middleware guards routes; login redirects to the API `/auth/login`; session cookie forwarded to API calls (server actions / route handlers).
- shadcn/ui app shell: sidebar (Tenants, Quotas, Audit, Search), top bar with operator identity + MFA/step-up indicator.

**Testing**:
- `E2E (Playwright, mocked IdP): unauthenticated visit → redirected to login; after login → tenants page`.
- `Unit: SDK type for GET /tenants matches Zod response schema (compile check)`.

#### 7.2 — Tenant list, detail & user/role management UI
**What**: The primary console screens.

**Design**:
- `/tenants`: server-rendered table with search/status/plan filters, cursor pagination, bulk-select → bulk status action.
- `/tenants/[id]`: tabs — Overview (health score, MRR, quota gauges, open alerts), Users (invite/suspend, role assignment), Roles, SSO/SCIM status, Billing, Audit (scoped).
- TanStack Query for client mutations with optimistic updates and toast feedback.

**Testing**:
- `E2E: search "acme" → filtered list; open detail → tabs render`.
- `E2E: suspend a tenant via UI → status badge updates, audit tab shows the entry`.
- `E2E: assign a role to a user → reflected on reload`.

#### 7.3 — Quota, billing & audit views; impersonation launch
**What**: Quota gauges/alerts, billing snapshot, audit viewer with filters, and an impersonation launcher.

**Design**:
- Quota tab: per-quota progress bars (usage_pct), alert acknowledgement.
- Audit viewer: filter bar (actor/action/resource/date), virtualised list, CloudEvents export button.
- Impersonate button → modal requiring a reason + MFA step-up if needed → opens impersonation session, shows a persistent "Impersonating <tenant> (read-only)" banner.

**Testing**:
- `E2E: acknowledge a quota alert → removed from open list`.
- `E2E: start impersonation without MFA → step-up modal; with MFA → banner appears`.
- `E2E: audit filter by action="tenant.suspended" → only matching rows`.

---

## Phase 8: AI-Native Operational Intelligence

### Purpose
Deliver the differentiating AI layer: composite tenant health scoring, anomaly detection,
natural-language tenant search (safely constrained), impersonation context briefs, and
auto-remediation suggestions. This is the product's moat and depends on data from all prior
phases.

### Tasks

#### 8.1 — LLM provider abstraction
**What**: A pluggable LLM/embedding interface with a deterministic mock for tests.

**Design**:
```ts
interface LlmProvider {
  complete(req: { system: string; user: string; json?: ZodSchema }): Promise<unknown>;
  embed(texts: string[]): Promise<number[][]>;
}
```
Env `LLM_PROVIDER` selects `anthropic`|`openai`|`mock`. All prompts live in `packages/llm/prompts`
with versioned templates. JSON outputs validated against Zod before use.

**Testing**:
- `Unit: mock provider returns canned structured output → schema-validated`.
- `Unit: malformed LLM JSON → ValidationError, no crash`.

#### 8.2 — Tenant health scoring
**What**: Periodic composite health score per tenant written to `tenant_health_scores`.

**Design**:
- Worker job gathers signals: `active_users_30d`, API-call trend (from `quota_usage`), open `anomaly_alerts`, `days_since_last_login`, `mrr_cents`, payment failures.
- Deterministic weighted sub-scores (billing/adoption/support/engagement) computed in code; the LLM produces a short natural-language **explanation** only (not the number) → reproducible scores. `signals` JSONB preserved for explainability.
- `GET /tenants/:id/health` → latest score + sub-scores + explanation.

**Testing**:
- `Unit: scoring is deterministic for fixed signals (no LLM in the number)`.
- `Integration: declining usage + payment failure → lower billing/engagement sub-scores`.
- `Integration (mock LLM): explanation generated and stored alongside score`.

#### 8.3 — Anomaly detection
**What**: Scheduled scans flagging quota spikes, API spikes, permission changes, login anomalies, billing failures.

**Design**:
- Worker computes rolling baselines (mean/stddev over trailing window) per tenant per metric; deviations beyond z-threshold → `anomaly_alerts` (`severity` by magnitude). Permission/role changes and payment failures are event-triggered, not statistical.
- LLM generates the human-readable `description` and a suggested remediation; dedup prevents alert storms.

**Testing**:
- `Unit: z-score over fixture series → spike at index N flagged, steady series → none`.
- `Integration: sudden role escalation event → permission_change anomaly created`.
- `Integration: dedup → repeated spike in window does not create duplicate open alerts`.

#### 8.4 — Natural-language tenant search (safe)
**What**: "Show all Pro tenants inactive 30 days" → constrained, tenant-safe query.

**Design**:
- LLM maps NL → a **constrained query object** (whitelisted fields/operators on a read-only `tenant_search` view), never raw SQL. The object is validated by Zod, compiled to a parameterised Drizzle query, and executed via the super-admin read path.
- `POST /search/tenants` `{ query: string }` → `{ interpreted: QuerySpec, results: TenantSummary[] }` so operators see how their question was interpreted.

**Testing**:
- `Unit: NL → QuerySpec for "Pro tenants inactive 30 days" → { plan:'pro', last_login_before: now-30d }`.
- `Integration: QuerySpec compiles to parameterised SQL (no string interpolation)`.
- `Security: adversarial prompt ("drop table") → rejected by QuerySpec schema, no SQL executed`.

#### 8.5 — Impersonation context brief & remediation suggestions
**What**: Pre-impersonation AI brief and inline remediation suggestions for common issues.

**Design**:
- On impersonation start (or `GET /tenants/:id/brief`), assemble recent activity, open alerts, billing status, health → LLM produces a concise brief shown in the impersonation modal.
- Remediation: for known issue signatures (failed webhook delivery, stuck payment, inactive SSO), attach a suggested fix string to the relevant alert.

**Testing**:
- `Integration (mock LLM): brief includes open alert count and billing status from real data`.
- `Integration: payment_failed alert → remediation suggestion attached`.

---

## Phase 9: Hardening, Packaging & Self-Host Distribution

### Purpose
Make the platform production-grade and turnkey to self-host: rate limiting, API keys, GDPR data
controls, observability, Docker Compose distribution, and CI/CD. Delivers on the README's
"turnkey vs Kubernetes complexity" promise.

### Tasks

#### 9.1 — API keys, rate limiting & request hardening
**What**: Programmatic API-key auth, per-key rate limits, and standard security headers.

**Design**:
- `POST /api-keys` (scoped, prefix shown once; argon2 `key_hash` stored). Auth plugin accepts `Authorization: Bearer <key>` for machine clients, resolving scopes → permissions.
- `@fastify/rate-limit` backed by Redis, keyed by api-key/IP; `429` with `Retry-After`.
- Helmet, CORS allowlist, body-size limits, request-id propagation.

**Testing**:
- `Integration: revoked/inactive key → 401`.
- `Integration: exceed rate limit → 429 with Retry-After`.
- `Unit: key shown in full only on creation; subsequent GET shows prefix only`.

#### 9.2 — GDPR data controls
**What**: Per-tenant data export and right-to-erasure operations.

**Design**:
- `POST /tenants/:id/gdpr/export` → full tenant data bundle (JSON) via worker; audited.
- `POST /tenants/:id/gdpr/erase` → hard-delete/anonymise tenant + users + scoped rows in a transaction (cascades), recording the erasure in the platform audit chain (the action survives; PII does not). `data_residency` honoured in export labelling.

**Testing**:
- `Integration: export bundle contains tenant, users, quotas, billing, audit summary`.
- `Integration: erase → tenant rows gone; platform audit retains the erasure event`.

#### 9.3 — Observability
**What**: Structured logging and tracing across api/worker.

**Design**:
- `pino` JSON logs with request-id + actor + tenant context (PII-redacted).
- OpenTelemetry traces (HTTP, DB, queue, LLM spans) exported via OTLP env config.

**Testing**:
- `Unit: log redaction removes secrets/tokens/PII fields`.
- `Integration: a request emits a trace spanning route → service → DB`.

#### 9.4 — Docker Compose distribution & CI/CD
**What**: One-command self-host and a full CI pipeline.

**Design**:
- `docker-compose.yml`: `postgres`, `redis`, `api`, `worker`, `web` with healthchecks; `api` runs migrations+seed on first boot (guarded). `make up` / `docker compose up` brings up a working stack from `.env`.
- GitHub Actions: lint → typecheck → unit → integration (Testcontainers) → build → publish images → emit `docs/openapi.json` artefact.

**Testing**:
- `E2E (CI): docker compose up → /readyz green; create+list a tenant via SDK against the live stack`.
- `CI: openapi.json regenerated and matches committed spec (drift check fails the build)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (DB, RLS, migrations)        ─── required by everything
    │
Phase 2: API skeleton, operator auth, MFA        ─── requires Phase 1
    │
Phase 3: Tenant & User management (core pt.1)     ─── requires Phase 2
    │
Phase 4: RBAC, Quotas, Entitlements (core pt.2)   ─── requires Phase 3
    │
    ├── Phase 5: SSO / SCIM / Billing integrations ─── requires Phase 4
    │       (5.1 SSO, 5.2 SCIM, 5.3 Stripe are mutually parallel)
    │
    └── Phase 6: Audit & Impersonation             ─── requires Phase 4
                 (audit hook retro-wires into 3–5)
         │
Phase 7: Web Dashboard                            ─── requires Phases 3–6 (SDK from 2.1)
    │
Phase 8: AI Operational Intelligence              ─── requires Phases 4–6 (data signals)
    │
Phase 9: Hardening, GDPR, packaging, CI/CD        ─── requires all; finalises distribution
```

**Parallelism opportunities**
- After Phase 4: Phases 5 and 6 can be built concurrently by separate developers.
- Within Phase 5: the SSO (5.1), SCIM (5.2), and Stripe (5.3) tracks are independent.
- Phase 7 (web) and Phase 8 (AI) can proceed in parallel once Phase 6 lands, since the AI layer
  is API/worker-side and the web layer consumes finished endpoints incrementally.
- Phase 9 sub-tasks (API keys, GDPR, observability, packaging) are largely independent.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`turbo test test:integration`).
3. Biome lint/format passes with zero errors.
4. `tsc --noEmit` passes in strict mode across affected packages.
5. New/changed endpoints appear correctly in the auto-generated OpenAPI 3.1 spec, and the SDK regenerates without type errors.
6. Drizzle migrations created, applied cleanly to a fresh database, and idempotent on re-run; any RLS policy changes verified by an isolation test.
7. Every new mutating operation emits an audit log entry (from Phase 6 onward).
8. The feature works end-to-end against a real Postgres/Redis (Testcontainers), and operator-visible features have a passing Playwright path (Phase 7+).
9. New configuration/env vars documented in `.env.example` and the README.
10. Docker build succeeds for affected services; `docker compose up` reaches `/readyz` green (from Phase 9, enforced in CI).
```
