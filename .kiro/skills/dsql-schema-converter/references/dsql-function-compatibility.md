# PostgreSQL Function Compatibility in DSQL

Reference for which built-in PostgreSQL functions work in Aurora DSQL and which need alternatives.

Sources:
- [Supported SQL Features](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-sql-features.html)
- [Supported Data Types — JSON Functions](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-data-types.html)
- [PostgreSQL 16 JSON Functions and Operators](https://www.postgresql.org/docs/current/functions-json.html)
- [Migration Guide](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-migration-guide.html)

---

## Fully Supported Functions

### Aggregate Functions

| Function | Notes |
|---|---|
| COUNT(*), COUNT(col) | Full support |
| SUM(col) | Full support |
| AVG(col) | Full support |
| MIN(col), MAX(col) | Full support |
| bool_and(col), bool_or(col) | Full support |
| string_agg(col, delimiter) | Full support |
| array_agg(col) | Returns runtime array (cannot store result) |
| json_agg(col) | Full support (returns json) |
| jsonb_agg(col) | Full support (runtime jsonb) |
| json_object_agg(key, val) | Full support |

### String Functions

| Function | Notes |
|---|---|
| length(text) | Full support |
| char_length(text) | Full support |
| lower(text), upper(text) | Full support (C collation — ASCII only) |
| trim(), ltrim(), rtrim() | Full support |
| substring(text, start, len) | Full support |
| position(sub IN text) | Full support |
| replace(text, from, to) | Full support |
| concat(a, b, ...) | Full support |
| concat_ws(sep, a, b, ...) | Full support |
| left(text, n), right(text, n) | Full support |
| repeat(text, n) | Full support |
| reverse(text) | Full support |
| split_part(text, delim, n) | Full support |
| format(fmt, args...) | Full support |
| encode(bytea, format) | Full support |
| decode(text, format) | Full support |
| md5(text) | Full support |
| regexp_replace(text, pat, rep) | Full support |
| regexp_match(text, pat) | Full support (returns text[]) |

### Numeric Functions

| Function | Notes |
|---|---|
| abs(n) | Full support |
| ceil(n), floor(n) | Full support |
| round(n), round(n, s) | Full support |
| trunc(n), trunc(n, s) | Full support |
| mod(a, b) | Full support |
| power(a, b) | Full support |
| sqrt(n) | Full support |
| random() | Full support |
| greatest(a, b, ...) | Full support |
| least(a, b, ...) | Full support |

### Date/Time Functions

| Function | Notes |
|---|---|
| now() | Full support (returns timestamptz) |
| current_timestamp | Full support |
| current_date | Full support |
| current_time | Full support |
| clock_timestamp() | Full support |
| date_trunc(field, ts) | Full support |
| date_part(field, ts) | Full support |
| extract(field FROM ts) | Full support |
| age(ts1, ts2) | Full support |
| make_interval(...) | Full support |
| make_date(y, m, d) | Full support |
| to_char(ts, fmt) | Full support |
| to_date(text, fmt) | Full support |
| to_timestamp(text, fmt) | Full support |

### JSON Functions

| Function | Notes |
|---|---|
| json_build_object(k, v, ...) | Full support |
| json_build_array(a, b, ...) | Full support |
| jsonb_build_object(k, v, ...) | Full support (runtime jsonb) |
| jsonb_build_array(a, b, ...) | Full support (runtime jsonb) |
| row_to_json(record) | Full support |
| json_extract_path(json, ...) | Full support |
| json_extract_path_text(json, ...) | Full support |
| jsonb_extract_path(json, ...) | Full support (cast to jsonb first) |
| json_each(json) | Full support |
| json_array_elements(json) | Full support |
| jsonb_set(jsonb, path, val) | Full support (runtime) |
| jsonb_strip_nulls(jsonb) | Full support (runtime) |
| json_typeof(json) | Full support |
| json_array_length(json) | Full support |
| -> operator | Full support |
| ->> operator | Full support |
| #> operator | Full support |
| #>> operator | Full support |
| @> containment (jsonb) | Full support (runtime, not indexable) |
| ? key exists (jsonb) | Full support (runtime) |

### UUID Functions

| Function | Notes |
|---|---|
| gen_random_uuid() | Full support (built-in, no extension needed) |
| uuid_generate_v4() | NOT available (use gen_random_uuid()) |

### Conditional Expressions

| Function | Notes |
|---|---|
| CASE WHEN ... THEN ... END | Full support |
| COALESCE(a, b, ...) | Full support |
| NULLIF(a, b) | Full support |
| GREATEST(a, b, ...) | Full support |
| LEAST(a, b, ...) | Full support |

### Subquery Expressions

| Expression | Notes |
|---|---|
| EXISTS (subquery) | Full support |
| IN (subquery) | Full support |
| ANY/SOME (subquery) | Full support |
| ALL (subquery) | Full support |

### Window Functions

| Function | Notes |
|---|---|
| ROW_NUMBER() OVER (...) | Full support |
| RANK() OVER (...) | Full support |
| DENSE_RANK() OVER (...) | Full support |
| LAG(col) OVER (...) | Full support |
| LEAD(col) OVER (...) | Full support |
| FIRST_VALUE(col) OVER (...) | Full support |
| LAST_VALUE(col) OVER (...) | Full support |
| NTH_VALUE(col, n) OVER (...) | Full support |
| NTILE(n) OVER (...) | Full support |
| SUM/AVG/COUNT OVER (...) | Full support |

### Type Casting

| Expression | Notes |
|---|---|
| CAST(x AS type) | Full support |
| x::type | Full support |
| x::jsonb | Runtime cast (json stored, jsonb in queries) |

---

## Partially Supported Functions

### generate_series()

```sql
-- Works for integer series
SELECT generate_series(1, 100);  -- ✅

-- Works for timestamp series
SELECT generate_series(
  '2024-01-01'::timestamp,
  '2024-12-31'::timestamp,
  '1 month'::interval
);  -- ✅

-- LIMITATION: Cannot use in FROM clause with large ranges in write transactions
-- (3,000 row limit applies if inserting results)
```

### array_agg() / ARRAY constructor

```sql
-- Works at runtime (SELECT)
SELECT array_agg(name) FROM users WHERE org_id = 1;  -- ✅ returns text[]

-- CANNOT store result in a column (arrays not a stored type)
-- Use json_agg() instead if you need to persist
SELECT json_agg(name) FROM users WHERE org_id = 1;  -- ✅ storable as json
```

### unnest()

```sql
-- Works with runtime arrays
SELECT unnest(ARRAY['a','b','c']);  -- ✅

-- Works with json arrays via json_array_elements
SELECT json_array_elements_text('["a","b","c"]'::json);  -- ✅
```

---

## Not Supported / No Equivalent

### Full-Text Search

| Function | Alternative |
|---|---|
| to_tsvector(text) | Application-layer (OpenSearch, Elasticsearch) |
| to_tsquery(text) | Application-layer |
| ts_rank(tsvector, tsquery) | Application-layer |
| plainto_tsquery(text) | Application-layer |
| @@ operator | Application-layer |

### Advisory Locks

| Function | Alternative |
|---|---|
| pg_advisory_lock(id) | DynamoDB conditional write or Redis SETNX |
| pg_advisory_unlock(id) | Release in DynamoDB/Redis |
| pg_try_advisory_lock(id) | DynamoDB conditional write |

### Notification

| Function | Alternative |
|---|---|
| pg_notify(channel, payload) | Amazon SNS, SQS, or EventBridge |
| LISTEN channel | SQS polling or EventBridge rules |
| NOTIFY channel | SNS Publish |

### Table/System Functions

| Function | Alternative |
|---|---|
| pg_table_size(table) | Not available (DSQL manages storage) |
| pg_total_relation_size(table) | Not available |
| pg_stat_user_tables | Limited system catalog support |
| pg_stat_activity | Not available |
| pg_cancel_backend(pid) | Not available |
| pg_terminate_backend(pid) | Not available |
| current_setting(name) | Limited (no custom GUCs) |
| set_config(name, val, local) | Not available |

### Sequence Functions (edge cases)

| Function | Notes |
|---|---|
| nextval(regclass) | ✅ Supported |
| currval(regclass) | ✅ Supported |
| setval(regclass, bigint) | ✅ Supported |
| lastval() | NOT supported (use currval with explicit sequence name) |

### Large Object Functions

| Function | Alternative |
|---|---|
| lo_create(oid) | Use S3 for large objects |
| lo_import(path) | Use S3 |
| lo_export(oid, path) | Use S3 |

### Maintenance Commands

| Command | DSQL Behavior |
|---|---|
| VACUUM | Not needed — DSQL handles automatically |
| VACUUM ANALYZE | Not needed — automatic statistics |
| ANALYZE (table) | Supported (relation name only) |
| REINDEX | Not needed — DSQL manages indexes |
| CLUSTER | Not applicable — data is always PK-ordered |
| COPY FROM/TO | Not supported — use INSERT batches |

### Transaction Control

| Command | DSQL Support |
|---|---|
| BEGIN / COMMIT / ROLLBACK | ✅ Supported |
| SAVEPOINT | NOT supported |
| RELEASE SAVEPOINT | NOT supported |
| ROLLBACK TO SAVEPOINT | NOT supported |
| SET TRANSACTION ISOLATION LEVEL | Only REPEATABLE READ accepted |

### XML Functions

| Function | Alternative |
|---|---|
| xmlparse(...) | Store as text, parse in application |
| xpath(expr, xml) | Parse in application |
| xmlelement(...) | Build in application |

---

## Collation Impact on String Functions

DSQL uses C collation exclusively. This affects:

| Behavior | PostgreSQL (en_US.UTF-8) | DSQL (C collation) |
|---|---|---|
| ORDER BY text | Locale-aware (ä after a) | Byte-order (ä after z) |
| lower('Ä') | 'ä' | 'ä' (works for common cases) |
| upper('ß') | 'SS' (locale-dependent) | May not transform |
| LIKE 'abc%' | Uses locale rules | Byte comparison |
| Pattern matching | Locale-aware | Byte-based |

### Mitigation

- For case-insensitive search: use `lower(col) = lower(input)` in queries
- For locale-aware sorting: sort in application layer
- For accent-insensitive search: normalize in application before storing

---

## DSQL-Specific Functions & Features

These are available in DSQL but not standard PostgreSQL or behave differently:

| Function/Feature | Purpose |
|---|---|
| gen_random_uuid() | Generate UUID v4 (built-in, no extension needed) |
| CREATE INDEX ASYNC | Non-blocking index creation (required syntax) |
| sys.jobs | Monitor async DDL job status |
| CREATE DOMAIN | Supported (unlike some distributed DBs) |
| EXPLAIN ANALYZE VERBOSE | Recommended for identifying non-pushdown operations |

### JSON Storage Clarification

Per official DSQL documentation (May 2026):
- `json` is a **supported stored data type** with all PostgreSQL JSON operators
- `jsonb` is a **runtime-only type** — use `::jsonb` cast in queries
- All functions from PostgreSQL section 9.16 (JSON Functions and Operators) work
- `json_populate_record` works with table/view row types (not composite types)
- JSON columns support automatic compression (1 MiB compressed limit)

```sql
-- Correct: store as json, query with jsonb operators
CREATE TABLE config (id uuid PRIMARY KEY, data json);
INSERT INTO config VALUES (gen_random_uuid(), '{"key": "value"}');
SELECT data::jsonb -> 'key' FROM config;  -- ✅ works
SELECT data::jsonb @> '{"key": "value"}' FROM config;  -- ✅ containment works
```

---

## Migration Checklist for Function Usage

1. **grep your codebase** for functions in the "Not Supported" section
2. **Replace uuid_generate_v4()** with `gen_random_uuid()`
3. **Replace lastval()** with explicit `currval('sequence_name')`
4. **Replace pg_notify/LISTEN** with SNS/SQS/EventBridge
5. **Replace advisory locks** with DynamoDB conditional writes
6. **Replace full-text search** with OpenSearch/Elasticsearch
7. **Test ORDER BY** results with C collation — may need app-layer sort
8. **Test LIKE patterns** with non-ASCII data
