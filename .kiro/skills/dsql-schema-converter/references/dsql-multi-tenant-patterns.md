# Multi-Tenant Schema Patterns for DSQL

How to design and convert multi-tenant PostgreSQL schemas for Aurora DSQL.

Sources:
- [DSQL SQL Dialect Blog](https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/)
- [Migration Guide](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-migration-guide.html)
- [Quotas and Limits](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/CHAP_quotas.html)

---

## Why Multi-Tenancy Matters for DSQL

PostgreSQL multi-tenant apps typically rely on:
- Row-Level Security (RLS) for tenant isolation → **Not supported in DSQL**
- Foreign keys for data integrity → **Not supported in DSQL**
- Schema-per-tenant isolation → **Limited to 10 schemas in DSQL**
- Partitioning by tenant → **Not supported in DSQL**

All of these must be replaced with application-layer patterns during conversion.

---

## Isolation Strategies

### Strategy 1: Shared Tables with tenant_id Column (Recommended)

Best for most SaaS applications. All tenants share tables, isolated by `tenant_id`.

```sql
-- Every table MUST include tenant_id
CREATE TABLE orders (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,          -- REQUIRED for isolation
    customer_id uuid NOT NULL,
    total numeric(10,2),
    status varchar(20) COLLATE "C" DEFAULT 'pending',
    created_at timestamptz DEFAULT now()
);

-- Composite index with tenant_id FIRST (critical for DSQL's PK-ordered storage)
CREATE INDEX ASYNC idx_orders_tenant ON orders (tenant_id, created_at DESC);
CREATE INDEX ASYNC idx_orders_tenant_status ON orders (tenant_id, status);
```

**Rules:**
- `tenant_id` in EVERY table (no exceptions)
- `tenant_id` as FIRST column in composite indexes
- Application MUST include `tenant_id` in every WHERE clause
- Application MUST validate tenant_id on every request (no database-level enforcement)

### Strategy 2: Schema-Per-Tenant (Limited)

Possible for ≤10 tenants. Each tenant gets their own schema.

```sql
-- Max 10 schemas per database
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;

GRANT USAGE ON SCHEMA tenant_acme TO acme_role;
GRANT USAGE ON SCHEMA tenant_globex TO globex_role;

CREATE TABLE tenant_acme.orders (...);
CREATE TABLE tenant_globex.orders (...);
```

**Limitations:**
- Hard limit of 10 schemas per database
- 1,000 tables total across ALL schemas
- Not viable for SaaS with many tenants
- Use only for enterprise customers with strict isolation requirements

### Strategy 3: Cluster-Per-Tenant (Maximum Isolation)

Each tenant gets their own DSQL cluster. Maximum isolation but highest cost.

- 20 single-region clusters per account (increasable)
- 5 multi-region clusters per account (increasable)
- Complete data isolation
- Independent scaling
- Use for regulated industries (healthcare, finance) requiring physical separation

---

## Converting PostgreSQL Multi-Tenant Schemas

### RLS Replacement

```sql
-- PostgreSQL (Row-Level Security)
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON orders
    USING (tenant_id = current_setting('app.current_tenant')::uuid);

-- DSQL: RLS not supported. Enforce in application layer.
-- Every query MUST include tenant_id filter:
SELECT * FROM orders WHERE tenant_id = $1 AND status = 'pending';

-- NEVER allow queries without tenant_id:
-- BAD:  SELECT * FROM orders WHERE status = 'pending';
-- GOOD: SELECT * FROM orders WHERE tenant_id = $1 AND status = 'pending';
```

### FK Validation with Tenant Scope

```sql
-- Validate FK within same tenant (cross-tenant references are invalid)
CREATE FUNCTION validate_fk_orders_customer(p_tenant_id uuid, p_customer_id uuid)
RETURNS boolean
LANGUAGE sql AS $$
    SELECT EXISTS (
        SELECT 1 FROM customers
        WHERE id = p_customer_id AND tenant_id = p_tenant_id
    );
$$;
```

### Unique Constraints Scoped to Tenant

```sql
-- PostgreSQL: unique email per tenant
CREATE UNIQUE INDEX idx_users_email ON users (tenant_id, email);

-- DSQL: same (works directly)
CREATE UNIQUE INDEX ASYNC idx_users_email ON users (tenant_id, email);
```

---

## Primary Key Design for Multi-Tenant

DSQL stores data in PK order. For multi-tenant, this affects query performance:

### Option A: UUID PK (distributed writes)

```sql
CREATE TABLE events (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id uuid NOT NULL,
    ...
);
-- Writes distributed randomly across storage
-- Tenant queries need index on (tenant_id, ...)
```

### Option B: Composite PK with tenant_id (co-located reads)

```sql
CREATE TABLE events (
    tenant_id uuid,
    event_id uuid DEFAULT gen_random_uuid(),
    ...
    PRIMARY KEY (tenant_id, event_id)
);
-- All data for one tenant is physically co-located
-- Range scans within a tenant are sequential reads
-- Best for tenant-scoped queries
```

**Recommendation:** Use Option B (composite PK) when:
- Most queries are scoped to a single tenant
- You need efficient range scans within a tenant
- Tenant data size is manageable (not billions of rows per tenant)

Use Option A (UUID PK) when:
- Cross-tenant queries are common (admin dashboards)
- Write distribution is more important than read locality
- You need simple single-column PK references

---

## Index Design for Multi-Tenant

```sql
-- ALWAYS put tenant_id first in composite indexes
CREATE INDEX ASYNC idx_orders_tenant_date ON orders (tenant_id, created_at DESC);
CREATE INDEX ASYNC idx_orders_tenant_status ON orders (tenant_id, status);

-- Covering index for common tenant queries (avoids storage round-trip)
CREATE INDEX ASYNC idx_orders_tenant_summary ON orders (tenant_id, status)
    INCLUDE (total, created_at);

-- Cross-tenant index (for admin/analytics only)
CREATE INDEX ASYNC idx_orders_status_global ON orders (status, created_at DESC);
```

---

## Application-Layer Tenant Enforcement

### Middleware Pattern (all frameworks)

```
Every request:
1. Extract tenant_id from auth token / session / header
2. Validate tenant_id exists and user has access
3. Inject tenant_id into all database queries
4. NEVER trust client-supplied tenant_id without validation
```

### Django Example

```python
class TenantMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        request.tenant_id = self.get_tenant_from_token(request)
        if not request.tenant_id:
            return HttpResponse(status=401)
        return self.get_response(request)

# In views/services:
def list_orders(request):
    return Order.objects.filter(tenant_id=request.tenant_id)
```

### Query Safety Rules

| Rule | Rationale |
|---|---|
| ALWAYS include `WHERE tenant_id = $1` | No RLS to enforce isolation |
| NEVER allow `SELECT * FROM table` without tenant filter | Data leak risk |
| Validate tenant_id server-side before query | Client can't be trusted |
| Use parameterized queries | SQL injection prevention |
| Log tenant_id with every query | Audit trail |

---

## Conversion Checklist for Multi-Tenant Schemas

- [ ] Add `tenant_id` column to every table (if not already present)
- [ ] Remove all RLS policies (not supported)
- [ ] Remove all FK constraints (use validate_fk with tenant scope)
- [ ] Add composite indexes with `tenant_id` first
- [ ] Consider composite PK `(tenant_id, id)` for tenant-scoped tables
- [ ] Implement tenant validation in application middleware
- [ ] Ensure all queries include `tenant_id` in WHERE clause
- [ ] Add tenant-scoped unique constraints where needed
- [ ] Remove schema-per-tenant if >10 tenants (use shared tables)
- [ ] Remove PARTITION BY tenant (not supported; use indexes instead)
- [ ] Test cross-tenant data isolation at application layer
