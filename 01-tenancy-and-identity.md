# 01 — Tenancy, Identity, RBAC & Invitations

**Tables:** `tenants`, `users`, `tenant_members`, `tenant_invitations`  
**Blueprint Alignment:** Sections 3, 5.1, 12, 13 · FixJi Workflow Steps 1A & 1B

---

## Design Decisions

1. **Separated Invitations (`tenant_invitations`) Instead of Admin-Set Passwords:**
   - When a Platform Admin creates `FixJi` (Step 1A) or the FixJi Owner invites `Karthik (dev@fixji.in)` and `Suresh (manager@fixji.in)` (Step 1B), RelayAI issues a cryptographically random invite token (storing only its SHA-256 `token_hash`) rather than assigning passwords on behalf of users.
   - If an invited email already exists in `users` (e.g., an agency developer working across both `FixJi` and `ShopNow`), accepting the invitation simply links their existing `user_id` into `tenant_members` without failing on `users.email UNIQUE`.
2. **Exact Blueprint Roles:**
   Enforced via `CHECK` constraint across both `tenant_members` and `tenant_invitations`: `OWNER`, `ADMIN`, `DEVELOPER`, `ANALYST`, `APPROVER`.
3. **One Active Membership Row per `(tenant_id, user_id)`:**
   `UNIQUE (tenant_id, user_id)` prevents conflicting duplicate roles for the same user inside one tenant.
4. **Soft-Deactivation & Quota Controls on `tenants`:**
   `rate_limit_rpm`, `max_concurrency`, and `monthly_token_quota` live directly on `tenants` (cached in Redis with short TTL) so Gateway fairness checks run in $O(1)$ without hardcoding plan numbers.

---

## 1. `tenants`

Root isolation entity for every customer organization (`FixJi`, `ShopNow`/`ShopSphere`, `HealthPlus`/`CarePoint`, etc.).

```sql
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
    BEFORE UPDATE ON tenants
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

| Column | Type | Purpose |
| --- | --- | --- |
| `id` | `VARCHAR(50)` | Stable tenant identifier (e.g. `tnt_fixji_001`) |
| `slug` | `VARCHAR(100)` | URL-safe unique tenant identifier (`fixji`) |
| `plan_tier` | `VARCHAR(30)` | Subscription tier (`STARTER`, `GROWTH`, `ENTERPRISE`) |
| `status` | `VARCHAR(20)` | Gateway refuses traffic when not `ACTIVE` |
| `rate_limit_rpm` | `INT` | Per-tenant requests-per-minute limit enforced in Redis |
| `max_concurrency` | `INT` | Max simultaneous in-flight agent runs for this tenant |
| `monthly_token_quota` | `BIGINT` | Monthly LLM token cap across all tenant deployments |
| `data_retention_days` | `INT` | Controls conversation/log retention policy (Blueprint Section 19) |

**Example Row:**
```json
{
  "id": "tnt_fixji_001",
  "name": "FixJi",
  "slug": "fixji",
  "business_type": "Home services",
  "plan_tier": "GROWTH",
  "status": "ACTIVE",
  "rate_limit_rpm": 100,
  "max_concurrency": 20,
  "monthly_token_quota": 5000000,
  "data_retention_days": 90
}
```

---

## 2. `users`

Global user identities covering both RelayAI internal operators (`is_platform_admin = true`) and customer workspace members.

```sql
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
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

| Column | Type | Purpose |
| --- | --- | --- |
| `email` | `VARCHAR(255)` | Normalized lowercase email address (`CHECK (email = LOWER(email))`) |
| `password_hash` | `TEXT` | Argon2id/bcrypt hash (`NULL` for SSO-only accounts) |
| `external_auth_sub` | `VARCHAR(255)` | Optional Microsoft Entra ID / OIDC subject identifier |
| `is_platform_admin` | `BOOLEAN` | Grants access to Platform Admin Dashboard APIs only |
| `is_active` | `BOOLEAN` | Global account kill-switch |

**Example Row:**
```json
{
  "id": "a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11",
  "email": "founder@fixji.in",
  "full_name": "FixJi Founder",
  "is_platform_admin": false,
  "is_active": true
}
```

---

## 3. `tenant_members`

Maps users to customer workspaces and assigns exact RBAC permissions.

```sql
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
    BEFORE UPDATE ON tenant_members
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### Role Permission Matrix

| Role | Manage Team / Billing | Connect MCP / Vault | Author & Publish Agents | Approve HITL Actions | View Analytics / Traces |
| --- | --- | --- | --- | --- | --- |
| `OWNER` | Yes | Yes | Yes | Yes | Yes |
| `ADMIN` | Yes | Yes | Yes | Yes | Yes |
| `DEVELOPER` | No | Yes | Yes | No | Yes |
| `APPROVER` | No | No | No | Yes | Yes |
| `ANALYST` | No | No | Read-only | No | Yes |

---

## 4. `tenant_invitations`

Tracks pending workspace invitations with expiring cryptographic tokens.

```sql
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
    ON tenant_invitations(tenant_id, email)
    WHERE status = 'PENDING';
```

**Why `idx_tenant_invites_pending_email` Matters:** Prevents sending duplicate simultaneous pending invites for `dev@fixji.in` within the same workspace while still preserving historical accepted/revoked invites for auditability.
