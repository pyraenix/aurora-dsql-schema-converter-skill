# DSQL Migration Patterns

Operational guidance for migrating from PostgreSQL to Aurora DSQL beyond schema conversion.

Sources:
- [Migration Guide](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-migration-guide.html)
- [Concurrency Control](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-concurrency-control.html)
- [Quotas and Limits](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/CHAP_quotas.html)
- [Database Roles and IAM](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/using-database-and-iam-roles.html)
- [Considerations](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/considerations.html)

---

## 1. OCC Retry Pattern

DSQL uses Optimistic Concurrency Control (OCC). Any write transaction can fail with `SQLSTATE 40001` (serialization failure). This is normal and expected — not a bug.

### Retry Strategy

```
Max retries: 5
Base delay: 50ms
Backoff: exponential with jitter
Formula: delay = min(base * 2^attempt + random(0, base), max_delay)
Max delay: 5000ms
```

### Application Code Pattern (pseudocode)

```sql
-- Pseudocode for retry wrapper
FUNCTION execute_with_retry(sql_statement, max_retries=5):
  FOR attempt IN 0..max_retries:
    BEGIN TRANSACTION
      EXECUTE sql_statement
      COMMIT
      RETURN success
    EXCEPTION WHEN sqlstate '40001':
      -- OCC conflict, retry
      delay = min(50 * power(2, attempt) + random(0, 50), 5000)
      SLEEP(delay)
    EXCEPTION WHEN OTHERS:
      -- Non-retryable error
      RAISE
  END FOR
  RAISE 'max retries exceeded'
```

### Language-Specific Examples

**Python (psycopg2):**
```python
import time, random, psycopg2

def execute_with_retry(conn_params, sql, params=None, max_retries=5):
    for attempt in range(max_retries):
        conn = psycopg2.connect(**conn_params)
        try:
            with conn.cursor() as cur:
                cur.execute(sql, params)
            conn.commit()
            return
        except psycopg2.errors.SerializationFailure:
            conn.rollback()
            delay = min(0.05 * (2 ** attempt) + random.uniform(0, 0.05), 5.0)
            time.sleep(delay)
        finally:
            conn.close()
    raise Exception("Max retries exceeded")
```

**Node.js (pg):**
```javascript
async function executeWithRetry(pool, sql, params, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      await client.query(sql, params);
      await client.query('COMMIT');
      return;
    } catch (err) {
      await client.query('ROLLBACK');
      if (err.code === '40001') {
        const delay = Math.min(50 * Math.pow(2, attempt) + Math.random() * 50, 5000);
        await new Promise(r => setTimeout(r, delay));
      } else {
        throw err;
      }
    } finally {
      client.release();
    }
  }
  throw new Error('Max retries exceeded');
}
```

### When OCC Conflicts Are Likely

- High-contention rows (counters, balances, status fields)
- Batch updates touching overlapping row sets
- Long-running transactions (more time = more conflict chance)

### Mitigation Strategies

- Keep transactions short (fewer rows, less time)
- Avoid hot rows (shard counters, use CACHE 65536 sequences)
- Use idempotent operations where possible
- Batch writes in groups of ≤100 rows to reduce conflict surface

---

## 2. Data Migration Strategy

### Constraints

- DSQL max 3,000 rows per write transaction
- No COPY command support
- No TRUNCATE (use DELETE FROM for cleanup)
- One DDL per transaction (schema must be deployed first)

### Migration Order

```
1. Deploy schema (one DDL per transaction)
   ├── Sequences first
   ├── Tables (no FKs)
   ├── Indexes (ASYNC — non-blocking)
   └── Functions
2. Migrate data (batched INSERTs)
3. Validate (row counts, spot checks)
4. Switch application connection string
```

### Batch Insert Pattern

```sql
-- Batch size: 500-1000 rows per transaction (well under 3,000 limit)
-- Use COPY FROM on PostgreSQL side to export, then batch INSERT on DSQL side

BEGIN;
INSERT INTO users (id, email, status, created_at) VALUES
  ('uuid1', 'a@b.com', 'active', '2024-01-01'),
  ('uuid2', 'c@d.com', 'active', '2024-01-02'),
  -- ... up to 500-1000 rows
  ;
COMMIT;
```

### Recommended Tooling

| Tool | Use Case |
|---|---|
| pg_dump --data-only --inserts | Export as INSERT statements |
| Custom script (Python/Node) | Batch INSERTs with retry logic |
| AWS DMS | Managed migration (check DSQL support) |
| psql \copy to CSV + custom loader | Two-step: export CSV, import with batching |

### Data Validation Queries

```sql
-- Compare row counts
SELECT 'users' AS table_name, COUNT(*) FROM users
UNION ALL
SELECT 'tickets', COUNT(*) FROM tickets
UNION ALL
SELECT 'comments', COUNT(*) FROM comments;

-- Check sequences are ahead of max IDs
SELECT last_value FROM ticket_seq;
SELECT MAX(ticket_number) FROM tickets;

-- Verify async indexes are ready
SELECT indexrelid::regclass, indisvalid
FROM pg_index
WHERE NOT indisvalid;  -- Should return 0 rows when all indexes are built
```

---

## 3. Application-Layer FK Enforcement

### Where to Call Validation Functions

The generated `validate_fk_*()` functions need to be called from your application before INSERT/UPDATE operations that reference other tables.

### Integration Patterns

**Pattern A: Middleware/Service Layer**
```python
# Before inserting a ticket
def create_ticket(org_id, reporter_id, title):
    # Validate FKs first
    with db.cursor() as cur:
        cur.execute("SELECT validate_fk_tickets_org_id(%s)", [org_id])
        if not cur.fetchone()[0]:
            raise ValueError(f"Organization {org_id} does not exist")

        cur.execute("SELECT validate_fk_tickets_reporter_id(%s)", [reporter_id])
        if not cur.fetchone()[0]:
            raise ValueError(f"User {reporter_id} does not exist")

    # Proceed with insert
    db.execute("INSERT INTO tickets (...) VALUES (...)")
```

**Pattern B: ORM Hooks (SQLAlchemy example)**
```python
from sqlalchemy import event

@event.listens_for(Ticket, 'before_insert')
def validate_ticket_fks(mapper, connection, target):
    result = connection.execute(
        text("SELECT validate_fk_tickets_org_id(:id)"),
        {"id": target.org_id}
    )
    if not result.scalar():
        raise IntegrityError("FK violation: org_id")
```

**Pattern C: Database Function Wrapper**
```sql
-- Wrap INSERT with validation in a single function
CREATE FUNCTION insert_ticket(
  p_org_id bigint,
  p_reporter_id uuid,
  p_title text
) RETURNS uuid
LANGUAGE sql AS $$
  -- This will return NULL if FK is invalid (caller checks)
  SELECT CASE
    WHEN NOT validate_fk_tickets_org_id(p_org_id) THEN NULL
    WHEN NOT validate_fk_tickets_reporter_id(p_reporter_id) THEN NULL
    ELSE (
      INSERT INTO tickets (org_id, reporter_id, title)
      VALUES (p_org_id, p_reporter_id, p_title)
      RETURNING id
    )
  END;
$$;
```

### Cascade Operations

For ON DELETE CASCADE replacement:
```python
def delete_organization(org_id):
    # Call cascade function first
    db.execute("SELECT cascade_resolve(%s)", [org_id])
    # Then delete the parent
    db.execute("DELETE FROM organizations WHERE id = %s", [org_id])
```

### Trigger Replacement Calling Points

| Original Trigger | When to Call Replacement | Where |
|---|---|---|
| BEFORE UPDATE (set_updated_at) | After every UPDATE | App layer or DB function wrapper |
| BEFORE INSERT (validate_hours) | N/A — CHECK constraint | Automatic |
| AFTER INSERT (log_change) | After INSERT/UPDATE/DELETE | App layer |
| BEFORE DELETE (cascade_resolve) | Before DELETE on parent | App layer |

---

## 4. Multi-Schema Migration

DSQL supports up to 10 schemas per database. Migration strategy depends on your source schema count.

### If ≤10 schemas: Direct Migration

```sql
-- PostgreSQL schemas migrate directly to DSQL
CREATE SCHEMA billing;
GRANT USAGE ON SCHEMA billing TO app_role;
CREATE TABLE billing.invoices (...);
CREATE TABLE billing.payments (...);

CREATE SCHEMA support;
GRANT USAGE ON SCHEMA support TO app_role;
CREATE TABLE support.tickets (...);
```

This works as-is in DSQL. No table renaming needed.

### If >10 schemas: Consolidate with Prefixes

```sql
-- PostgreSQL (>10 schemas — must consolidate)
CREATE SCHEMA analytics;   -- schema #11 — won't fit
CREATE TABLE analytics.reports (...);

-- DSQL: merge overflow schemas into existing ones using prefixes
CREATE TABLE public.analytics_reports (...);
```

### Strategy Decision Matrix

| Source Schema Count | DSQL Strategy | Effort |
|---|---|---|
| 1 (public only) | No change needed | None |
| 2-10 | Migrate schemas directly | Low (just CREATE SCHEMA + GRANT) |
| 11+ | Consolidate into ≤10, prefix overflow tables | Medium |

### What to Update When Consolidating

- All table references in application code
- All function bodies referencing schema-qualified names
- All view definitions
- Index names (prefix to avoid collisions)
- Sequence names
- GRANT statements (consolidate permissions)

### search_path Behavior

```sql
-- DSQL supports search_path
SET search_path TO billing, public;
SELECT * FROM invoices;  -- resolves to billing.invoices ✅

-- NOTE: After schema DDL, refresh connection for visibility
```

---

## 5. Pre-Flight Validation Checklist

Run through this before switching production traffic to DSQL:

### Schema Validation

- [ ] All DDL statements execute without error (one per transaction)
- [ ] All async indexes show `indisvalid = true` in `pg_index`
- [ ] Sequences are set to values beyond current max IDs
- [ ] All SQL functions execute without syntax errors

### Data Validation

- [ ] Row counts match source database (per table)
- [ ] Spot-check 10 random rows per table for data integrity
- [ ] NULL counts match for nullable columns
- [ ] Unique constraints hold (no duplicate violations during load)

### Application Validation

- [ ] All CRUD operations succeed with retry logic
- [ ] FK validation functions return correct results
- [ ] OCC retry fires and succeeds under concurrent load
- [ ] No TRUNCATE, LISTEN/NOTIFY, or temp table usage in app code
- [ ] ORDER BY results acceptable with C collation
- [ ] JSON queries work with `::jsonb` cast where needed

### Performance Validation

- [ ] Query plans use indexes (EXPLAIN shows Index Scan)
- [ ] Batch operations stay under 3,000 rows per transaction
- [ ] Sequence nextval() performs acceptably under load
- [ ] No serialization failures under normal (non-contention) load

### Rollback Plan

- [ ] Source PostgreSQL database remains available during cutover
- [ ] Connection string switch is reversible (feature flag or config)
- [ ] Data written to DSQL during validation can be discarded
