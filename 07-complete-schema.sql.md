# 07 — Complete Consolidated PostgreSQL DDL & FixJi End-to-End Seed

This file provides the single, topologically ordered **PostgreSQL 16+** migration script combining all 17 tables, composite FKs, immutability triggers, indexes, RLS policies, and `SECURITY DEFINER` bootstrap resolvers, followed by an executable **FixJi Steps 1–7 verification test**.

---

## Part A: Consolidated Migration DDL

```sql
-- ============================================================================
-- RELAYAI PRODUCTION SCHEMA (PostgreSQL 16+)
-- ============================================================================

-- 0. Shared trigger functions
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION prevent_agent_version_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'agent_versions rows are immutable and cannot be updated';
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION prevent_audit_log_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only and cannot be modified or deleted directly';
END;
$$ LANGUAGE plpgsql;

-- 1. Tenancy & Identity
CREATE TABLE tenants (
    id                  VARCHAR(50) PRIMARY KEY,
    name                VARCHAR(100) NOT NULL,
    slug                VARCHAR(100) UNIQUE NOT NULL,
    business_type       VARCHAR(100),
    plan_tier           VARCHAR(30) NOT NULL DEFAULT 'STARTER'
                        CHECK (plan_tier IN ('STARTER', 'GROWTH', 'ENTERPRISE')),
    status              VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
                        CHECK (status IN ('ACTIVE', 'SUSPENDED', 'ARCHIVED')),
    rate_limit_rpm      INT NOT NULL DEFAULT 100 CHECK (rate_limit_rpm > 0),
    max_concurrency     INT NOT NULL DEFAULT 20 CHECK (max_concurrency > 0),
    monthly_token_quota BIGINT NOT NULL DEFAULT 5000000 CHECK (monthly_token_quota >= 0),
    data_retention_days INT NOT NULL DEFAULT 90 CHECK (data_retention_days > 0),
    settings            JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TRIGGER trg_tenants_updated_at
    BEFORE UPDATE ON tenants FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE users (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email             VARCHAR(255) UNIQUE NOT NULL,
    password_hash     TEXT,
    external_auth_sub VARCHAR(255) UNIQUE,
    full_name         VARCHAR(150) NOT NULL,
    is_platform_admin BOOLEAN NOT NULL DEFAULT FALSE,
    is_active         BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at     TIMESTAMPTZ,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (email = LOWER(email))
);

CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE tenant_members (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role       VARCHAR(30) NOT NULL
               CHECK (role IN ('OWNER', 'ADMIN', 'DEVELOPER', 'ANALYST', 'APPROVER')),
    status     VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
               CHECK (status IN ('ACTIVE', 'DISABLED')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, user_id)
);

CREATE INDEX idx_tenant_members_user ON tenant_members(user_id) WHERE status = 'ACTIVE';

CREATE TRIGGER trg_tenant_members_updated_at
    BEFORE UPDATE ON tenant_members FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE tenant_invitations (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id          VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email              VARCHAR(255) NOT NULL,
    role               VARCHAR(30) NOT NULL
                       CHECK (role IN ('OWNER', 'ADMIN', 'DEVELOPER', 'ANALYST', 'APPROVER')),
    token_hash         VARCHAR(64) UNIQUE NOT NULL,
    invited_by_user_id UUID NOT NULL REFERENCES users(id),
    status             VARCHAR(20) NOT NULL DEFAULT 'PENDING'
                       CHECK (status IN ('PENDING', 'ACCEPTED', 'REVOKED', 'EXPIRED')),
    expires_at         TIMESTAMPTZ NOT NULL,
    accepted_at        TIMESTAMPTZ,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    CHECK (email = LOWER(email))
);

CREATE UNIQUE INDEX idx_tenant_invites_pending_email
    ON tenant_invitations(tenant_id, email) WHERE status = 'PENDING';

-- 2. Vault References, MCP Servers & Tools
CREATE TABLE secret_references (
    id                    VARCHAR(50) PRIMARY KEY,
    tenant_id             VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    scope                 VARCHAR(30) NOT NULL DEFAULT 'MCP_AUTH'
                          CHECK (scope IN ('MCP_AUTH', 'LLM_PROVIDER', 'WEBHOOK')),
    name                  VARCHAR(100) NOT NULL,
    key_vault_secret_name VARCHAR(255) UNIQUE NOT NULL,
    secret_version        VARCHAR(100),
    created_by_user_id    UUID REFERENCES users(id),
    rotated_at            TIMESTAMPTZ,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id)
);

CREATE TABLE mcp_servers (
    id                 VARCHAR(50) PRIMARY KEY,
    tenant_id          VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name               VARCHAR(100) NOT NULL,
    endpoint_url       TEXT NOT NULL,
    transport          VARCHAR(30) NOT NULL DEFAULT 'STREAMABLE_HTTP'
                       CHECK (transport IN ('STREAMABLE_HTTP', 'SSE')),
    auth_type          VARCHAR(30) NOT NULL DEFAULT 'BEARER_TOKEN'
                       CHECK (auth_type IN ('NONE', 'BEARER_TOKEN', 'API_KEY', 'CUSTOM_HEADER', 'OAUTH2_CLIENT_CREDENTIALS')),
    auth_header_name   VARCHAR(100),
    secret_ref_id      VARCHAR(50),
    request_timeout_ms INT NOT NULL DEFAULT 15000 CHECK (request_timeout_ms BETWEEN 1000 AND 60000),
    max_response_bytes INT NOT NULL DEFAULT 262144 CHECK (max_response_bytes BETWEEN 1024 AND 2097152),
    status             VARCHAR(20) NOT NULL DEFAULT 'CONNECTED'
                       CHECK (status IN ('CONNECTED', 'ERROR', 'DISABLED')),
    last_tested_at     TIMESTAMPTZ,
    last_error_code    VARCHAR(100),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, name),
    FOREIGN KEY (tenant_id, secret_ref_id) REFERENCES secret_references(tenant_id, id),
    CHECK (
        (auth_type = 'NONE' AND secret_ref_id IS NULL) OR
        (auth_type <> 'NONE' AND secret_ref_id IS NOT NULL)
    ),
    CHECK (endpoint_url ~* '^https?://')
);

CREATE TRIGGER trg_mcp_servers_updated_at
    BEFORE UPDATE ON mcp_servers FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE mcp_tools (
    id                        VARCHAR(50) PRIMARY KEY,
    tenant_id                 VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    mcp_server_id             VARCHAR(50) NOT NULL,
    tool_name                 VARCHAR(150) NOT NULL,
    description               TEXT,
    input_schema              JSONB NOT NULL,
    schema_hash               VARCHAR(64) NOT NULL,
    operation_type            VARCHAR(20) NOT NULL DEFAULT 'READ'
                              CHECK (operation_type IN ('READ', 'WRITE', 'DESTRUCTIVE')),
    default_requires_approval BOOLEAN NOT NULL DEFAULT FALSE,
    is_active                 BOOLEAN NOT NULL DEFAULT TRUE,
    last_synced_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at                TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (mcp_server_id, tool_name),
    FOREIGN KEY (tenant_id, mcp_server_id) REFERENCES mcp_servers(tenant_id, id) ON DELETE CASCADE
);

CREATE INDEX idx_mcp_tools_tenant_active ON mcp_tools(tenant_id, mcp_server_id) WHERE is_active = TRUE;

-- 3. Agents, Draft Tool Bindings, Immutable Versions & Deployments
CREATE TABLE agents (
    id                          VARCHAR(50) PRIMARY KEY,
    tenant_id                   VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                        VARCHAR(100) NOT NULL,
    description                 TEXT,
    execution_mode              VARCHAR(30) NOT NULL DEFAULT 'HYBRID'
                                CHECK (execution_mode IN ('AUTONOMOUS', 'HYBRID', 'ASSISTED')),
    draft_persona               TEXT,
    draft_system_prompt         TEXT NOT NULL,
    draft_model_provider        VARCHAR(50) NOT NULL DEFAULT 'azure_ai_foundry'
                                CHECK (draft_model_provider IN ('azure_ai_foundry', 'openai', 'gemini', 'anthropic')),
    draft_model_name            VARCHAR(100) NOT NULL DEFAULT 'gpt-4.1',
    draft_model_settings        JSONB NOT NULL DEFAULT '{"temperature": 0.2, "max_tokens": 2048}'::jsonb,
    draft_memory_config         JSONB NOT NULL DEFAULT '{"history_window": 20, "summarize_after": 30}'::jsonb,
    draft_safety_config         JSONB NOT NULL DEFAULT '{"max_tool_calls_per_turn": 5}'::jsonb,
    latest_published_version_id VARCHAR(50),
    created_by_user_id          UUID REFERENCES users(id),
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, name)
);

CREATE TRIGGER trg_agents_updated_at
    BEFORE UPDATE ON agents FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE agent_tools (
    id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id                VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    agent_id                 VARCHAR(50) NOT NULL,
    mcp_tool_id              VARCHAR(50) NOT NULL,
    is_enabled               BOOLEAN NOT NULL DEFAULT TRUE,
    requires_approval        BOOLEAN NOT NULL DEFAULT FALSE,
    approval_timeout_seconds INT NOT NULL DEFAULT 900 CHECK (approval_timeout_seconds BETWEEN 60 AND 604800),
    created_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (agent_id, mcp_tool_id),
    FOREIGN KEY (tenant_id, agent_id) REFERENCES agents(tenant_id, id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id, mcp_tool_id) REFERENCES mcp_tools(tenant_id, id) ON DELETE CASCADE
);

CREATE INDEX idx_agent_tools_lookup ON agent_tools(tenant_id, agent_id) WHERE is_enabled = TRUE;

CREATE TRIGGER trg_agent_tools_updated_at
    BEFORE UPDATE ON agent_tools FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE agent_versions (
    id                   VARCHAR(50) PRIMARY KEY,
    tenant_id            VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    agent_id             VARCHAR(50) NOT NULL,
    version_number       INT NOT NULL CHECK (version_number > 0),
    version_tag          VARCHAR(30) NOT NULL,
    config_checksum      VARCHAR(64) NOT NULL,
    published_config     JSONB NOT NULL,
    published_by_user_id UUID NOT NULL REFERENCES users(id),
    published_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, agent_id, id),
    UNIQUE (agent_id, version_number),
    UNIQUE (agent_id, version_tag),
    FOREIGN KEY (tenant_id, agent_id) REFERENCES agents(tenant_id, id) ON DELETE CASCADE
);

ALTER TABLE agents
    ADD CONSTRAINT fk_agents_latest_version
    FOREIGN KEY (tenant_id, id, latest_published_version_id)
    REFERENCES agent_versions(tenant_id, agent_id, id)
    ON DELETE SET NULL (latest_published_version_id)
    DEFERRABLE INITIALLY DEFERRED;

CREATE TRIGGER trg_agent_versions_immutable
    BEFORE UPDATE ON agent_versions FOR EACH ROW EXECUTE FUNCTION prevent_agent_version_mutation();

CREATE TABLE deployments (
    id                  VARCHAR(50) PRIMARY KEY,
    tenant_id           VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    agent_id            VARCHAR(50) NOT NULL,
    public_key          VARCHAR(100) UNIQUE NOT NULL,
    name                VARCHAR(100) NOT NULL DEFAULT 'Production Website',
    active_version_id   VARCHAR(50) NOT NULL,
    previous_version_id VARCHAR(50),
    allowed_origins     JSONB NOT NULL,
    widget_config       JSONB NOT NULL DEFAULT '{"position": "bottom-right", "theme": "light"}'::jsonb,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    FOREIGN KEY (tenant_id, agent_id, active_version_id)
        REFERENCES agent_versions(tenant_id, agent_id, id),
    FOREIGN KEY (tenant_id, agent_id, previous_version_id)
        REFERENCES agent_versions(tenant_id, agent_id, id),
    CHECK (jsonb_typeof(allowed_origins) = 'array')
);

CREATE INDEX idx_deployments_public_key ON deployments(public_key) WHERE is_active = TRUE;

CREATE TRIGGER trg_deployments_updated_at
    BEFORE UPDATE ON deployments FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- 4. Runtime Conversations, Messages, Tool Executions & HITL Approvals
CREATE TABLE conversations (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id          VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    display_code       VARCHAR(30) NOT NULL,
    channel            VARCHAR(20) NOT NULL CHECK (channel IN ('WIDGET', 'PLAYGROUND', 'API')),
    deployment_id      VARCHAR(50),
    agent_id           VARCHAR(50) NOT NULL,
    pinned_version_id  VARCHAR(50),
    visitor_token_hash VARCHAR(64),
    created_by_user_id UUID REFERENCES users(id),
    status             VARCHAR(30) NOT NULL DEFAULT 'ACTIVE'
                       CHECK (status IN ('ACTIVE', 'WAITING_FOR_APPROVAL', 'CLOSED')),
    metadata           JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, display_code),
    FOREIGN KEY (tenant_id, agent_id) REFERENCES agents(tenant_id, id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id, deployment_id) REFERENCES deployments(tenant_id, id),
    FOREIGN KEY (tenant_id, agent_id, pinned_version_id)
        REFERENCES agent_versions(tenant_id, agent_id, id),
    CHECK (
        (channel = 'WIDGET' AND deployment_id IS NOT NULL AND pinned_version_id IS NOT NULL AND visitor_token_hash IS NOT NULL)
        OR channel IN ('PLAYGROUND', 'API')
    )
);

CREATE INDEX idx_conversations_tenant_recent ON conversations(tenant_id, updated_at DESC);
CREATE INDEX idx_conversations_waiting ON conversations(tenant_id, status) WHERE status = 'WAITING_FOR_APPROVAL';

CREATE TRIGGER trg_conversations_updated_at
    BEFORE UPDATE ON conversations FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    conversation_id UUID NOT NULL,
    sequence_no     INT NOT NULL CHECK (sequence_no > 0),
    sender_role     VARCHAR(20) NOT NULL CHECK (sender_role IN ('SYSTEM', 'USER', 'ASSISTANT', 'TOOL')),
    visibility      VARCHAR(20) NOT NULL DEFAULT 'PUBLIC' CHECK (visibility IN ('PUBLIC', 'INTERNAL')),
    content         TEXT,
    tool_calls      JSONB,
    tool_call_id    VARCHAR(150),
    metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (conversation_id, sequence_no),
    FOREIGN KEY (tenant_id, conversation_id) REFERENCES conversations(tenant_id, id) ON DELETE CASCADE,
    CHECK (sender_role <> 'TOOL' OR tool_call_id IS NOT NULL)
);

CREATE INDEX idx_messages_widget_poll ON messages(conversation_id, visibility, sequence_no)
    WHERE visibility = 'PUBLIC';

CREATE TABLE tool_executions (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    conversation_id  UUID NOT NULL,
    agent_version_id VARCHAR(50),
    mcp_server_id    VARCHAR(50) NOT NULL,
    mcp_tool_id      VARCHAR(50) NOT NULL,
    tool_name        VARCHAR(150) NOT NULL,
    llm_tool_call_id VARCHAR(150) NOT NULL,
    input_arguments  JSONB NOT NULL,
    output_payload   JSONB,
    status           VARCHAR(30) NOT NULL
                     CHECK (status IN (
                         'STARTED', 'WAITING_APPROVAL', 'EXECUTING',
                         'SUCCESS', 'FAILED', 'TIMEOUT', 'AMBIGUOUS_TIMEOUT',
                         'SCHEMA_MISMATCH', 'POLICY_DENIED', 'REJECTED', 'EXPIRED', 'CANCELLED'
                     )),
    is_dry_run       BOOLEAN NOT NULL DEFAULT FALSE,
    error_code       VARCHAR(100),
    error_message    TEXT,
    latency_ms       INT CHECK (latency_ms IS NULL OR latency_ms >= 0),
    created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at     TIMESTAMPTZ,
    UNIQUE (tenant_id, id),
    FOREIGN KEY (tenant_id, conversation_id) REFERENCES conversations(tenant_id, id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id, agent_version_id) REFERENCES agent_versions(tenant_id, id),
    FOREIGN KEY (tenant_id, mcp_server_id) REFERENCES mcp_servers(tenant_id, id),
    FOREIGN KEY (tenant_id, mcp_tool_id) REFERENCES mcp_tools(tenant_id, id),
    CHECK (is_dry_run = TRUE OR agent_version_id IS NOT NULL)
);

CREATE INDEX idx_tool_exec_tenant_time ON tool_executions(tenant_id, created_at DESC);
CREATE INDEX idx_tool_exec_conv ON tool_executions(conversation_id, created_at);

CREATE TABLE approvals (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id          VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    display_ref        VARCHAR(30) NOT NULL,
    conversation_id    UUID NOT NULL,
    agent_version_id   VARCHAR(50),
    tool_execution_id  UUID NOT NULL UNIQUE,
    mcp_server_id      VARCHAR(50) NOT NULL,
    mcp_tool_id        VARCHAR(50) NOT NULL,
    tool_name          VARCHAR(150) NOT NULL,
    llm_tool_call_id   VARCHAR(150) NOT NULL,
    proposed_arguments JSONB NOT NULL,
    ui_summary         JSONB NOT NULL DEFAULT '{}'::jsonb,
    status             VARCHAR(25) NOT NULL DEFAULT 'PENDING'
                       CHECK (status IN ('PENDING', 'EXECUTING', 'APPROVED', 'REJECTED', 'FAILED', 'AMBIGUOUS_TIMEOUT', 'EXPIRED', 'CANCELLED')),
    is_sandbox         BOOLEAN NOT NULL DEFAULT FALSE,
    idempotency_key    VARCHAR(255) UNIQUE NOT NULL,
    expires_at         TIMESTAMPTZ NOT NULL,
    decided_by_user_id UUID REFERENCES users(id),
    decision_reason    TEXT,
    decided_at         TIMESTAMPTZ,
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, display_ref),
    FOREIGN KEY (tenant_id, conversation_id) REFERENCES conversations(tenant_id, id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id, agent_version_id) REFERENCES agent_versions(tenant_id, id),
    FOREIGN KEY (tenant_id, tool_execution_id) REFERENCES tool_executions(tenant_id, id) ON DELETE CASCADE,
    FOREIGN KEY (tenant_id, mcp_server_id) REFERENCES mcp_servers(tenant_id, id),
    FOREIGN KEY (tenant_id, mcp_tool_id) REFERENCES mcp_tools(tenant_id, id),
    CHECK (is_sandbox = TRUE OR agent_version_id IS NOT NULL),
    CHECK (
        status IN ('PENDING', 'EXPIRED', 'CANCELLED')
        OR (decided_by_user_id IS NOT NULL AND decided_at IS NOT NULL)
    )
);

CREATE INDEX idx_approvals_pending_queue ON approvals(tenant_id, status, expires_at) WHERE status = 'PENDING';

CREATE TRIGGER trg_approvals_updated_at
    BEFORE UPDATE ON approvals FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- 5. Usage Metering & Immutable Audit Logs
CREATE TABLE usage_records (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id          VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    conversation_id    UUID,
    message_id         UUID,
    agent_version_id   VARCHAR(50),
    channel            VARCHAR(20) NOT NULL CHECK (channel IN ('WIDGET', 'PLAYGROUND', 'API')),
    model_provider     VARCHAR(50) NOT NULL,
    model_name         VARCHAR(100) NOT NULL,
    input_tokens       INT NOT NULL DEFAULT 0 CHECK (input_tokens >= 0),
    output_tokens      INT NOT NULL DEFAULT 0 CHECK (output_tokens >= 0),
    cached_tokens      INT NOT NULL DEFAULT 0 CHECK (cached_tokens >= 0),
    estimated_cost_usd NUMERIC(10, 6) NOT NULL DEFAULT 0 CHECK (estimated_cost_usd >= 0),
    latency_ms         INT NOT NULL DEFAULT 0 CHECK (latency_ms >= 0),
    status             VARCHAR(20) NOT NULL DEFAULT 'SUCCESS'
                       CHECK (status IN ('SUCCESS', 'ERROR', 'RATE_LIMITED')),
    error_code         VARCHAR(100),
    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    FOREIGN KEY (tenant_id, conversation_id) REFERENCES conversations(tenant_id, id) ON DELETE SET NULL,
    FOREIGN KEY (tenant_id, message_id) REFERENCES messages(tenant_id, id) ON DELETE SET NULL,
    FOREIGN KEY (tenant_id, agent_version_id) REFERENCES agent_versions(tenant_id, id) ON DELETE SET NULL
);

CREATE INDEX idx_usage_tenant_month ON usage_records(tenant_id, created_at DESC);
CREATE INDEX idx_usage_channel_status ON usage_records(tenant_id, channel, status, created_at DESC);

CREATE TABLE audit_logs (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     VARCHAR(50) REFERENCES tenants(id) ON DELETE CASCADE,
    actor_user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    actor_type    VARCHAR(20) NOT NULL DEFAULT 'USER'
                  CHECK (actor_type IN ('USER', 'PLATFORM_ADMIN', 'SYSTEM')),
    action        VARCHAR(100) NOT NULL,
    target_type   VARCHAR(50) NOT NULL,
    target_id     VARCHAR(100) NOT NULL,
    outcome       VARCHAR(20) NOT NULL DEFAULT 'SUCCESS'
                  CHECK (outcome IN ('SUCCESS', 'DENIED', 'FAILED')),
    details       JSONB NOT NULL DEFAULT '{}'::jsonb,
    ip_address    INET,
    user_agent    TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_tenant_recent ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX idx_audit_platform_recent ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_action ON audit_logs(action, created_at DESC);

CREATE TRIGGER trg_audit_logs_immutable
    BEFORE UPDATE OR DELETE ON audit_logs FOR EACH ROW EXECUTE FUNCTION prevent_audit_log_mutation();

-- 6. RLS Helpers & Policies
CREATE OR REPLACE FUNCTION current_app_tenant_id()
RETURNS VARCHAR(50) AS $$
    SELECT NULLIF(current_setting('app.current_tenant_id', true), '')::VARCHAR(50);
$$ LANGUAGE sql STABLE;

CREATE OR REPLACE FUNCTION is_platform_admin_session()
RETURNS BOOLEAN AS $$
    SELECT COALESCE(NULLIF(current_setting('app.is_platform_admin', true), '')::BOOLEAN, FALSE);
$$ LANGUAGE sql STABLE;

ALTER TABLE tenants ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenants FORCE ROW LEVEL SECURITY;
CREATE POLICY tenants_isolation ON tenants
    USING (is_platform_admin_session() OR id = current_app_tenant_id())
    WITH CHECK (is_platform_admin_session());

DO $$
DECLARE
    tbl TEXT;
    tenant_tables TEXT[] := ARRAY[
        'tenant_members', 'tenant_invitations', 'secret_references',
        'mcp_servers', 'mcp_tools', 'agents', 'agent_tools',
        'agent_versions', 'deployments', 'conversations', 'messages',
        'tool_executions', 'approvals', 'usage_records'
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

ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs FORCE ROW LEVEL SECURITY;

CREATE POLICY audit_logs_read ON audit_logs FOR SELECT
    USING (is_platform_admin_session() OR (tenant_id IS NOT NULL AND tenant_id = current_app_tenant_id()));

CREATE POLICY audit_logs_insert ON audit_logs FOR INSERT
    WITH CHECK (is_platform_admin_session() OR (tenant_id IS NOT NULL AND tenant_id = current_app_tenant_id()));

-- 7. SECURITY DEFINER Bootstrap Functions
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
        d.id, d.tenant_id, t.status, t.rate_limit_rpm, t.max_concurrency,
        d.agent_id, d.active_version_id, d.allowed_origins, d.widget_config, d.is_active
    FROM deployments d
    JOIN tenants t ON t.id = d.tenant_id
    WHERE d.public_key = p_public_key
    LIMIT 1;
$$;

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

---

## Part B: FixJi Step 1–7 Smoke Test SQL

Run this inside a transaction as Platform Admin (`app.is_platform_admin = 'true'`) to verify every foreign key and constraint works end-to-end:

```sql
BEGIN;
SELECT set_config('app.is_platform_admin', 'true', true);

-- Step 1: Tenant & Team
INSERT INTO tenants (id, name, slug, business_type, plan_tier, rate_limit_rpm)
VALUES ('tnt_fixji_001', 'FixJi', 'fixji', 'Home services', 'GROWTH', 100);

INSERT INTO users (id, email, full_name) VALUES
    ('11111111-1111-1111-1111-111111111101', 'founder@fixji.in', 'FixJi Founder'),
    ('11111111-1111-1111-1111-111111111102', 'dev@fixji.in', 'Karthik'),
    ('11111111-1111-1111-1111-111111111103', 'manager@fixji.in', 'Suresh');

INSERT INTO tenant_members (tenant_id, user_id, role) VALUES
    ('tnt_fixji_001', '11111111-1111-1111-1111-111111111101', 'OWNER'),
    ('tnt_fixji_001', '11111111-1111-1111-1111-111111111102', 'DEVELOPER'),
    ('tnt_fixji_001', '11111111-1111-1111-1111-111111111103', 'APPROVER');

-- Step 2: Secret Ref, MCP Server & Tools
INSERT INTO secret_references (id, tenant_id, name, key_vault_secret_name, created_by_user_id)
VALUES ('sec_fixji_01', 'tnt_fixji_001', 'FixJi MCP Bearer', 'relayai-tnt-fixji-001-sec-01', '11111111-1111-1111-1111-111111111102');

INSERT INTO mcp_servers (id, tenant_id, name, endpoint_url, transport, auth_type, secret_ref_id)
VALUES ('srv_01', 'tnt_fixji_001', 'FixJi Booking MCP', 'https://mcp.fixji.in/mcp', 'STREAMABLE_HTTP', 'BEARER_TOKEN', 'sec_fixji_01');

INSERT INTO mcp_tools (id, tenant_id, mcp_server_id, tool_name, input_schema, schema_hash, operation_type, default_requires_approval) VALUES
    ('tl_2', 'tnt_fixji_001', 'srv_01', 'check_availability', '{"type":"object"}', 'hash_2', 'READ', false),
    ('tl_3', 'tnt_fixji_001', 'srv_01', 'create_booking', '{"type":"object"}', 'hash_3', 'WRITE', true);

-- Step 3 & 4: Agent Draft, Bindings, Version v1.2, Deployment
INSERT INTO agents (id, tenant_id, name, draft_persona, draft_system_prompt, created_by_user_id)
VALUES ('agt_fixji_booking', 'tnt_fixji_001', 'FixJi Booking Assistant', 'Home Services Assistant', 'Help customers book services.', '11111111-1111-1111-1111-111111111101');

INSERT INTO agent_tools (tenant_id, agent_id, mcp_tool_id, is_enabled, requires_approval) VALUES
    ('tnt_fixji_001', 'agt_fixji_booking', 'tl_2', true, false),
    ('tnt_fixji_001', 'agt_fixji_booking', 'tl_3', true, true);

INSERT INTO agent_versions (id, tenant_id, agent_id, version_number, version_tag, config_checksum, published_config, published_by_user_id)
VALUES ('ver_v1_2', 'tnt_fixji_001', 'agt_fixji_booking', 1, 'v1.2', 'chk_123', '{"tools":["tl_2","tl_3"]}', '11111111-1111-1111-1111-111111111101');

UPDATE agents SET latest_published_version_id = 'ver_v1_2' WHERE id = 'agt_fixji_booking';

INSERT INTO deployments (id, tenant_id, agent_id, public_key, active_version_id, allowed_origins)
VALUES ('dep_fixji_live', 'tnt_fixji_001', 'agt_fixji_booking', 'pub_demo_123', 'ver_v1_2', '["https://fixji.in"]');

-- Step 5 & 6: Conversation, Tool Execution, Pending Approval BK-2048
INSERT INTO conversations (id, tenant_id, display_code, channel, deployment_id, agent_id, pinned_version_id, visitor_token_hash, status)
VALUES ('7f9c2d10-3a44-4c5b-9f10-8a7b6c5d4e3f', 'tnt_fixji_001', 'conv_7121', 'WIDGET', 'dep_fixji_live', 'agt_fixji_booking', 'ver_v1_2', 'sha256_visitor_abc', 'WAITING_FOR_APPROVAL');

INSERT INTO tool_executions (id, tenant_id, conversation_id, agent_version_id, mcp_server_id, mcp_tool_id, tool_name, llm_tool_call_id, input_arguments, status)
VALUES ('9a1b2c3d-4e5f-6789-abcd-ef0123456789', 'tnt_fixji_001', '7f9c2d10-3a44-4c5b-9f10-8a7b6c5d4e3f', 'ver_v1_2', 'srv_01', 'tl_3', 'create_booking', 'call_2', '{"service":"House painting","area":"Nellore","time":"Tomorrow - 10 AM"}', 'WAITING_APPROVAL');

INSERT INTO approvals (tenant_id, display_ref, conversation_id, agent_version_id, tool_execution_id, mcp_server_id, mcp_tool_id, tool_name, llm_tool_call_id, proposed_arguments, idempotency_key, expires_at)
VALUES ('tnt_fixji_001', 'BK-2048', '7f9c2d10-3a44-4c5b-9f10-8a7b6c5d4e3f', 'ver_v1_2', '9a1b2c3d-4e5f-6789-abcd-ef0123456789', 'srv_01', 'tl_3', 'create_booking', 'call_2', '{"service":"House painting","area":"Nellore","time":"Tomorrow - 10 AM"}', 'idem_conv_7121_call_2', NOW() + INTERVAL '15 minutes');

ROLLBACK;
```
