# MCP Server Setup (Companion Power)

The DSQL Schema Converter Skill has a companion Power that provides programmatic schema conversion via MCP tools. The skill works standalone without the Power — it provides knowledge that Kiro applies directly in conversation. The Power adds automated, structured execution.

---

## Installation

### Option 1: Install via Kiro Powers Panel

1. Open Kiro Powers panel
2. Add the `dsql-schema-converter` power (local path or GitHub URL)
3. The MCP server installs automatically

### Option 2: Manual Setup

1. Clone the Power repo:
   ```bash
   git clone https://github.com/pyraenix/aurora-dsql-content.git
   ```

2. Install dependencies:
   ```bash
   cd aurora-dsql-content/mcp-server
   npm install
   ```

3. Add to `.kiro/settings/mcp.json`:
   ```json
   {
     "mcpServers": {
       "dsql-schema-converter": {
         "command": "node",
         "args": ["<path-to>/aurora-dsql-content/mcp-server/src/index.js"],
         "env": {},
         "disabled": false
       }
     }
   }
   ```

### Prerequisites

- **Node.js 18+**
- No other dependencies required (no database connection needed)

---

## Available Tools

### 1. `convert_schema`

Convert PostgreSQL DDL to Aurora DSQL-compatible DDL.

| Parameter | Required | Description |
|---|---|---|
| `sql` | One of sql/file_path | Raw SQL DDL text to convert |
| `file_path` | One of sql/file_path | Path to a .sql file to convert |
| `source_dialect` | Yes | Source dialect (`"postgresql"`) |

**Returns:** Converted DDL + compatibility report + summary

**Example:**
```
convert_schema({
  sql: "CREATE TABLE users (id SERIAL PRIMARY KEY, data JSONB, tags TEXT[]);",
  source_dialect: "postgresql"
})
```

### 2. `analyze_compatibility`

Check schema for DSQL compatibility without generating DDL.

| Parameter | Required | Description |
|---|---|---|
| `sql` | One of sql/file_path | Raw SQL DDL text |
| `file_path` | One of sql/file_path | Path to a .sql file |
| `source_dialect` | Yes | Source dialect (`"postgresql"`) |

**Returns:** Compatibility report (what would change, no DDL output)

### 3. `list_type_mappings`

Show the complete type mapping table for a source dialect.

| Parameter | Required | Description |
|---|---|---|
| `source_dialect` | Yes | Source dialect (`"postgresql"`) |

**Returns:** Source type → NormalizedType → DSQL type mapping table

### 4. `list_supported_dialects`

List all supported source database dialects.

No parameters required.

**Returns:** Table of supported dialects with status and key conversions

---

## Architecture

```
Input SQL ──► sql-parser.js ──► converter.js ──► DDL Output
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
            type-mappings/   plpgsql-transpiler.js  dsql-constraints.js
            postgresql.js    (10 patterns)          (validation rules)
```

### Source Files

| File | Purpose |
|---|---|
| `src/index.js` | MCP server entry point (tool registration) |
| `src/converter.js` | Main conversion pipeline |
| `src/sql-parser.js` | SQL DDL parser |
| `src/plpgsql-transpiler.js` | PL/pgSQL → SQL function transpilation |
| `src/dsql-constraints.js` | DSQL platform rules and validation |
| `src/dsql-lint.js` | Linting and compatibility checking |
| `src/type-mappings/` | Dialect-specific type mapping files |

---

## How Skill + Power Work Together

```
┌─────────────────────────────────────────────────────┐
│                    Kiro Agent                         │
│                                                      │
│  ┌─────────────────────┐  ┌──────────────────────┐  │
│  │  Skill (knowledge)  │  │  Power (execution)   │  │
│  │                     │  │                      │  │
│  │  • Type mappings    │  │  • convert_schema    │  │
│  │  • PL/pgSQL rules   │  │  • analyze_compat    │  │
│  │  • Constraints      │  │  • list_type_maps    │  │
│  │  • Best practices   │  │  • list_dialects     │  │
│  │  • ORM guides       │  │                      │  │
│  │  • Multi-region     │  │  Structured JSON     │  │
│  │  • Multi-tenant     │  │  output for CI/CD    │  │
│  └─────────────────────┘  └──────────────────────┘  │
│                                                      │
│  Without Power: Kiro applies rules manually in chat  │
│  With Power: Kiro calls tools for structured output  │
└─────────────────────────────────────────────────────┘
```

### Without the Power

- Kiro reads the skill's knowledge files
- Applies conversion rules inline during conversation
- Produces DDL directly in chat
- Works for interactive conversion, Q&A, and one-off migrations

### With the Power

- Kiro calls `convert_schema` for batch/automated conversion
- Gets structured output (DDL + report + warnings)
- Can be used in CI/CD pipelines
- Provides consistent, repeatable output
- Skill knowledge still informs Kiro's explanations and follow-up guidance

---

## CI/CD Integration

Use the Power in your pipeline to validate schema changes:

```bash
# In CI pipeline — fail if unconvertible patterns exist
node mcp-server/src/index.js <<EOF
{"tool": "convert_schema", "args": {"file_path": "db/schema.sql", "source_dialect": "postgresql"}}
EOF
# Check: summary.functions_stubbed == 0
```

---

## Verification

Test that the Power is working:

```bash
cd <path-to>/dsql-schema-converter/mcp-server
node verify-all.js
```

This runs the test pipeline against sample schemas and validates all conversion paths.

---

## Trigger Phrases

The Power activates when conversation mentions: dsql, aurora, postgresql, postgres, schema, migration, convert, ddl, database, sql, migrate, plpgsql
