---
name: dsql-schema-converter
description: "Convert PostgreSQL schemas to Aurora DSQL-compatible DDL — type mappings, PL/pgSQL transpilation, FK validation functions, index conversion, OCC retry patterns, multi-region design, ORM migration guides. Triggers on phrases like: convert to DSQL, migrate PostgreSQL to DSQL, DSQL schema, DSQL compatibility, PostgreSQL to DSQL, DSQL type mapping, PL/pgSQL to SQL, DSQL foreign key, DSQL index async, DSQL JSONB, DSQL ENUM, DSQL sequence CACHE, DSQL OCC retry, DSQL multi-region, DSQL Django, DSQL Hibernate, DSQL Rails, DSQL ALTER TABLE."
license: Apache-2.0
---

# DSQL Schema Converter — Companion Skill

## Relationship to the Power

This skill provides conversion knowledge and guidance. For actual schema conversion execution, use the dsql-schema-converter MCP power tools (`convert_schema`, `analyze_compatibility`, `list_type_mappings`, `list_supported_dialects`).

The Power handles execution. The Skill handles knowledge. Together they give Kiro the ability to both explain and act.

## Reference Files

Load these files as needed for detailed guidance:

### MCP:

#### [mcp-setup.md](mcp/mcp-setup.md)
When: Load when user asks about automated conversion tools or MCP integration
Contains: Planned MCP tools (convert_schema, analyze_compatibility), configuration, skill+power architecture

### [postgresql-type-mappings.md](references/postgresql-type-mappings.md)
When: ALWAYS load before converting data types or answering type questions
Contains: Two-stage mapping pipeline (PG → Normalized → DSQL), storage sizes, indexability, NUMERIC defaults, JSON/JSONB behavior, array/INET runtime patterns

### [dsql-constraints.md](references/dsql-constraints.md)
When: ALWAYS load before recommending schema designs or answering "can DSQL do X?"
Contains: Transaction limits, schema limits, database object limits, PK design guidance, SELECT FOR UPDATE behavior, roles/IAM, collation, sequence CACHE rules

### [plpgsql-patterns.md](references/plpgsql-patterns.md)
When: MUST load when converting triggers or PL/pgSQL functions
Contains: 10 transpilation patterns with before/after code, app-responsibility notes

### [dsql-alter-table-matrix.md](references/dsql-alter-table-matrix.md)
When: MUST load when user asks about ALTER TABLE, DROP COLUMN, or schema evolution
Contains: Full support matrix, workaround patterns for unsupported operations

### [dsql-migration-patterns.md](references/dsql-migration-patterns.md)
When: Load when planning data migration, implementing OCC retry, or wiring FK validation
Contains: OCC retry code (Python, Node.js), batching strategy, FK enforcement patterns, pre-flight checklist

### [dsql-multi-region.md](references/dsql-multi-region.md)
When: Load when user asks about multi-region, active-active, or high availability
Contains: Architecture, setup, performance considerations, geographic partitioning, monitoring

### [dsql-multi-tenant-patterns.md](references/dsql-multi-tenant-patterns.md)
When: MUST load when converting multi-tenant schemas or when user mentions tenant isolation
Contains: Isolation strategies, RLS replacement, tenant-scoped FK validation, PK design for multi-tenant, index patterns, application-layer enforcement

### [dsql-function-compatibility.md](references/dsql-function-compatibility.md)
When: Load when user asks "does function X work in DSQL?" or encounters function errors
Contains: Full function compatibility matrix (supported, partial, unsupported), maintenance commands, transaction control

### ORM Guides:

#### [orm-django.md](references/orm-django.md)
When: Load when user is migrating a Django application to DSQL
Contains: aurora-dsql-django adapter setup, model changes, migration patterns, OCC retry middleware

#### [orm-hibernate.md](references/orm-hibernate.md)
When: Load when user is migrating a Java/Spring Boot application to DSQL
Contains: Hibernate dialect, HikariCP config, entity changes, Spring Retry, Liquibase patterns

#### [orm-rails.md](references/orm-rails.md)
When: Load when user is migrating a Ruby on Rails application to DSQL
Contains: IAM token initializer, model associations without FK, async indexes, OCC retry concern

### [extending-dialects.md](references/extending-dialects.md)
When: Load when user asks about adding MySQL, Spanner, or other source dialect support
Contains: Architecture for adding new dialects, NormalizedType inventory, custom type resolvers

### [REFERENCES.md](references/REFERENCES.md)
When: Load when verifying source documentation or checking for outdated information
Contains: All official AWS documentation links mapped to skill files

## Overview

The DSQL Schema Converter converts PostgreSQL database schemas to Aurora DSQL-compatible DDL. It handles:

- **Tables & Types**: Maps all PostgreSQL types to DSQL equivalents (20 NormalizedTypes)
- **ENUM → CHECK**: Converts custom ENUM types to CHECK constraints with allowed values
- **Foreign Keys → SQL Functions**: Generates `validate_fk_*()` functions for app-layer enforcement
- **PL/pgSQL Transpilation**: 10 recognized patterns converted to pure SQL functions
- **Triggers → SQL Functions**: Generates callable replacement functions
- **Indexes**: All converted to CREATE INDEX ASYNC, GIN/GiST → btree, partial WHERE removed, INCLUDE preserved
- **Sequences**: Preserved natively with CACHE 1 (DSQL supports CREATE SEQUENCE)
- **Identity Columns**: GENERATED BY DEFAULT/ALWAYS AS IDENTITY supported natively
- **Views**: Regular views preserved, materialized views demoted to regular views
- **Computed Columns**: GENERATED ALWAYS AS (expr) STORED preserved (DSQL supports this)
- **Multi-Schema**: Supported (up to 10 schemas); consolidate with prefixes if >10
- **Roles/Permissions**: CREATE ROLE and GRANT/REVOKE supported (linked to IAM)
- **OCC Retry Patterns**: Guidance for handling SQLSTATE 40001 serialization failures
- **Data Migration**: Batching strategies for the 3,000-row transaction limit
- **Function Compatibility**: Reference for which PostgreSQL functions work in DSQL

## Prerequisites

- **Node.js 18+** required for the MCP server
- **dsql-schema-converter MCP Power** must be installed and running
- **dsql-lint** (recommended): `pip install dsql-lint` — provides proper SQL parsing and validation in the production pipeline

MCP configuration (in `.kiro/settings/mcp.json`):
```json
{
  "mcpServers": {
    "dsql-schema-converter": {
      "command": "node",
      "args": ["mcp-server/src/index.js"]
    }
  }
}
```

## Conversion Rules Summary

### Data Types

| PostgreSQL | DSQL | Notes |
|---|---|---|
| SMALLINT, INT2 | smallint | Direct |
| INTEGER, INT, INT4 | integer | Direct |
| BIGINT, INT8 | bigint | Direct |
| REAL, FLOAT4 | real | Direct |
| DOUBLE PRECISION, FLOAT8 | double precision | Direct |
| NUMERIC(p,s) | numeric(p,s) | Precision preserved (max 38,37) |
| SERIAL/BIGSERIAL | integer/bigint | Auto-increment removed |
| CHAR(n), VARCHAR(n) | char(n), varchar(n) | Length preserved. COLLATE "C" added. |
| TEXT | text | COLLATE "C" added. |
| TIMETZ | time with time zone | Direct |
| INTERVAL | interval | Not indexable |
| BOOLEAN | boolean | Direct |
| BYTEA | bytea | Not indexable |
| UUID | uuid | Direct |
| JSON | json | DSQL supports json natively |
| JSONB | json | Stored as json. Use `::jsonb` in queries. |
| TEXT[], INT[] | text | Arrays are runtime-only |
| INET, CIDR, MACADDR | text | Runtime-only in DSQL |
| TSVECTOR, TSQUERY | text | No DSQL equivalent |
| Geometric types | text | No DSQL equivalent |

### Indexes

- All indexes: `CREATE INDEX ASYNC`
- GIN/GiST/BRIN: → btree
- Partial indexes (WHERE): WHERE clause removed
- INCLUDE columns: preserved
- Unique indexes: preserved

### Schema Objects

- Foreign keys → `validate_fk_*()` SQL function
- ENUM types → CHECK constraint
- Sequences → preserved with CACHE 1 (or CACHE 65536 for high-throughput)
- Identity columns → preserved (GENERATED BY DEFAULT/ALWAYS AS IDENTITY)
- Temporary tables → regular with `_tmp_` prefix
- Partitioned tables → flat (PARTITION BY removed)
- Inherited tables → flat (INHERITS removed)
- Materialized views → regular VIEW
- GENERATED ALWAYS AS STORED → preserved
- Multiple schemas → preserved if ≤10; prefix table names if >10
- CREATE ROLE / GRANT / REVOKE → preserved (linked to IAM)
- CREATE DOMAIN → preserved
- Extensions → dropped (alternatives noted)

## PL/pgSQL Transpilation Patterns

### Pattern 1: SET_COLUMN
**Before:** `NEW.updated_at = now(); RETURN NEW;`
**After:** `UPDATE table SET updated_at = now() WHERE id = p_id;`

### Pattern 2: VALIDATION → CHECK
**Before:** `IF NEW.price < 0 THEN RAISE EXCEPTION '...'; END IF;`
**After:** `ALTER TABLE t ADD CONSTRAINT chk CHECK (price >= 0);`

### Pattern 3: AUDIT_INSERT
**Before:** `INSERT INTO audit_log (action, details) VALUES (TG_OP, row_to_json(NEW));`
**After:** SQL function with parameterized INSERT

### Pattern 4: CASCADE_DML
**Before:** `UPDATE orders SET status = 'cancelled' WHERE user_id = OLD.id;`
**After:** SQL function: `cascade_fn(p_id bigint)`

### Pattern 5: FOR_LOOP → Set-Based
**Before:** `FOR r IN SELECT... LOOP UPDATE... END LOOP;`
**After:** `UPDATE t SET... FROM (SELECT...) AS _src WHERE t.id = _src.id;`

### Pattern 6: IF_ELSE → CASE WHEN
**Before:** `IF cond THEN RETURN 'x'; ELSE RETURN 'y'; END IF;`
**After:** `SELECT CASE WHEN cond THEN 'x' ELSE 'y' END;`

### Pattern 7: EXCEPTION unique_violation → ON CONFLICT
**Before:** `INSERT... EXCEPTION WHEN unique_violation THEN UPDATE...`
**After:** `INSERT... ON CONFLICT DO NOTHING;`

### Pattern 8: Dynamic SQL → Expanded
**Before:** `EXECUTE format('DELETE FROM %I WHERE...', tbl);`
**After:** One concrete SQL function per target table

### Pattern 9: CURSOR → Set-Based
**Before:** `DECLARE cur CURSOR FOR SELECT... LOOP INSERT... END LOOP;`
**After:** `INSERT INTO t SELECT... FROM source;`

### Pattern 10: EXCEPTION no_data_found → COALESCE
**Before:** `SELECT INTO result... EXCEPTION WHEN no_data_found THEN RETURN NULL;`
**After:** `SELECT COALESCE((SELECT...), NULL);`

### Unconvertible (stubs generated)
- **PERFORM**: No SQL equivalent
- **Complex ELSIF** (3+ branches): Too complex for automation

## Workflows

### Schema Conversion
1. `convert_schema` with `file_path` or `sql`, `source_dialect: "postgresql"`
2. Review conversion report
3. Run each DDL in its own transaction

### Compatibility Check
1. `analyze_compatibility` — report only, no DDL generated
2. Estimate migration effort from the summary

### CI/CD Validation
1. Run `convert_schema` in pipeline
2. Fail if `summary.functions_stubbed > 0`

### Type Mapping Lookup
1. `list_type_mappings` with `source_dialect: "postgresql"`

### Full Migration (Schema + Data)
1. Convert schema using this skill
2. Deploy DDL to DSQL (one statement per transaction)
3. Wait for async indexes to become valid
4. Migrate data in batches (≤1000 rows per transaction, with OCC retry)
5. Validate row counts and spot-check data
6. Update application code (FK validation calls, trigger replacement calls, retry logic)
7. Switch connection string

## Identity Columns (Preferred over SERIAL)

DSQL supports identity columns natively. Prefer these over SERIAL + explicit sequence:

```sql
-- SERIAL (old way — still works)
CREATE SEQUENCE users_id_seq CACHE 1;
CREATE TABLE users (id bigint DEFAULT nextval('users_id_seq'), ...);

-- Identity column (preferred for new DSQL tables)
CREATE TABLE users (id bigint GENERATED BY DEFAULT AS IDENTITY (CACHE 1), ...);

-- Strict identity (prevents manual ID insertion)
CREATE TABLE audit (id bigint GENERATED ALWAYS AS IDENTITY (CACHE 65536), ...);
```

**When to use which:**
- `GENERATED BY DEFAULT` — migrating existing data (allows manual INSERT of known IDs)
- `GENERATED ALWAYS` — new tables (strict auto-generation, no manual override)
- Explicit sequence — when multiple tables share a sequence or you need setval()

## Multi-Schema Handling

DSQL supports up to 10 schemas per database (not just `public`). Admin users create schemas and grant access:

```sql
-- DSQL supports CREATE SCHEMA (up to 10)
CREATE SCHEMA billing;
GRANT USAGE ON SCHEMA billing TO app_role;

CREATE TABLE billing.invoices (id bigint, ...);
```

**Migration notes:**
- If your PostgreSQL database uses ≤10 schemas, they can migrate directly
- If >10 schemas, consolidate or use table name prefixing for overflow
- Refresh connections after schema DDL for immediate visibility
- Non-admin users create objects in user-created schemas

## OCC Retry (Critical for Production)

Every write to DSQL can fail with SQLSTATE 40001 (serialization failure). This is normal OCC behavior.

**Required pattern:**
```
retry(max=5, backoff=exponential, base=50ms, max_delay=5s):
  BEGIN
  ... DML ...
  COMMIT
on 40001: retry
on other: raise
```

See `references/dsql-migration-patterns.md` for language-specific implementations.

## Collation Behavior (C Only)

DSQL uses C collation exclusively. Key impacts:
- `ORDER BY text` sorts by byte value (uppercase before lowercase, accented after ASCII)
- `LIKE 'abc%'` works correctly for ASCII
- No locale-aware sorting (ä sorts after z, not after a)
- Use `lower(col)` for case-insensitive comparisons

If your application relies on locale-aware sorting, sort in the application layer.

## ALTER TABLE Limitations (Critical Post-Deployment)

DSQL's ALTER TABLE support is limited compared to PostgreSQL. Key gaps:

**Supported:**
- ADD COLUMN (with data type and STORAGE clause)
- RENAME COLUMN / RENAME TABLE / RENAME CONSTRAINT
- SET SCHEMA (move table between schemas)
- OWNER TO
- ALTER COLUMN identity options (SET GENERATED, RESTART, DROP IDENTITY)
- ADD UNIQUE constraint via existing VALID index

**NOT Supported:**
- DROP COLUMN
- ALTER COLUMN SET DATA TYPE (change column type)
- ALTER COLUMN SET/DROP DEFAULT
- ALTER COLUMN SET/DROP NOT NULL
- ADD CHECK constraint (must be at CREATE TABLE time)
- DROP CONSTRAINT

**Impact:** Design your schema carefully upfront. Column types, NOT NULL, CHECK constraints, and defaults are permanent once the table is created. See `references/dsql-alter-table-matrix.md` for workaround patterns.

## Multi-Region (Active-Active)

DSQL's key differentiator: active-active writes across 2 regions with strong consistency.

**Key facts:**
- 99.999% availability (vs 99.99% single-region)
- Both regions handle reads AND writes simultaneously
- Synchronous replication (not eventual consistency)
- Same schema automatically in both regions — deploy DDL once
- OCC conflicts work the same way cross-region (SQLSTATE 40001)
- Cross-region writes add ~50-100ms latency vs single-region

**Design for multi-region:**
- Partition data by geography to minimize cross-region conflicts
- Use UUIDs for PKs (random distribution in both regions)
- Keep transactions short (less conflict window)
- Read-only transactions have zero commit latency

See `references/dsql-multi-region.md` for architecture, setup, and optimization patterns.

## ORM Migration Guides

Framework-specific guides for running ORMs against DSQL:

| Framework | Guide | Key Adapter |
|---|---|---|
| Django | `references/orm-django.md` | `aurora-dsql-django` (PyPI) |
| Hibernate / Spring Boot | `references/orm-hibernate.md` | `aurora-dsql-hibernate` (Maven) |
| Ruby on Rails | `references/orm-rails.md` | `aws-sdk-dsql` + custom initializer |

**Also supported** (see [official samples](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/aws-sdks.html)):
- SQLAlchemy (Python)
- Tortoise ORM (Python)
- TypeORM (TypeScript)
- Sequelize (TypeScript)
- Prisma (TypeScript)
- Liquibase (Java)

## Best Practices

- Use `gen_random_uuid()` for primary keys (no extension needed) — avoids hot spots
- One DDL per transaction (DSQL requirement)
- Call FK validation functions before INSERT/UPDATE
- Use CACHE 1 for low-volume sequences, CACHE 65536 for high-throughput
- Use `json` type for JSON data (not TEXT)
- Prefer identity columns over SERIAL for new tables
- Always implement OCC retry logic (SQLSTATE 40001)
- Batch data loads at ≤1000 rows per transaction (leaves headroom under 3,000 limit)
- Keep transactions under 5 minutes and 10 MiB
- Check async index status before relying on index performance
- Test ORDER BY results with C collation — may differ from PostgreSQL locale
- Replace uuid_generate_v4() with gen_random_uuid()
- Replace lastval() with currval('explicit_sequence_name')
- Use INCLUDE columns on indexes aggressively (covering indexes avoid storage round-trips)
- Avoid sequential/timestamp-based primary keys on high-write tables
- Use EXPLAIN ANALYZE VERBOSE to identify operations not pushed to storage
- Remove VACUUM/ANALYZE from maintenance scripts (DSQL handles automatically)
- Design idempotent transactions for OCC retry safety
- Always review the conversion report

## Common Pitfalls

- No multi-statement DDL transactions
- No TRUNCATE (use DELETE FROM)
- No COPY command (use INSERT batches)
- No temporary tables (use CTEs or regular tables with _tmp_ prefix)
- No PL/pgSQL (SQL functions only)
- No partial indexes (WHERE clause)
- No savepoints within transactions
- No table inheritance (INHERITS)
- No materialized views (use regular views)
- JSONB is runtime-only (store as json, cast with ::jsonb in queries)
- No LISTEN/NOTIFY (use SNS/SQS/EventBridge)
- No advisory locks (use DynamoDB conditional writes)
- Max 10 schemas per database (consolidate if >10)
- Max 1,000 tables per database
- Max 24 indexes per table
- Max 255 columns per table
- Async indexes not immediately usable (check pg_index.indisvalid)
- C collation sorts differently than en_US.UTF-8
- OCC conflicts (40001) are normal — must have retry logic
- Sequence CACHE must be exactly 1 or ≥65536 (no values in between)
- Max transaction time is 5 minutes
- Max 3,000 rows modified per transaction
- Max 10 MiB data modified per transaction
- SELECT FOR UPDATE does NOT block — OCC detects conflicts at commit
- Sequential/auto-increment PKs create hot spots (use UUIDs)
- VACUUM not needed (automatic) — remove from maintenance scripts

## Troubleshooting

- **"Unsupported dialect"**: Only postgresql supported currently
- **DDL fails on DSQL**: Check for expression indexes, unsupported CHECK functions
- **PL/pgSQL stub**: Uses PERFORM or complex ELSIF — rewrite manually
- **Sequence CACHE warning**: Add CACHE 1 or CACHE 65536
- **JSONB confusion**: DSQL stores as json, use ::jsonb in queries
- **Partial index fails**: WHERE not supported — full index created instead
- **OCC 40001 errors**: Normal — implement retry with exponential backoff
- **ORDER BY different**: C collation — sort in app layer if locale-aware needed
- **uuid_generate_v4() fails**: Replace with gen_random_uuid()
- **lastval() fails**: Use currval('sequence_name') with explicit name
- **Index not used in query plan**: Check if async index is built (indisvalid = true)
- **GRANT fails for non-admin**: Only admin role can GRANT; connect as admin
- **Schema not visible after CREATE**: Refresh connection (disconnect/reconnect)
- **"more than 10 schemas not allowed"**: Consolidate schemas or use table prefixes
- **"more than 1000 tables not allowed"**: Split across clusters or consolidate
- **"transaction row limit exceeded"**: Batch to ≤3,000 rows per transaction
- **"transaction size limit 10mb exceeded"**: Reduce batch size or row payload
- **"transaction age limit of 300s exceeded"**: Transaction took >5 min — break into smaller units
- **Data load too slow**: Increase batch size (up to 1000), use CACHE 65536 sequences
- **generate_series() in INSERT**: Results count toward 3,000 row limit
- **Hot spot / slow writes**: Sequential PK — switch to UUID or random distribution
- **SELECT FOR UPDATE not blocking**: Expected — DSQL uses OCC, not locks
- **COPY command fails**: Not supported — use batched INSERT statements
