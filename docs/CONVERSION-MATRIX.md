# Conversion Matrix

Complete reference of every PostgreSQL feature and its DSQL conversion outcome.

---

## 1. Data Type Mappings

### Numeric Types

| PostgreSQL Type | Aliases | DSQL Type | Indexable | Notes |
|---|---|---|---|---|
| SMALLINT | INT2 | smallint | ✅ | Direct mapping |
| INTEGER | INT, INT4 | integer | ✅ | Direct mapping |
| BIGINT | INT8 | bigint | ✅ | Direct mapping |
| REAL | FLOAT4 | real | ✅ | Direct mapping |
| DOUBLE PRECISION | FLOAT8 | double precision | ✅ | Direct mapping |
| NUMERIC(p,s) | DECIMAL(p,s) | numeric(p,s) | ✅ | Max precision 38, max scale 37 |
| MONEY | — | numeric(19,4) | ✅ | Converted to numeric |
| SERIAL | SERIAL4 | integer | ✅ | Auto-increment removed; use sequence |
| BIGSERIAL | SERIAL8 | bigint | ✅ | Auto-increment removed; use sequence |
| SMALLSERIAL | SERIAL2 | smallint | ✅ | Auto-increment removed; use sequence |

### String Types

| PostgreSQL Type | Aliases | DSQL Type | Indexable | Notes |
|---|---|---|---|---|
| CHAR(n) | CHARACTER(n) | char(n) COLLATE "C" | ✅ | Max 4096 bytes |
| VARCHAR(n) | CHARACTER VARYING(n) | varchar(n) COLLATE "C" | ✅ | Max 65535 bytes |
| BPCHAR | — | bpchar COLLATE "C" | ✅ | Blank-padded character |
| TEXT | — | text COLLATE "C" | ✅ | Max 1 MiB |

### Date/Time Types

| PostgreSQL Type | Aliases | DSQL Type | Indexable | Notes |
|---|---|---|---|---|
| DATE | — | date | ✅ | Direct mapping |
| TIME | TIME WITHOUT TIME ZONE | time | ✅ | Direct mapping |
| TIMETZ | TIME WITH TIME ZONE | time with time zone | ❌ | Not indexable |
| TIMESTAMP | TIMESTAMP WITHOUT TIME ZONE | timestamp | ✅ | Direct mapping |
| TIMESTAMPTZ | TIMESTAMP WITH TIME ZONE | timestamptz | ✅ | Direct mapping |
| INTERVAL | — | interval | ❌ | Not indexable |

### Binary & Boolean Types

| PostgreSQL Type | Aliases | DSQL Type | Indexable | Notes |
|---|---|---|---|---|
| BOOLEAN | BOOL | boolean | ✅ | Direct mapping |
| BYTEA | — | bytea | ❌ | Max 1 MiB |
| UUID | — | uuid | ✅ | Direct mapping |
| BIT(n) | VARBIT(n) | text COLLATE "C" | ✅ | No native equivalent |

### JSON Types

| PostgreSQL Type | DSQL Type | Indexable | Notes |
|---|---|---|---|
| JSON | json | ❌ | Native support, all operators work |
| JSONB | json | ❌ | Stored as json; use `::jsonb` in queries for operators |

### Network Types (→ text)

| PostgreSQL Type | DSQL Type | Indexable | Notes |
|---|---|---|---|
| INET | text COLLATE "C" | ✅ | Runtime-only in PostgreSQL sense |
| CIDR | text COLLATE "C" | ✅ | No native equivalent |
| MACADDR | text COLLATE "C" | ✅ | No native equivalent |
| MACADDR8 | text COLLATE "C" | ✅ | No native equivalent |

### Full-Text Search Types (→ text)

| PostgreSQL Type | DSQL Type | Indexable | Notes |
|---|---|---|---|
| TSVECTOR | text COLLATE "C" | ✅ | No FTS in DSQL |
| TSQUERY | text COLLATE "C" | ✅ | No FTS in DSQL |

### Geometric Types (→ text)

| PostgreSQL Type | DSQL Type | Notes |
|---|---|---|
| POINT | text COLLATE "C" | No spatial support |
| LINE | text COLLATE "C" | No spatial support |
| LSEG | text COLLATE "C" | No spatial support |
| BOX | text COLLATE "C" | No spatial support |
| PATH | text COLLATE "C" | No spatial support |
| POLYGON | text COLLATE "C" | No spatial support |
| CIRCLE | text COLLATE "C" | No spatial support |

### Array Types (→ text)

| PostgreSQL Type | DSQL Type | Notes |
|---|---|---|
| TEXT[] | text COLLATE "C" | Arrays are runtime-only |
| INTEGER[] | text COLLATE "C" | Store as json array instead |
| UUID[] | text COLLATE "C" | Store as json array instead |
| *any*[] | text COLLATE "C" | All array types → text |

### System/Internal Types (→ text)

| PostgreSQL Type | DSQL Type | Notes |
|---|---|---|
| OID | text COLLATE "C" | No equivalent |
| REGCLASS | text COLLATE "C" | No equivalent |
| REGTYPE | text COLLATE "C" | No equivalent |
| PG_LSN | text COLLATE "C" | No equivalent |
| XML | text COLLATE "C" | No native XML |

---

## 2. Schema Object Conversions

### Tables

| PostgreSQL Feature | DSQL Conversion | Action |
|---|---|---|
| CREATE TABLE | CREATE TABLE | Preserved (types mapped) |
| SERIAL/BIGSERIAL columns | integer/bigint + CREATE SEQUENCE | Auto-generated sequence |
| DEFAULT expressions | Preserved | Most expressions work |
| NOT NULL | Preserved | Direct support |
| UNIQUE constraints | Preserved | Direct support |
| PRIMARY KEY | Preserved | Direct support |
| CHECK constraints | Preserved | Direct support |
| GENERATED ALWAYS AS ... STORED | Preserved | DSQL supports computed columns |
| FOREIGN KEY | Removed | → validate_fk_*() function |
| PARTITION BY | Removed | DSQL handles distribution |
| INHERITS | Removed | Columns merged into child |
| TEMPORARY / TEMP | Removed | → regular table with `_tmp_` prefix |
| UNLOGGED | Removed | All DSQL tables are durable |
| TABLESPACE | Removed | DSQL manages storage |
| WITH (fillfactor=...) | Removed | No storage parameters |

### Indexes

| PostgreSQL Feature | DSQL Conversion | Notes |
|---|---|---|
| CREATE INDEX | CREATE INDEX ASYNC | Always async |
| CREATE UNIQUE INDEX | CREATE UNIQUE INDEX ASYNC | Uniqueness preserved |
| USING btree | USING btree (default) | Direct |
| USING gin | → btree | GIN not supported |
| USING gist | → btree | GiST not supported |
| USING brin | → btree | BRIN not supported |
| USING hash | → btree | Hash not supported |
| WHERE clause (partial) | Removed | Not supported |
| INCLUDE (columns) | Preserved | DSQL supports covering indexes |
| DESC/ASC | Preserved | Sort order supported |
| NULLS FIRST/LAST | Preserved | Null ordering supported |
| CONCURRENTLY | Removed | Use ASYNC instead |
| Expression indexes | Removed | Not supported |
| IF NOT EXISTS | Preserved | Supported |

### Sequences

| PostgreSQL Feature | DSQL Conversion | Notes |
|---|---|---|
| CREATE SEQUENCE | CREATE SEQUENCE ... CACHE 1 | CACHE clause required |
| START WITH | Preserved | Direct support |
| INCREMENT BY | Preserved | Direct support |
| MINVALUE/MAXVALUE | Preserved | Direct support |
| CYCLE/NO CYCLE | Preserved | Direct support |
| CACHE n | CACHE 1 or CACHE ≥65536 | Must be 1 or ≥65536 |
| OWNED BY | Removed | Not supported |
| nextval() | Preserved | Direct support |
| currval() | Preserved | Direct support |
| setval() | Preserved | Direct support |

### Views

| PostgreSQL Feature | DSQL Conversion | Notes |
|---|---|---|
| CREATE VIEW | CREATE VIEW | Preserved |
| CREATE OR REPLACE VIEW | CREATE OR REPLACE VIEW | Preserved |
| WITH CHECK OPTION | Removed | Not supported |
| CREATE MATERIALIZED VIEW | → CREATE VIEW | Demoted to regular view |
| REFRESH MATERIALIZED VIEW | N/A | No longer needed |

### Functions

| PostgreSQL Feature | DSQL Conversion | Notes |
|---|---|---|
| LANGUAGE sql | Preserved | Fully supported |
| LANGUAGE plpgsql | → LANGUAGE sql | Transpiled (10 patterns) |
| RETURNS TRIGGER | Removed | Triggers not supported |
| RETURNS void/text/int/etc. | Preserved | Return types supported |
| IMMUTABLE/STABLE/VOLATILE | Preserved | Supported |
| SECURITY DEFINER | Removed | Not supported |
| SET search_path | Removed | Single schema |

### Triggers

| PostgreSQL Feature | DSQL Conversion | Notes |
|---|---|---|
| CREATE TRIGGER | Dropped entirely | → SQL function + app call |
| BEFORE/AFTER | N/A | Logic moves to app layer |
| FOR EACH ROW | N/A | Logic moves to app layer |
| WHEN (condition) | N/A | Logic moves to app layer |

### Types

| PostgreSQL Feature | DSQL Conversion | Notes |
|---|---|---|
| CREATE TYPE ... AS ENUM | → CHECK constraint on column | Values preserved |
| CREATE TYPE ... AS (composite) | Dropped | Use json or separate columns |
| CREATE DOMAIN | Dropped | Use CHECK constraint |

### Extensions

| PostgreSQL Extension | DSQL Equivalent | Notes |
|---|---|---|
| uuid-ossp | gen_random_uuid() | Built-in function |
| pgcrypto | gen_random_uuid() | App-layer for other crypto |
| pg_trgm | None | App-layer trigram search |
| postgis | None | Store coordinates as text/json |
| hstore | json | Use json type |
| citext | varchar + app-layer LOWER() | Case-insensitive via app |
| pg_stat_statements | None | DSQL has own monitoring |

---

## 3. Constraint Conversions

| PostgreSQL Constraint | DSQL Conversion | Enforcement |
|---|---|---|
| PRIMARY KEY | Preserved | Database |
| UNIQUE | Preserved | Database |
| NOT NULL | Preserved | Database |
| CHECK (expression) | Preserved | Database |
| DEFAULT (expression) | Preserved | Database |
| FOREIGN KEY | → validate_fk_*() function | Application |
| EXCLUDE USING | Dropped (no equivalent) | Application |
| ON DELETE CASCADE | Dropped | Application (cascade function) |
| ON UPDATE CASCADE | Dropped | Application (cascade function) |
| ON DELETE SET NULL | Dropped | Application |
| DEFERRABLE | Dropped | Not supported |

---

## 4. PL/pgSQL Pattern Matrix

| # | Pattern Name | Detection Signal | DSQL Output | App Action Required |
|---|---|---|---|---|
| 1 | SET_COLUMN | `NEW.<col> = <expr>; RETURN NEW;` | SQL UPDATE function | Call after UPDATE |
| 2 | VALIDATION | `IF NEW.<col> ... RAISE EXCEPTION` | CHECK constraint | None (auto-enforced) |
| 3 | AUDIT_INSERT | `INSERT INTO audit_log ... TG_OP` | SQL INSERT function | Call after DML |
| 4 | CASCADE_DML | `UPDATE/DELETE ... WHERE ... OLD.id` | SQL DML function | Call before DELETE |
| 5 | FOR_LOOP | `FOR r IN SELECT ... LOOP UPDATE` | Set-based UPDATE...FROM | None |
| 6 | IF_ELSE | `IF cond THEN RETURN x ELSE RETURN y` | CASE WHEN expression | None |
| 7 | UPSERT | `EXCEPTION WHEN unique_violation` | ON CONFLICT clause | None |
| 8 | DYNAMIC_SQL | `EXECUTE format(...)` | One function per table | Call specific function |
| 9 | CURSOR | `DECLARE cur CURSOR ... FOR rec IN cur LOOP` | INSERT...SELECT | None |
| 10 | COALESCE | `EXCEPTION WHEN no_data_found` | COALESCE(subquery, NULL) | None |
| — | PERFORM | `PERFORM pg_notify(...)` | Stub (TODO) | Manual rewrite |
| — | Complex ELSIF | 3+ conditional branches | Stub (TODO) | Manual rewrite |

---

## 5. Transaction & Operational Constraints

| PostgreSQL Behavior | DSQL Behavior | Migration Impact |
|---|---|---|
| Multiple DDL in one transaction | 1 DDL per transaction | Split DDL scripts |
| READ COMMITTED (default) | REPEATABLE READ (fixed) | May see OCC conflicts |
| TRUNCATE TABLE | Not supported | Use DELETE FROM |
| LOCK TABLE | Not supported | OCC handles concurrency |
| LISTEN/NOTIFY | Not supported | Use SQS/SNS/EventBridge |
| Advisory locks | Not supported | Use DynamoDB for locking |
| Savepoints | Not supported | Restructure transaction logic |
| Row-level security | Not supported | App-layer authorization |
| Schemas (CREATE SCHEMA) | Single public schema | Prefix table names |

---

## 6. Limits Reference

| Resource | DSQL Limit |
|---|---|
| Max rows per transaction (write) | 3,000 |
| DDL statements per transaction | 1 |
| NUMERIC precision | 38 |
| NUMERIC scale | 37 |
| VARCHAR max length | 65,535 bytes |
| TEXT/json/bytea max size | 1 MiB |
| CHAR max length | 4,096 bytes |
| Sequence CACHE | 1 or ≥65,536 |
| Database per cluster | 1 (named `postgres`) |
| Connection timeout | 1 hour |
| Collation | C only (UTF-8 encoding) |

---

## 7. Conversion Decision Flowchart

```
Is it a data type?
├── Yes → Look up in Type Mapping table (Section 1)
│         ├── Direct mapping exists → Use it
│         ├── Falls to TEXT → Add COLLATE "C", note in report
│         └── SERIAL/BIGSERIAL → integer/bigint + sequence
│
├── Is it an index?
│   ├── Yes → CREATE INDEX ASYNC
│   │         ├── btree → preserved
│   │         ├── gin/gist/brin/hash → btree
│   │         ├── WHERE clause → removed
│   │         ├── INCLUDE → preserved
│   │         └── Expression → removed (warn)
│   │
├── Is it a constraint?
│   ├── PK/UNIQUE/NOT NULL/CHECK/DEFAULT → preserved
│   ├── FOREIGN KEY → validate_fk_*() function
│   └── EXCLUDE → dropped (warn)
│
├── Is it a function?
│   ├── LANGUAGE sql → preserved
│   ├── LANGUAGE plpgsql → match pattern (1-10)
│   │   ├── Pattern matched → generate SQL equivalent
│   │   └── No match → generate stub
│   └── Other languages → dropped (warn)
│
├── Is it a trigger?
│   └── Dropped → replacement function generated
│
├── Is it a type?
│   ├── ENUM → CHECK constraint on referencing columns
│   ├── Composite → dropped (use json)
│   └── Domain → dropped (use CHECK)
│
├── Is it a view?
│   ├── Regular → preserved
│   └── Materialized → demoted to regular VIEW
│
├── Is it a sequence?
│   └── Preserved with CACHE 1
│
├── Is it a table modifier?
│   ├── PARTITION BY → removed
│   ├── INHERITS → removed (columns merged)
│   ├── TEMPORARY → regular table (_tmp_ prefix)
│   └── UNLOGGED → removed
│
└── Is it an extension?
    └── Dropped with equivalent noted
```

---

## 8. Migration Effort Estimation

Use this matrix to estimate effort for a PostgreSQL → DSQL migration:

| Schema Feature | Count Method | Effort per Item | Automation |
|---|---|---|---|
| Tables | COUNT tables | Low | Full |
| Columns (type mapping) | COUNT columns | None | Full |
| ENUM types | COUNT DISTINCT types | Low | Full |
| Foreign keys | COUNT FK constraints | Medium | Full (function generated) |
| Indexes | COUNT indexes | Low | Full |
| PL/pgSQL functions (patterns 1-10) | COUNT functions | Low | Full |
| PL/pgSQL functions (unconvertible) | COUNT stubs | High | Manual |
| Triggers | COUNT triggers | Medium | Partial (function generated, app wiring manual) |
| Materialized views | COUNT mat views | Low | Full |
| Extensions | COUNT extensions | Varies | Manual (find alternatives) |
| TRUNCATE usage | grep codebase | Low | Find & replace with DELETE |
| Multi-DDL transactions | grep codebase | Medium | Split into single-DDL txns |
| LISTEN/NOTIFY usage | grep codebase | High | Rewrite with event service |

**Effort scale:**
- **None** — Handled automatically, no review needed
- **Low** — Auto-converted, quick review recommended
- **Medium** — Auto-converted but requires app-layer changes
- **High** — Manual rewrite required
