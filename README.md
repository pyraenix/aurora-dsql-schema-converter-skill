# DSQL Schema Converter — Kiro Skill

A Kiro Skill that gives your AI assistant deep knowledge of Aurora DSQL platform constraints and the ability to convert PostgreSQL schemas to DSQL-compatible DDL directly in chat.

## What It Does

When this skill is installed, Kiro can:

- **Convert** PostgreSQL DDL to Aurora DSQL-compatible SQL
- **Analyze** schemas for DSQL compatibility issues
- **Transpile** PL/pgSQL trigger functions to pure SQL functions
- **Generate** foreign key validation functions for app-layer enforcement
- **Advise** on DSQL best practices, type mappings, and migration strategies

The skill activates automatically when your conversation touches DSQL, schema conversion, PostgreSQL migration, or related topics.

## Installation

Copy the skill folder into your project:

```bash
# From this repo root
cp -r .kiro/skills/dsql-schema-converter /path/to/your-project/.kiro/skills/
```

Or clone and copy:

```bash
git clone https://github.com/<your-org>/dsql-schema-converter-skill.git
cp -r dsql-schema-converter-skill/.kiro /path/to/your-project/
```

That's it. Kiro detects the skill on next conversation.

## Requirements

- **Kiro IDE** with Skills support
- No additional dependencies, runtimes, or MCP servers required

## Usage

Just ask Kiro about DSQL conversion in chat. Example prompts:

| Prompt | What happens |
|---|---|
| "Convert this schema to DSQL" | Full DDL conversion with report |
| "What's incompatible with DSQL here?" | Compatibility analysis |
| "How do I handle this trigger in DSQL?" | PL/pgSQL transpilation guidance |
| "Map JSONB to DSQL" | Type mapping explanation |
| "What are DSQL's constraints?" | Platform limitations summary |

### Example

Paste any PostgreSQL DDL:

```sql
CREATE TYPE status AS ENUM ('active', 'inactive');
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  status status DEFAULT 'active',
  data JSONB,
  tags TEXT[],
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_users_data ON users USING gin (data);
```

Ask: "Convert this to DSQL"

Kiro produces DSQL-compatible DDL with:
- ENUM → CHECK constraint
- BIGSERIAL → bigint + sequence (CACHE 1)
- JSONB → json
- TEXT[] → text
- GIN index → btree with CREATE INDEX ASYNC
- COLLATE "C" on string columns

## Companion Power (Optional)

This skill works standalone. For automated batch conversion via MCP tools (`convert_schema`, `analyze_compatibility`, `list_type_mappings`), install the companion [dsql-schema-converter Power](https://github.com/<your-org>/dsql-schema-converter-power).

| Component | Role | Required |
|---|---|---|
| Skill (this repo) | Knowledge, rules, guidance | Yes |
| Power (separate) | MCP execution tools | No |

## Skill Contents

```
.kiro/skills/dsql-schema-converter/
├── SKILL.md                              # Main skill definition
└── references/
    ├── dsql-constraints.md               # Platform limits, IAM auth, collation, error codes
    ├── dsql-function-compatibility.md    # Which PG functions work in DSQL
    ├── dsql-migration-patterns.md        # OCC retry, data batching, FK enforcement, checklist
    ├── extending-dialects.md             # Adding MySQL, Spanner, etc.
    ├── plpgsql-patterns.md               # 10 transpilation patterns
    └── postgresql-type-mappings.md       # Two-stage type pipeline
```

## What Gets Converted

| Source Feature | DSQL Output | Automated |
|---|---|---|
| Data types (30+) | Mapped to 20 NormalizedTypes | ✅ |
| ENUM types | CHECK constraints | ✅ |
| Foreign keys | `validate_fk_*()` SQL functions | ✅ |
| SERIAL/BIGSERIAL | integer/bigint + sequence or identity column | ✅ |
| Indexes (all types) | CREATE INDEX ASYNC (btree) | ✅ |
| PL/pgSQL (10 patterns) | Pure SQL functions | ✅ |
| Triggers | Dropped + replacement functions | ✅ |
| Sequences | Preserved with CACHE 1 or 65536 | ✅ |
| Materialized views | Regular VIEW | ✅ |
| Partitioned tables | Flat tables | ✅ |
| Table inheritance | Flat tables (columns merged) | ✅ |
| Temporary tables | Regular tables (`_tmp_` prefix) | ✅ |
| Computed columns | Preserved (GENERATED STORED) | ✅ |
| Extensions | Dropped with alternatives noted | ✅ |
| Multiple schemas | Flattened with table name prefix | ✅ |
| Roles/GRANT/REVOKE | Dropped (IAM replaces) | ✅ |
| PERFORM / complex ELSIF | Stub with TODO | ⚠️ Manual |
| LISTEN/NOTIFY | Dropped (use SNS/SQS) | ⚠️ Manual |
| Advisory locks | Dropped (use DynamoDB) | ⚠️ Manual |
## Key Design Decisions

1. **JSONB → json** (not TEXT): DSQL supports `json` as a stored type with full operator support
2. **COLLATE "C"** added to all string types: DSQL only supports C collation
3. **One DDL per transaction**: DSQL requirement — output is structured accordingly
4. **FK validation as functions**: Allows app-layer enforcement without database-level FK support
5. **Sequences preserved**: DSQL supports CREATE SEQUENCE natively (CACHE 1 required)
6. **Identity columns preferred**: Cleaner than SERIAL + explicit sequence for new tables
7. **OCC retry is mandatory**: Every write can fail with 40001 — not optional error handling
8. **Multi-schema → prefix**: Simple, predictable flattening strategy
9. **IAM over roles**: DSQL authentication is AWS-native, not PostgreSQL-native

## Contributing

1. Fork this repo
2. Edit files in `.kiro/skills/dsql-schema-converter/`
3. Test by installing in a Kiro project and running conversion prompts
4. Submit a PR with before/after examples

### Adding a New Dialect

See [extending-dialects.md](.kiro/skills/dsql-schema-converter/references/extending-dialects.md) for the architecture and step-by-step guide.

## License

MIT

## References

### AWS Official Documentation

- [Aurora DSQL User Guide](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/)
- [Supported Data Types](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-data-types.html)
- [Supported SQL Features](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-sql-features.html)
- [Migration Guide](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-migration-guide.html)
- [Quotas and Limits](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/CHAP_quotas.html)
- [Considerations](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/considerations.html)
- [Concurrency Control (OCC)](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-concurrency-control.html)
- [Sequences and Identity Columns](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/sequences-identity-columns.html)
- [Database Roles and IAM Authentication](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/using-database-and-iam-roles.html)
- [Asynchronous Indexes](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-indexes.html)

### AWS Blog Posts

- [DSQL SQL Dialect: How Aurora DSQL differs from single-instance PostgreSQL](https://aws.amazon.com/blogs/database/dsql-sql-dialect-how-amazon-aurora-dsql-differs-from-single-instance-postgresql/) (Apr 2026)
- [Securing Amazon Aurora DSQL: Access Control Best Practices](https://aws.amazon.com/blogs/database/securing-amazon-aurora-dsql-access-control-best-practices/)

### PostgreSQL Reference

- [PostgreSQL 16 Data Types](https://www.postgresql.org/docs/16/datatype.html)
- [PostgreSQL 16 JSON Functions](https://www.postgresql.org/docs/current/functions-json.html)
- [PostgreSQL 16 PL/pgSQL](https://www.postgresql.org/docs/16/plpgsql.html)

### Kiro

- [Kiro Skills Documentation](https://kiro.dev/docs/skills)

### Full Reference Index

See [.kiro/skills/dsql-schema-converter/references/REFERENCES.md](.kiro/skills/dsql-schema-converter/references/REFERENCES.md) for the complete source mapping showing which official docs informed each skill file.
