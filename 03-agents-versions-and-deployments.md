# 03 — Agents, Tool Bindings, Immutable Versions & Deployments

**Tables:** `agents`, `agent_tools`, `agent_versions`, `deployments`  
**Blueprint Alignment:** Sections 5.4, 5.5, 5.6, 12, 14 · FixJi Workflow Steps 3 & 4 · Prototype Pages 11–13

---

## Design Decisions

1. **Relational Draft Bindings (`agent_tools`) + Immutable Snapshot (`agent_versions`):**
   - In the mutable draft (`agents`), tools are assigned via `agent_tools` with composite FKs to both `agents(tenant_id, id)` and `mcp_tools(tenant_id, id)`. This prevents cross-tenant tool assignment, prevents tool-name collisions across multiple MCP servers, and powers the Page 12 **Tool Catalogue → Agent assignment** view with a direct indexed join.
   - When an Admin clicks **Publish**, the backend compiles `agents` + `agent_tools` + `mcp_tools` + `mcp_servers` into a self-contained JSONB snapshot inside `agent_versions.published_config`. Once written, a database trigger forbids modifying or deleting that version.
2. **Opaque Public Key vs. Internal ID on `deployments`:**
   - Prototype Page 13 shows `data-public-key="pub_demo_123"` alongside `data-agent`. In production, `deployments.public_key` (`dep_live_...` or `pub_...`) is the single opaque lookup credential sent by the widget. The server never trusts any browser-provided `tenant_id` or `agent_id`.
3. **Strict Same-Agent Version Guard:**
   - `deployments` uses a 3-column composite FK `(tenant_id, agent_id, active_version_id) REFERENCES agent_versions(tenant_id, agent_id, id)`. It is impossible to deploy a version belonging to a different agent or different tenant.
4. **1-Second Rollback Tracking:**
   - `deployments.previous_version_id` stores the prior active version alongside an `audit_logs` entry on every rollout/rollback.

---

## 1. `agents`

Mutable workspace draft authored in Agent Studio and tested in Agent Playground.

```sql
CREATE TABLE agents (
    id                           VARCHAR(50) PRIMARY KEY,
    tenant_id                    VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name                         VARCHAR(100) NOT NULL,
    description                  TEXT,
    execution_mode               VARCHAR(30) NOT NULL DEFAULT 'HYBRID'
                                 CHECK (execution_mode IN ('AUTONOMOUS', 'HYBRID', 'ASSISTED')),
    draft_persona                TEXT,
    draft_system_prompt          TEXT NOT NULL,
    draft_model_provider         VARCHAR(50) NOT NULL DEFAULT 'azure_ai_foundry'
                                 CHECK (draft_model_provider IN ('azure_ai_foundry', 'openai', 'gemini', 'anthropic')),
    draft_model_name             VARCHAR(100) NOT NULL DEFAULT 'gpt-4.1',
    draft_model_settings         JSONB NOT NULL DEFAULT '{"temperature": 0.2, "max_tokens": 2048}'::jsonb,
    draft_memory_config          JSONB NOT NULL DEFAULT '{"history_window": 20, "summarize_after": 30}'::jsonb,
    draft_safety_config          JSONB NOT NULL DEFAULT '{"max_tool_calls_per_turn": 5}'::jsonb,
    latest_published_version_id  VARCHAR(50),
    created_by_user_id           UUID REFERENCES users(id),
    created_at                   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (tenant_id, name)
);

CREATE TRIGGER trg_agents_updated_at
    BEFORE UPDATE ON agents
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 2. `agent_tools`

Draft-stage authorization matrix mapping an agent to allowed MCP tools and HITL approval rules.

```sql
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
    BEFORE UPDATE ON agent_tools
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 3. `agent_versions`

Immutable compiled configuration snapshots created on **Publish**. Live conversations pin `agent_versions.id`.

```sql
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

-- Link agents.latest_published_version_id back after agent_versions exists
ALTER TABLE agents
    ADD CONSTRAINT fk_agents_latest_version
    FOREIGN KEY (tenant_id, id, latest_published_version_id)
    REFERENCES agent_versions(tenant_id, agent_id, id)
    ON DELETE SET NULL (latest_published_version_id)
    DEFERRABLE INITIALLY DEFERRED;

CREATE OR REPLACE FUNCTION prevent_agent_version_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'agent_versions rows are immutable and cannot be updated';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_agent_versions_immutable
    BEFORE UPDATE ON agent_versions
    FOR EACH ROW EXECUTE FUNCTION prevent_agent_version_mutation();
```

### Canonical `published_config` JSONB Snapshot Structure

Every runtime call reads this single JSON document without assembling live draft state:

```json
{
  "agent_id": "agt_fixji_booking",
  "agent_name": "FixJi Booking Assistant",
  "persona": "Home Services Assistant",
  "system_prompt": "Help customers discover services, check technician availability, and prepare bookings.",
  "model": {
    "provider": "azure_ai_foundry",
    "model_name": "gpt-4.1",
    "temperature": 0.2,
    "max_tokens": 2048
  },
  "memory": {
    "history_window": 20,
    "summarize_after": 30
  },
  "safety": {
    "max_tool_calls_per_turn": 5
  },
  "tools": [
    {
      "mcp_tool_id": "tl_1",
      "mcp_server_id": "srv_01",
      "tool_name": "list_services",
      "operation_type": "READ",
      "requires_approval": false,
      "schema_hash": "abc123...",
      "input_schema": { "type": "object", "properties": { "city": { "type": "string" } } }
    },
    {
      "mcp_tool_id": "tl_3",
      "mcp_server_id": "srv_01",
      "tool_name": "create_booking",
      "operation_type": "WRITE",
      "requires_approval": true,
      "approval_timeout_seconds": 900,
      "schema_hash": "e3b0c4...",
      "input_schema": {
        "type": "object",
        "properties": {
          "service": { "type": "string" },
          "area": { "type": "string" },
          "time": { "type": "string" }
        },
        "required": ["service", "area", "time"]
      }
    }
  ]
}
```

---

## 4. `deployments`

Maps public widget endpoints and allowed website origins to an active published version.

```sql
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
    BEFORE UPDATE ON deployments
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

**Example Row:**
```json
{
  "id": "dep_fixji_live",
  "tenant_id": "tnt_fixji_001",
  "agent_id": "agt_fixji_booking",
  "public_key": "pub_fixji_9f8e7d6c5b4a",
  "name": "FixJi Main Website",
  "active_version_id": "ver_v1_2",
  "previous_version_id": "ver_v1_1",
  "allowed_origins": ["https://fixji.in", "https://www.fixji.in"],
  "widget_config": {
    "position": "bottom-right",
    "theme": "light",
    "brand_color": "#2563EB",
    "welcome_message": "Hi! How can FixJi help with your home service today?"
  },
  "is_active": true
}
```
