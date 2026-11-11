# 06 — Row-Level Security (RLS), Roles & Gateway Bootstrap Functions

**Scope:** Database-layer multi-tenant isolation across all tenant-owned tables  
**Blueprint Alignment:** Sections 3, 6, 7, 12, 13, 17 (Acceptance Criteria #8)

---

## Why Application `WHERE tenant_id = ?` Is Not Enough

In a shared multi-tenant PostgreSQL database, relying solely on developers remembering `WHERE tenant_id = ...` in every ORM/SQL query is fragile. A single missing filter in a join or analytics endpoint leaks cross-tenant conversations or configurations.

RelayAI enforces **defense-in-depth**:
1. **Separate DB Roles:**
   - `relayai_migrator`: Owns tables and executes DDL migrations only.
   - `relayai_app`: Runtime connection pool role (`NOSUPERUSER NOBYPASSRLS`). Because `FORCE ROW LEVEL SECURITY` is enabled on every tenant table, even table owners cannot skip RLS unless explicitly running as `BYPASSRLS`.
2. **Transaction-Local Tenant Context:**
   Every request sets **transaction-scoped** PostgreSQL variables (`SET LOCAL`, automatically cleared at `COMMIT`/`ROLLBACK`—safe with PgBouncer / connection pooling!):
   - `app.current_tenant_id`: e.g. `'tnt_fixji_001'`
   - `app.is_platform_admin`: `'true'` **only** on authenticated Platform Admin endpoints
3. **Bootstrap Problem Solved Safely (`SECURITY DEFINER`):**
   Public widget requests arrive with `public_key` (`pub_fixji_...`) **before** `tenant_id` is known. A tightly scoped `SECURITY DEFINER` function resolves `public_key` $\rightarrow$ `tenant_id` (plus `allowed_origins` and `active_version_id`), and the Gateway immediately locks the transaction to that `tenant_id` via `set_config`.

---

## 1. Database Roles & Privileges

```sql
-- Run once during cluster/database provisioning
CREATE ROLE relayai_migrator NOLOGIN; -- or login for CI/CD migrations only
CREATE ROLE relayai_app LOGIN PASSWORD '<rotated-managed-identity-or-kv-secret>' NOSUPERUSER NOCREATEDB NOCREATEROLE NOBYPASSRLS;

GRANT USAGE ON SCHEMA public TO relayai_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO relayai_app;
-- Note: agent_versions UPDATE and audit_logs UPDATE/DELETE are still blocked by triggers
REVOKE UPDATE, DELETE ON audit_logs FROM relayai_app;
REVOKE UPDATE ON agent_versions FROM relayai_app;
```

---

## 2. Helper Context Functions

Using `NULLIF(current_setting(..., true), '')` avoids PostgreSQL's empty-string trap after `RESET` or connection pool reuse.

```sql
CREATE OR REPLACE FUNCTION current_app_tenant_id()
RETURNS VARCHAR(50) AS $$
    SELECT NULLIF(current_setting('app.current_tenant_id', true), '')::VARCHAR(50);
$$ LANGUAGE sql STABLE;

CREATE OR REPLACE FUNCTION is_platform_admin_session()
RETURNS BOOLEAN AS $$
    SELECT COALESCE(NULLIF(current_setting('app.is_platform_admin', true), '')::BOOLEAN, FALSE);
$$ LANGUAGE sql STABLE;
```

---

## 3. Enabling & Forcing RLS on All Tenant Tables

### `tenants` Table Policy
```sql
ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenants FORCE ROW LEVEL SECURITY;

CREATE POLICY tenants_isolation ON tenants
    USING (is_platform_admin_session() OR id = current_app_tenant_id())
    WITH CHECK (is_platform_admin_session());
```

### Standard Tenant-Owned Tables Policy Macro
Applied identically to: `tenant_members`, `tenant_invitations`, `secret_references`, `mcp_servers`, `mcp_tools`, `agents`, `agent_tools`, `agent_versions`, `deployments`, `conversations`, `messages`, `tool_executions`, `approvals`, `usage_records`:

```sql
DO $$
DECLARE
    tbl TEXT;
    tenant_tables TEXT[] := ARRAY[
        'tenant_members',
        'tenant_invitations',
        'secret_references',
        'mcp_servers',
        'mcp_tools',
        'agents',
        'agent_tools',
        'agent_versions',
        'deployments',
        'conversations',
        'messages',
        'tool_executions',
        'approvals',
        'usage_records'
    ];
BEGIN
    FOREACH tbl IN ARRAY tenant_tables LOOP
        EXECUTE format('ALTER TABLE %I ENABLE ROW LEVEL SECURITY;', tbl);
        EXECUTE format('ALTER TABLE %I FORCE ROW LEVEL SECURITY;', tbl);
        EXECUTE format(
            'CREATE POLICY tenant_isolation_%I ON %I ' ||
            'USING (is_platform_admin_session() OR tenant_id = current_app_tenant_id()) ' ||
            'WITH CHECK (is_platform_admin_session() OR tenant_id = current_app_tenant_id());',
            tbl, tbl
        );
    END LOOP;
END $$;
```

### `audit_logs` Policy (Platform + Tenant Scoped)
Platform operators see all logs; tenant workspaces see only rows where `tenant_id = current_app_tenant_id()` (never platform-global `tenant_id IS NULL` events or other tenants):

```sql
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs FORCE ROW LEVEL SECURITY;

CREATE POLICY audit_logs_read ON audit_logs
    FOR SELECT
    USING (
        is_platform_admin_session()
        OR (tenant_id IS NOT NULL AND tenant_id = current_app_tenant_id())
    );

CREATE POLICY audit_logs_insert ON audit_logs
    FOR INSERT
    WITH CHECK (
        is_platform_admin_session()
        OR (tenant_id IS NOT NULL AND tenant_id = current_app_tenant_id())
    );
```

---

## 4. `SECURITY DEFINER` Bootstrap Resolvers

Because `relayai_app` cannot query `deployments` or `tenant_invitations` before `app.current_tenant_id` is set, provide narrow, read-only lookup functions owned by `relayai_migrator` (with `SET search_path = public, pg_temp` to prevent search-path hijacking):

```sql
-- 1. Gateway Step 5A: Resolve public widget key -> tenant & active version
CREATE OR REPLACE FUNCTION resolve_deployment_by_public_key(p_public_key TEXT)
RETURNS TABLE (
    deployment_id     VARCHAR(50),
    tenant_id         VARCHAR(50),
    tenant_status     VARCHAR(20),
    rate_limit_rpm    INT,
    max_concurrency   INT,
    agent_id          VARCHAR(50),
    active_version_id VARCHAR(50),
    allowed_origins   JSONB,
    widget_config     JSONB,
    is_active         BOOLEAN
)
LANGUAGE sql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
    SELECT
        d.id,
        d.tenant_id,
        t.status,
        t.rate_limit_rpm,
        t.max_concurrency,
        d.agent_id,
        d.active_version_id,
        d.allowed_origins,
        d.widget_config,
        d.is_active
    FROM deployments d
    JOIN tenants t ON t.id = d.tenant_id
    WHERE d.public_key = p_public_key
    LIMIT 1;
$$;

-- 2. Onboarding Step 1B: Resolve invite token hash -> tenant & role
CREATE OR REPLACE FUNCTION resolve_invitation_by_token_hash(p_token_hash TEXT)
RETURNS TABLE (
    invitation_id UUID,
    tenant_id     VARCHAR(50),
    email         VARCHAR(255),
    role          VARCHAR(30),
    status        VARCHAR(20),
    expires_at    TIMESTAMPTZ
)
LANGUAGE sql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
    SELECT id, tenant_id, email, role, status, expires_at
    FROM tenant_invitations
    WHERE token_hash = p_token_hash
    LIMIT 1;
$$;

-- 3. Auth Helper: List user memberships after password/SSO login
CREATE OR REPLACE FUNCTION get_user_memberships(p_user_id UUID)
RETURNS TABLE (
    tenant_id   VARCHAR(50),
    tenant_name VARCHAR(100),
    role        VARCHAR(30),
    status      VARCHAR(20)
)
LANGUAGE sql
SECURITY DEFINER
SET search_path = public, pg_temp
AS $$
    SELECT tm.tenant_id, t.name, tm.role, tm.status
    FROM tenant_members tm
    JOIN tenants t ON t.id = tm.tenant_id
    WHERE tm.user_id = p_user_id
      AND tm.status = 'ACTIVE'
      AND t.status = 'ACTIVE';
$$;
```

### Mandatory Transaction Pattern in Backend Middleware

```sql
BEGIN;
-- Resolved via JWT workspace claim or resolve_deployment_by_public_key(...)
SELECT set_config('app.is_platform_admin', 'false', true);
SELECT set_config('app.current_tenant_id', 'tnt_fixji_001', true);

-- All subsequent SELECT/INSERT/UPDATE queries run strictly fenced to tnt_fixji_001
SELECT * FROM conversations WHERE id = :conv_id;
COMMIT;
```

---

## 5. Multi-Tenant Isolation Test Suite (Run in CI / QA)

```sql
-- Test 1: Unset tenant context must return 0 rows
BEGIN;
SET LOCAL ROLE relayai_app;
SELECT count(*) FROM conversations; -- MUST BE 0
ROLLBACK;

-- Test 2: FixJi session cannot read ShopNow rows even with unfiltered SELECT
BEGIN;
SET LOCAL ROLE relayai_app;
SELECT set_config('app.current_tenant_id', 'tnt_fixji_001', true);
SELECT count(*) FROM mcp_servers WHERE tenant_id = 'tnt_shop_002'; -- MUST BE 0
ROLLBACK;

-- Test 3: FixJi session cannot INSERT a row spoofed under ShopNow's tenant_id
BEGIN;
SET LOCAL ROLE relayai_app;
SELECT set_config('app.current_tenant_id', 'tnt_fixji_001', true);
INSERT INTO agents (id, tenant_id, name, draft_system_prompt)
VALUES ('agt_evil', 'tnt_shop_002', 'Spoofed', 'Bypass');
-- MUST FAIL: ERROR: new row violates row-level security policy for table "agents"
ROLLBACK;
```
