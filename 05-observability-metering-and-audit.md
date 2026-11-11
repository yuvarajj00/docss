# 05 — Usage Metering, Cost Attribution & Immutable Audit Logs

**Tables:** `usage_records`, `audit_logs`  
**Blueprint Alignment:** Sections 4.1, 4.2, 5.7, 12, 13, 17 · Prototype Pages 9–11

---

## Design Decisions

1. **Separated Telemetry (`usage_records`) vs. Security Forensics (`audit_logs`):**
   - `usage_records` tracks **token consumption, latency, model costs, and channel** (`WIDGET` vs `PLAYGROUND`) so Playground testing never corrupts live billing/customer KPIs.
   - `audit_logs` captures **who changed what and when** across both Platform Admin actions (`tenant_id IS NULL` or target tenant) and Client Workspace actions (`MCP_SERVER_ADDED`, `AGENT_PUBLISHED`, `DEPLOYMENT_ROLLED_BACK`, `APPROVAL_GRANTED`).
2. **Append-Only Enforcement:**
   - A `BEFORE UPDATE OR DELETE` trigger prevents tampering with `audit_logs` (except if an entire tenant is explicitly purged by platform compliance offboarding, or kept archived).
3. **High-Volume Partition Readiness:**
   - Both tables are append-only time-series tables (`created_at DESC`) designed to convert cleanly to monthly PostgreSQL declarative partitioning (`PARTITION BY RANGE (created_at)`) when daily request volume scales beyond millions of rows.

---

## 1. `usage_records`

Tracks LLM invocation counts, tokens, latency, and estimated spend attributable to each tenant and agent version.

```sql
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
```

### Example Monthly Quota Check Query

```sql
SELECT
    t.monthly_token_quota,
    COALESCE(SUM(u.input_tokens + u.output_tokens), 0) AS used_tokens_this_month
FROM tenants t
LEFT JOIN usage_records u
    ON u.tenant_id = t.id
    AND u.created_at >= date_trunc('month', NOW())
    AND u.channel <> 'PLAYGROUND'
WHERE t.id = :tenant_id
GROUP BY t.id, t.monthly_token_quota;
```

*(Note: At gateway runtime, check the pre-aggregated counter in **Redis** first for sub-millisecond enforcement, periodically reconciled against `usage_records`.)*

---

## 2. `audit_logs`

Immutable security and governance event trail powering both **Platform Admin — Audit Logs** (Page 11) and tenant compliance exports.

```sql
CREATE TABLE audit_logs (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id     VARCHAR(50) REFERENCES tenants(id) ON DELETE CASCADE, -- NULL for platform-global actions
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

CREATE OR REPLACE FUNCTION prevent_audit_log_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs is append-only and cannot be modified or deleted directly';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_logs_immutable
    BEFORE UPDATE OR DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_log_mutation();
```

### Standard Event Vocabulary (`action`)

| Category | Event Actions |
| --- | --- |
| **Tenancy & RBAC** | `TENANT_CREATED`, `TENANT_QUOTA_UPDATED`, `MEMBER_INVITED`, `MEMBER_JOINED`, `MEMBER_ROLE_CHANGED` |
| **MCP & Secrets** | `SECRET_CREATED`, `SECRET_ROTATED`, `MCP_SERVER_REGISTERED`, `MCP_TOOLS_DISCOVERED`, `MCP_SSRF_BLOCKED` |
| **Agents & Deploy** | `AGENT_PUBLISHED`, `DEPLOYMENT_UPDATED`, `DEPLOYMENT_ROLLED_BACK`, `DEPLOYMENT_DISABLED` |
| **HITL & Runtime** | `APPROVAL_REQUESTED`, `APPROVAL_GRANTED`, `APPROVAL_REJECTED`, `APPROVAL_EXPIRED`, `TOOL_POLICY_DENIED` |

**Example Row (matches Page 9 Recent Platform Events):**
```json
{
  "id": "e1a2b3c4-1111-2222-3333-444455556666",
  "tenant_id": "tnt_fixji_001",
  "actor_user_id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "actor_type": "USER",
  "action": "AGENT_PUBLISHED",
  "target_type": "agent_version",
  "target_id": "ver_v1_2",
  "outcome": "SUCCESS",
  "details": {
    "agent_id": "agt_fixji_booking",
    "version_tag": "v1.2",
    "enabled_tool_count": 4
  },
  "created_at": "2026-11-09T10:12:00Z"
}
```
