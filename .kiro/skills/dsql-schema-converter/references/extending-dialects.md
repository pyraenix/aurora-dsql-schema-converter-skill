# Extending the Schema Converter to New Dialects

Sources:
- [Supported Data Types](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-data-types.html) — defines the 20 NormalizedTypes
- [Supported SQL Features](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-sql-features.html) — DDL/DML support for target

## Architecture

```
Source SQL → dsql-lint (diagnostics) → Parser → Converter (adapter) → DSQL DDL
                                         ↓
                               PL/pgSQL Transpiler (10 patterns)
```

## Adding a New Dialect

### Step 1: Create type mapping file

Create `mcp-server/src/type-mappings/<dialect>.js`:

```javascript
import { NormalizedType } from "../dsql-constraints.js";

export const MY_TYPE_MAPPING = Object.freeze({
  "INT": NormalizedType.INTEGER,
  "BIGINT": NormalizedType.BIGINT,
  "VARCHAR": NormalizedType.VARCHAR,
  "JSON": NormalizedType.JSON,        // DSQL supports json natively
  "GEOMETRY": NormalizedType.TEXT,     // No DSQL equivalent
});

export const MY_FALLBACK_TEXT_TYPES = new Set(["GEOMETRY"]);
export const MY_JSON_TYPES = new Set(["JSON"]);
export const MY_SERIAL_TYPES = new Set(["AUTO_INCREMENT"]);
```

### Step 2: Register in converter.js

```javascript
import { MY_TYPE_MAPPING, MY_FALLBACK_TEXT_TYPES, MY_SERIAL_TYPES } from "./type-mappings/mydialect.js";

const ADAPTERS = {
  postgresql: { ... },
  mydialect: {
    typeMapping: MY_TYPE_MAPPING,
    fallbackTextTypes: MY_FALLBACK_TEXT_TYPES,
    serialTypes: MY_SERIAL_TYPES,
    resolveType: null,
  },
};
```

### Step 3: Update MCP server tool schemas

Add the dialect to the `source_dialect` enum in `index.js`.

### Step 4: Add steering file

Create `steering/<dialect>-conversion.md` with type mapping table.

### Step 5: Update POWER.md

Add dialect to description, keywords, and tool docs.

## Available NormalizedTypes (20)

| NormalizedType | DSQL Type | Indexable |
|---|---|---|
| SMALLINT | smallint | Yes |
| INTEGER | integer | Yes |
| BIGINT | bigint | Yes |
| REAL | real | Yes |
| DOUBLE_PRECISION | double precision | Yes |
| NUMERIC | numeric | Yes |
| CHAR | char | Yes |
| VARCHAR | varchar | Yes |
| BPCHAR | bpchar | Yes |
| TEXT | text | Yes |
| DATE | date | Yes |
| TIME | time | Yes |
| TIMETZ | time with time zone | No |
| TIMESTAMP | timestamp | Yes |
| TIMESTAMPTZ | timestamptz | Yes |
| INTERVAL | interval | No |
| BOOLEAN | boolean | Yes |
| BYTEA | bytea | No |
| UUID | uuid | Yes |
| JSON | json | No |

## Custom Type Resolvers

For parameterized types (Spanner's `STRING(255)`, `ARRAY<INT64>`):

```javascript
export function resolveMyType(rawType) {
  if (/^STRING\(\d+\)$/i.test(rawType)) return NormalizedType.VARCHAR;
  if (/^STRING\(MAX\)$/i.test(rawType)) return NormalizedType.TEXT;
  if (/^ARRAY<.+>$/i.test(rawType)) return NormalizedType.TEXT;
  return null; // unmapped → TEXT with warning
}
```

## Planned Dialects

| Dialect | Key Challenges |
|---|---|
| MySQL | AUTO_INCREMENT, ENUM/SET, DATETIME vs TIMESTAMP, no TIMETZ |
| CockroachDB | STRING type, interleaved tables, GEOGRAPHY/GEOMETRY |
| Spanner | GoogleSQL vs PG dialect, STRING(N)/STRING(MAX), ARRAY<T> |

## Key Design Decisions

1. **JSONB → json**: DSQL supports json as stored type (dsql-lint maps to TEXT)
2. **Partial indexes removed**: WHERE not in CREATE INDEX ASYNC syntax
3. **INCLUDE preserved**: DSQL supports INCLUDE columns
4. **GENERATED STORED preserved**: DSQL supports computed columns
5. **Sequences preserved**: DSQL supports CREATE SEQUENCE with CACHE
