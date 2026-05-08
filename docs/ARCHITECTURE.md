# Architecture

## System Overview

The DSQL Schema Converter Skill is a knowledge-only component that augments Kiro's ability to convert PostgreSQL schemas to Aurora DSQL-compatible DDL. It operates entirely within Kiro's context window — no external runtime, no MCP server, no compilation step.

```
┌─────────────────────────────────────────────────────────────────┐
│                         Kiro IDE                                 │
│                                                                  │
│  ┌──────────────┐    ┌──────────────────────────────────────┐   │
│  │  User Chat   │───▶│         Kiro Agent (LLM)             │   │
│  └──────────────┘    │                                      │   │
│                       │  ┌────────────────────────────────┐  │   │
│                       │  │   DSQL Schema Converter Skill  │  │   │
│                       │  │                                │  │   │
│                       │  │  • SKILL.md (entry point)      │  │   │
│                       │  │  • dsql-constraints.md         │  │   │
│                       │  │  • postgresql-type-mappings.md  │  │   │
│                       │  │  • plpgsql-patterns.md         │  │   │
│                       │  │  • extending-dialects.md       │  │   │
│                       │  └────────────────────────────────┘  │   │
│                       │                                      │   │
│                       │  Applies rules to produce:           │   │
│                       │  • Converted DDL                     │   │
│                       │  • Compatibility reports             │   │
│                       │  • Migration guidance                │   │
│                       └──────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Optional: MCP Power (separate)               │   │
│  │  convert_schema | analyze_compatibility | list_type_maps  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. SKILL.md — Entry Point

The main skill definition file. Contains:
- Skill metadata (name, description, keywords)
- Conversion rules summary (quick reference)
- PL/pgSQL pattern catalog (10 patterns)
- Workflow descriptions
- Best practices and troubleshooting

**Activation:** Kiro loads this file when the conversation matches the skill's keywords (DSQL, schema conversion, PostgreSQL migration, type mappings, etc.).

### 2. Reference Files

Detailed knowledge documents loaded as needed:

| File | Purpose | When Referenced |
|---|---|---|
| `dsql-constraints.md` | Platform limits (transactions, types, indexes) | Compatibility analysis, constraint questions |
| `postgresql-type-mappings.md` | Two-stage type pipeline (PG → Normalized → DSQL) | Type conversion, mapping lookups |
| `plpgsql-patterns.md` | 10 transpilation patterns with before/after | Trigger/function conversion |
| `extending-dialects.md` | Architecture for adding MySQL, Spanner, etc. | Contributor guidance |

### 3. Conversion Pipeline (Conceptual)

When Kiro processes a schema conversion request, it follows this logical pipeline:

```
Input PostgreSQL DDL
        │
        ▼
┌─────────────────────┐
│  1. Parse & Classify │  Identify: tables, types, indexes, functions,
│                      │  triggers, sequences, views, extensions
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  2. Type Mapping     │  PostgreSQL type → NormalizedType → DSQL type
│                      │  Add COLLATE "C" to strings
│                      │  SERIAL → integer + sequence
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  3. Constraint       │  ENUM → CHECK constraint
│     Conversion       │  FK → validate_fk_*() function
│                      │  Composite types → dropped
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  4. Index            │  All → CREATE INDEX ASYNC
│     Conversion       │  GIN/GiST/BRIN → btree
│                      │  WHERE → removed
│                      │  INCLUDE → preserved
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  5. PL/pgSQL         │  Match against 10 patterns
│     Transpilation    │  Generate SQL function or CHECK
│                      │  Stub if unconvertible
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  6. Schema Object    │  Materialized view → VIEW
│     Conversion       │  Partitioned → flat
│                      │  Inherited → flat (merge columns)
│                      │  Temp → regular (_tmp_ prefix)
│                      │  Sequences → CACHE 1
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  7. Output           │  DSQL-compatible DDL
│     Generation       │  Conversion summary table
│                      │  Warnings & manual action items
└─────────────────────┘
```

## Type Mapping Architecture

The converter uses a two-stage normalization pipeline:

```
PostgreSQL Types (50+)
        │
        │  Stage 1: Dialect-specific mapping
        ▼
NormalizedTypes (20)
        │
        │  Stage 2: Fixed DSQL mapping
        ▼
DSQL SQL Types (20)
```

**Why two stages?**

1. **Extensibility** — Adding a new source dialect (MySQL, Spanner) only requires Stage 1 mapping
2. **Consistency** — Stage 2 is fixed and shared across all dialects
3. **Validation** — NormalizedTypes define what DSQL can store; anything unmapped falls to TEXT

### NormalizedType Inventory

```
SMALLINT │ INTEGER │ BIGINT │ REAL │ DOUBLE_PRECISION │ NUMERIC
CHAR │ VARCHAR │ BPCHAR │ TEXT
DATE │ TIME │ TIMETZ │ TIMESTAMP │ TIMESTAMPTZ │ INTERVAL
BOOLEAN │ BYTEA │ UUID │ JSON
```

## PL/pgSQL Transpilation Architecture

Pattern matching is intent-based, not syntax-based:

```
PL/pgSQL Function Body
        │
        ▼
┌─────────────────────────────────────────┐
│  Pattern Matcher                         │
│                                          │
│  Signals:                                │
│  • NEW.<col> = <expr>  → SET_COLUMN      │
│  • IF NEW.<col> ...    → VALIDATION      │
│  • INSERT INTO audit   → AUDIT_INSERT    │
│  • UPDATE ... OLD.id   → CASCADE_DML     │
│  • FOR r IN ... LOOP   → FOR_LOOP        │
│  • IF ... RETURN ...   → IF_ELSE         │
│  • EXCEPTION unique_   → UPSERT          │
│  • EXECUTE format(...) → DYNAMIC_SQL     │
│  • CURSOR ... LOOP     → CURSOR          │
│  • EXCEPTION no_data   → COALESCE        │
│  • PERFORM             → UNCONVERTIBLE   │
└─────────────────────────┬───────────────┘
                          │
                          ▼
┌─────────────────────────────────────────┐
│  Generator                               │
│                                          │
│  Produces:                               │
│  • SQL function (LANGUAGE sql)           │
│  • CHECK constraint (Pattern 2)         │
│  • ON CONFLICT clause (Pattern 7)       │
│  • Stub with TODO (unconvertible)       │
└─────────────────────────────────────────┘
```

## Skill vs. Power — Separation of Concerns

| Aspect | Skill (this repo) | Power (separate) |
|---|---|---|
| Runtime | None (loaded into LLM context) | Node.js MCP server |
| Execution | Kiro applies rules in chat | Programmatic tool calls |
| Output | Inline DDL + explanations | Structured JSON reports |
| Dependencies | None | Node.js 18+, dsql-lint (optional) |
| Use case | Interactive conversion, Q&A | CI/CD pipelines, batch processing |
| Installation | Copy folder | MCP server config |

**When to use just the Skill:**
- Interactive schema review and conversion
- Learning DSQL constraints
- One-off migrations
- Team knowledge sharing (commit to repo)

**When to add the Power:**
- Automated CI/CD validation
- Large schema batch processing
- Structured machine-readable output
- Integration with other tools

## File Size & Context Budget

| File | Lines | Purpose |
|---|---|---|
| SKILL.md | ~200 | Always loaded on activation |
| dsql-constraints.md | ~70 | Loaded for constraint questions |
| postgresql-type-mappings.md | ~60 | Loaded for type questions |
| plpgsql-patterns.md | ~250 | Loaded for function conversion |
| extending-dialects.md | ~100 | Loaded for contributor questions |

Total skill footprint: ~680 lines of markdown. Well within Kiro's context budget for a single skill.

## Extension Points

### Adding a Source Dialect

1. Create a new type mapping reference file (`mysql-type-mappings.md`)
2. Add dialect-specific conversion notes to SKILL.md
3. Document any unique patterns (e.g., MySQL's AUTO_INCREMENT, ENUM as column type)

### Adding Conversion Patterns

1. Document the pattern in `plpgsql-patterns.md` (before/after)
2. Add a summary entry in SKILL.md's pattern table
3. Include app-responsibility notes

### Updating DSQL Constraints

As Aurora DSQL evolves, update `dsql-constraints.md` with:
- New supported features (remove conversion rules that are no longer needed)
- Changed limits (transaction row counts, type sizes)
- New data types

## Design Principles

1. **Knowledge over code** — The skill teaches Kiro rules; it doesn't execute logic
2. **Deterministic mappings** — Every PostgreSQL construct has exactly one DSQL output
3. **Explicit over implicit** — Dropped features get comments explaining why
4. **App-layer honesty** — When DSQL can't enforce something, the output says "call from application"
5. **One DDL per transaction** — Output is structured for DSQL's transaction model
6. **Conservative defaults** — CACHE 1 for sequences, btree for all indexes, TEXT for unknowns
