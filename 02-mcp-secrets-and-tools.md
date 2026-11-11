# 02 — Vault Secret References, MCP Servers & Tool Catalogue

**Tables:** `secret_references`, `mcp_servers`, `mcp_tools`  
**Blueprint Alignment:** Sections 5.2, 5.3, 12, 13 · FixJi Workflow Step 2 · Prototype Page 12

---

## Design Decisions

1. **Zero Secret Material in PostgreSQL (`secret_references`):**
   - Plaintext tokens (like `fxj_live_password_999`) are written immediately to **Azure Key Vault** under a deterministic tenant-prefixed secret name (`relayai-tnt-fixji-001-<id>`), never stored in PostgreSQL.
   - `secret_references` tracks metadata (`key_vault_secret_name`, `secret_version`, `rotated_at`, `created_by_user_id`) so rotation and deletion remain auditable without exposing credentials.
2. **Strict Auth-to-Secret Integrity (`mcp_servers`):**
   - A `CHECK` constraint enforces that any server with `auth_type <> 'NONE'` MUST reference a valid `secret_references` row belonging to the **same `tenant_id`** via composite FK `(tenant_id, secret_ref_id)`.
   - `request_timeout_ms` and `max_response_bytes` live per MCP server (with hard platform ceilings) to satisfy Blueprint Section 13 (*"output size limits and execution timeouts"*).
3. **Schema Drift & Disappearance Handling (`mcp_tools`):**
   - Re-running **Discover Tools** computes `schema_hash` (SHA-256 of canonical `input_schema`).
   - If FixJi removes or renames a tool on its MCP server, `is_active` flips to `FALSE` instead of deleting the row—preserving historical `tool_executions`, `approvals`, and `agent_versions` foreign keys while warning admins in the UI.
4. **Canonical Tool Identity (`mcp_tool_id`):**
   - `UNIQUE (mcp_server_id, tool_name)` allows FixJi to have multiple MCP servers (`Catalog MCP`, `Availability MCP`, `Booking MCP`) without `tool_name` collisions corrupting agent permissions.

---

## 1. `secret_references`

Pointer table mapping tenant secrets to Azure Key Vault entries (also supports LLM provider BYOK references if enabled later).

```sql
CREATE TABLE secret_references (
    id                    VARCHAR(50) PRIMARY KEY,
    tenant_id             VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    scope                 VARCHAR(30) NOT NULL DEFAULT 'MCP_AUTH'
                          CHECK (scope IN ('MCP_AUTH', 'LLM_PROVIDER', 'WEBHOOK')),
    name                  VARCHAR(100) NOT NULL,
    key_vault_secret_name VARCHAR(255) NOT NULL,
    secret_version        VARCHAR(100),
    created_by_user_id    UUID REFERENCES users(id),
    rotated_at            TIMESTAMPTZ,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (key_vault_secret_name)
);
```

| Column | Type | Purpose |
| --- | --- | --- |
| `scope` | `VARCHAR(30)` | Classifies secret purpose (`MCP_AUTH`, `LLM_PROVIDER`, `WEBHOOK`) |
| `key_vault_secret_name` | `VARCHAR(255)` | Deterministic vault key (e.g., `relayai-tnt-fixji-001-sec-01`) |
| `secret_version` | `VARCHAR(100)` | Pinned or latest Key Vault version ID for rotation tracking |
| `rotated_at` | `TIMESTAMPTZ` | Timestamp of last credential rotation (displayed in admin activity) |

**Example Row:**
```json
{
  "id": "sec_fixji_01",
  "tenant_id": "tnt_fixji_001",
  "scope": "MCP_AUTH",
  "name": "FixJi Booking MCP Token",
  "key_vault_secret_name": "relayai-tnt-fixji-001-sec-fixji-01",
  "secret_version": "a1b2c3d4e5f6",
  "rotated_at": "2026-11-09T08:10:00Z"
}
```

---

## 2. `mcp_servers`

Stores tenant-registered external MCP endpoints and execution guardrails.

```sql
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
    BEFORE UPDATE ON mcp_servers
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### Security & SSRF Enforcement Notes (Application + Connector Layer)
- While PostgreSQL checks basic URL syntax (`^https?://`), the **MCP Connector** must enforce **HTTPS-only in production** and resolve DNS + block private/loopback/link-local CIDRs (`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `::1`, `fc00::/7`, `fe80::/10`) **at socket connect time** on both `discover` and runtime tool calls (prevents DNS rebinding & Azure IMDS SSRF).

**Example Row:**
```json
{
  "id": "srv_01",
  "tenant_id": "tnt_fixji_001",
  "name": "FixJi Booking MCP",
  "endpoint_url": "https://mcp.fixji.in/mcp",
  "transport": "STREAMABLE_HTTP",
  "auth_type": "BEARER_TOKEN",
  "secret_ref_id": "sec_fixji_01",
  "request_timeout_ms": 15000,
  "max_response_bytes": 262144,
  "status": "CONNECTED"
}
```

---

## 3. `mcp_tools`

Discovered tool catalogue populated by the MCP Connector (`list_tools`) and displayed in the **Tool Catalogue** UI (Blueprint Page 12).

```sql
CREATE TABLE mcp_tools (
    id                   VARCHAR(50) PRIMARY KEY,
    tenant_id            VARCHAR(50) NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    mcp_server_id        VARCHAR(50) NOT NULL,
    tool_name            VARCHAR(150) NOT NULL,
    description          TEXT,
    input_schema         JSONB NOT NULL,
    schema_hash          VARCHAR(64) NOT NULL,
    operation_type       VARCHAR(20) NOT NULL DEFAULT 'READ'
                         CHECK (operation_type IN ('READ', 'WRITE', 'DESTRUCTIVE')),
    default_requires_approval BOOLEAN NOT NULL DEFAULT FALSE,
    is_active            BOOLEAN NOT NULL DEFAULT TRUE,
    last_synced_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, id),
    UNIQUE (mcp_server_id, tool_name),
    FOREIGN KEY (tenant_id, mcp_server_id) REFERENCES mcp_servers(tenant_id, id) ON DELETE CASCADE
);

CREATE INDEX idx_mcp_tools_tenant_active ON mcp_tools(tenant_id, mcp_server_id) WHERE is_active = TRUE;
```

| Column | Type | Purpose |
| --- | --- | --- |
| `operation_type` | `VARCHAR(20)` | Maps directly to Page 12 Tool Catalogue (`Read`, `Write`, `Destructive`) |
| `default_requires_approval` | `BOOLEAN` | Sensible safety default when assigning a `WRITE`/`DESTRUCTIVE` tool to an agent |
| `schema_hash` | `VARCHAR(64)` | SHA-256 hash of `input_schema`; alerts admins if published `agent_versions` drifted |
| `is_active` | `BOOLEAN` | Set to `FALSE` if a tool disappears on subsequent `discover` syncs |

**Example Row:**
```json
{
  "id": "tl_3",
  "tenant_id": "tnt_fixji_001",
  "mcp_server_id": "srv_01",
  "tool_name": "create_booking",
  "description": "Create a service booking in FixJi database",
  "input_schema": {
    "type": "object",
    "properties": {
      "service": { "type": "string" },
      "area": { "type": "string" },
      "time": { "type": "string" }
    },
    "required": ["service", "area", "time"],
    "additionalProperties": false
  },
  "schema_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "operation_type": "WRITE",
  "default_requires_approval": true,
  "is_active": true
}
```
