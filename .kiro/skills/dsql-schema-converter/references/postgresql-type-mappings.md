# PostgreSQL → DSQL Type Mappings

Two-stage pipeline: PostgreSQL type → NormalizedType → DSQL SQL type.

Source: [DSQL Supported Data Types](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-data-types.html)

Additional sources:
- [PostgreSQL 16 Data Types](https://www.postgresql.org/docs/16/datatype.html) — source type definitions
- [PostgreSQL 16 JSON Functions](https://www.postgresql.org/docs/current/functions-json.html) — JSON/JSONB operator support
- [DSQL SQL Dialect Blog](https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/) — indexability, PK-ordered storage
- [Quotas and Limits](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/CHAP_quotas.html) — max column/row sizes

---

## Stage 1: PostgreSQL → NormalizedType

### Numeric Types

| PostgreSQL Type | Aliases | NormalizedType | Notes |
|---|---|---|---|
| SMALLINT | INT2 | SMALLINT | -32768 to +32767 |
| INTEGER | INT, INT4 | INTEGER | -2147483648 to +2147483647 |
| BIGINT | INT8 | BIGINT | -9223372036854775808 to +9223372036854775807 |
| REAL | FLOAT4 | REAL | 6 decimal digits precision |
| DOUBLE PRECISION | FLOAT8 | DOUBLE_PRECISION | 15 decimal digits precision |
| FLOAT(1-24) | — | REAL | Maps to 4-byte float |
| FLOAT(25-53) | — | DOUBLE_PRECISION | Maps to 8-byte float |
| FLOAT (no precision) | — | DOUBLE_PRECISION | Default is 8-byte |
| NUMERIC(p,s) | DECIMAL(p,s), DEC(p,s) | NUMERIC | Precision/scale preserved |
| NUMERIC (no precision) | DECIMAL | NUMERIC | DSQL defaults to NUMERIC(18,6) |
| MONEY | — | NUMERIC | Mapped to numeric(19,4) |
| SERIAL | SERIAL4 | INTEGER | Auto-increment removed; use sequence or identity |
| BIGSERIAL | SERIAL8 | BIGINT | Auto-increment removed; use sequence or identity |
| SMALLSERIAL | SERIAL2 | SMALLINT | Auto-increment removed; use sequence or identity |

### Character Types

| PostgreSQL Type | Aliases | NormalizedType | Notes |
|---|---|---|---|
| CHAR(n) | CHARACTER(n) | CHAR | Fixed-length. COLLATE "C" added. |
| CHAR (no length) | CHARACTER | CHAR | Treated as CHAR(1) |
| VARCHAR(n) | CHARACTER VARYING(n) | VARCHAR | Variable-length. COLLATE "C" added. |
| VARCHAR (no length) | CHARACTER VARYING | VARCHAR | No explicit limit (DSQL max 65535 bytes) |
| BPCHAR | — | BPCHAR | Blank-padded. COLLATE "C" added. |
| TEXT | — | TEXT | Variable unlimited. COLLATE "C" added. |

### Date/Time Types

| PostgreSQL Type | Aliases | NormalizedType | Notes |
|---|---|---|---|
| DATE | — | DATE | Calendar date (year, month, day) |
| TIME(p) | TIME WITHOUT TIME ZONE | TIME | Precision (0-6) preserved. Default 6. |
| TIMETZ(p) | TIME WITH TIME ZONE | TIMETZ | Precision preserved. |
| TIMESTAMP(p) | TIMESTAMP WITHOUT TIME ZONE | TIMESTAMP | Precision (0-6) preserved. Default 6. |
| TIMESTAMPTZ(p) | TIMESTAMP WITH TIME ZONE | TIMESTAMPTZ | Precision preserved. |
| INTERVAL(p) | — | INTERVAL | Precision preserved. |

### Boolean, Binary, UUID

| PostgreSQL Type | Aliases | NormalizedType | Notes |
|---|---|---|---|
| BOOLEAN | BOOL | BOOLEAN | true/false |
| BYTEA | — | BYTEA | Binary data |
| UUID | — | UUID | 128-bit identifier |

### JSON Types

| PostgreSQL Type | NormalizedType | Notes |
|---|---|---|
| JSON | JSON | Direct — DSQL supports json natively as stored type |
| JSONB | JSON | Stored as json. Use `::jsonb` in queries for JSONB operators. |

### Types Mapped to TEXT (no native DSQL equivalent)

| PostgreSQL Type | NormalizedType | Reason |
|---|---|---|
| TEXT[] | TEXT | Arrays are runtime-only in DSQL |
| INTEGER[] | TEXT | Arrays are runtime-only |
| UUID[] | TEXT | Arrays are runtime-only |
| *any*[] | TEXT | All array column types → text |
| INET | TEXT | Runtime-only type in DSQL |
| CIDR | TEXT | No native equivalent |
| MACADDR | TEXT | No native equivalent |
| MACADDR8 | TEXT | No native equivalent |
| TSVECTOR | TEXT | No full-text search in DSQL |
| TSQUERY | TEXT | No full-text search in DSQL |
| XML | TEXT | No native XML support |
| BIT(n) | TEXT | No native bit string |
| VARBIT(n) | TEXT | No native bit string |
| POINT | TEXT | No geometric types |
| LINE | TEXT | No geometric types |
| LSEG | TEXT | No geometric types |
| BOX | TEXT | No geometric types |
| PATH | TEXT | No geometric types |
| POLYGON | TEXT | No geometric types |
| CIRCLE | TEXT | No geometric types |
| OID | TEXT | System type, no equivalent |
| REGCLASS | TEXT | System type, no equivalent |
| REGTYPE | TEXT | System type, no equivalent |
| PG_LSN | TEXT | System type, no equivalent |

---

## Stage 2: NormalizedType → DSQL SQL Type (fixed)

| NormalizedType | DSQL SQL Type | Storage Size | Max Size | Indexable | Notes |
|---|---|---|---|---|---|
| SMALLINT | smallint | 2 bytes | — | Yes | |
| INTEGER | integer | 4 bytes | — | Yes | |
| BIGINT | bigint | 8 bytes | — | Yes | |
| REAL | real | 4 bytes | — | Yes | 6 decimal digits |
| DOUBLE_PRECISION | double precision | 8 bytes | — | Yes | 15 decimal digits |
| NUMERIC | numeric(p,s) | 8 + 2 bytes/digit | 27 bytes | Yes | Max precision 38, max scale 37. Default (18,6). |
| CHAR | char(n) COLLATE "C" | Variable up to 4100 bytes | 4096 bytes | Yes | Fixed-length |
| VARCHAR | varchar(n) COLLATE "C" | Variable up to 65539 bytes | 65535 bytes | Yes | Variable-length |
| BPCHAR | bpchar COLLATE "C" | Variable up to 4100 bytes | 4096 bytes | Yes | Blank-padded |
| TEXT | text COLLATE "C" | Variable up to 1 MiB | 1 MiB | Yes | |
| DATE | date | 4 bytes | — | Yes | 4713 BC – 5874897 AD |
| TIME | time(p) | 8 bytes | — | Yes | Microsecond resolution |
| TIMETZ | time(p) with time zone | 12 bytes | — | No | |
| TIMESTAMP | timestamp(p) | 8 bytes | — | Yes | 4713 BC – 294276 AD |
| TIMESTAMPTZ | timestamptz(p) | 8 bytes | — | Yes | 4713 BC – 294276 AD |
| INTERVAL | interval(p) | 16 bytes | — | No | ±178000000 years |
| BOOLEAN | boolean | 1 byte | — | Yes | |
| BYTEA | bytea | Variable up to 1 MiB | 1 MiB | No | |
| UUID | uuid | 16 bytes | — | Yes | |
| JSON | json | Variable up to 1 MiB | 1 MiB (compressed) | No | Auto-compression; actual data can exceed 1 MiB if it compresses below limit |

---

## DSQL-Specific Behaviors

### NUMERIC Default

If you don't specify precision/scale, DSQL enforces `NUMERIC(18,6)` as the default. PostgreSQL allows unbounded NUMERIC. This means:

```sql
-- PostgreSQL: stores any precision
CREATE TABLE t (val NUMERIC);
INSERT INTO t VALUES (12345678901234567890.1234567890);  -- works

-- DSQL: enforces (18,6) if not specified
CREATE TABLE t (val NUMERIC);  -- becomes NUMERIC(18,6)
INSERT INTO t VALUES (123456789012.123456);  -- max 12 digits before decimal
```

**Migration action:** Always specify explicit precision/scale for NUMERIC columns.

### JSON Compression

DSQL automatically compresses `json` columns. The 1 MiB limit applies to the compressed size, so you can store JSON values significantly larger than 1 MiB as long as they compress below the limit.

To disable compression: use the `STORAGE` keyword in CREATE TABLE or ALTER TABLE.

### JSONB Runtime Behavior

```sql
-- Store as json
CREATE TABLE config (id uuid PRIMARY KEY, data json);

-- Query with JSONB operators (cast at runtime)
SELECT data::jsonb -> 'key' FROM config;           -- extract
SELECT data::jsonb @> '{"a":1}' FROM config;       -- containment
SELECT data::jsonb ? 'key' FROM config;            -- key exists
SELECT jsonb_set(data::jsonb, '{key}', '"val"') FROM config;  -- modify

-- All PostgreSQL JSON/JSONB functions work (section 9.16)
```

### Array Runtime Behavior

Arrays cannot be stored as column types but work at query runtime:

```sql
-- Column definition: use text (or json for structured arrays)
CREATE TABLE t (tags text COLLATE "C");  -- store as comma-separated or json

-- Runtime arrays work in expressions:
SELECT string_to_array('a,b,c', ',');           -- returns {a,b,c}
SELECT array_agg(name) FROM users;              -- returns text[]
SELECT unnest(ARRAY[1,2,3]);                    -- works
SELECT * FROM unnest(ARRAY['a','b']) AS t(val); -- works

-- Alternative: store as json array
CREATE TABLE t (tags json);
INSERT INTO t VALUES ('["tag1","tag2","tag3"]');
SELECT json_array_elements_text(tags) FROM t;   -- unnest equivalent
```

### INET Runtime Behavior

INET is a runtime type — useful for parsing and filtering but not storable:

```sql
-- Store as text
CREATE TABLE connections (ip text COLLATE "C");

-- Cast to inet at query time for network operations
SELECT ip::inet FROM connections WHERE ip::inet << '192.168.0.0/16'::inet;
```

---

## Indexability Rules

### Can be used as index key

smallint, integer, bigint, real, double precision, numeric, char, varchar, bpchar, text, date, time, timestamp, timestamptz, boolean, uuid

### Cannot be used as index key

time with time zone (timetz), interval, bytea, json

### Cannot be indexed at all (runtime-only or TEXT-mapped)

Arrays, INET, JSONB, geometric types, TSVECTOR — these are stored as text and can be indexed as text, but their native operators won't use the index.

---

## Key Differences from dsql-lint

| Type | dsql-lint maps to | This converter maps to | Reason |
|---|---|---|---|
| JSONB | TEXT | json | DSQL docs list json as a supported stored type with all JSON operators |
| JSON | TEXT | json | Same — json is natively supported |
| MONEY | TEXT | numeric(19,4) | Preserves monetary precision |

---

## Migration Decision Matrix

| PostgreSQL Column Type | DSQL Column Type | Data Loss Risk | Action Required |
|---|---|---|---|
| SMALLINT/INTEGER/BIGINT | Same | None | None |
| REAL/DOUBLE PRECISION | Same | None | None |
| NUMERIC(p,s) | numeric(p,s) | None if p≤38, s≤37 | Check precision fits |
| NUMERIC (unbounded) | numeric(18,6) | Possible truncation | Add explicit (p,s) |
| SERIAL/BIGSERIAL | integer/bigint + sequence | None | Wire up sequence |
| CHAR/VARCHAR/TEXT | Same + COLLATE "C" | None (data) | Sort order changes |
| DATE/TIME/TIMESTAMP | Same | None | None |
| TIMESTAMPTZ | timestamptz | None | None |
| INTERVAL | interval | None | Cannot index |
| BOOLEAN | boolean | None | None |
| BYTEA | bytea | None | Cannot index |
| UUID | uuid | None | None |
| JSON | json | None | None |
| JSONB | json | None (operators still work via cast) | Use ::jsonb in queries |
| TEXT[] / INT[] | text | Semantic loss | Restructure to json array or join table |
| INET/CIDR/MACADDR | text | None (string representation) | Cast to inet at query time |
| TSVECTOR/TSQUERY | text | FTS capability lost | Use external search service |
| Geometric types | text | Spatial capability lost | Use external GIS service |
| XML | text | XML functions lost | Parse in application |
| MONEY | numeric(19,4) | None | Explicit precision |
| BIT/VARBIT | text | Bit operations lost | Use application logic |
