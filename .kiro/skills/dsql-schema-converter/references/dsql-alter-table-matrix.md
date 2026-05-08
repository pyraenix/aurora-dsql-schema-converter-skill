# ALTER TABLE Support Matrix

What ALTER TABLE operations are supported in Aurora DSQL and what's not.

Sources:
- [ALTER TABLE Syntax Support](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/alter-table-syntax-support.html)
- [CREATE TABLE Syntax Support](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/create-table-syntax-support.html)

---

## Supported ALTER TABLE Syntax

```sql
ALTER TABLE [ IF EXISTS ] [ ONLY ] name [ * ]
    action [, ... ]
ALTER TABLE [ IF EXISTS ] [ ONLY ] name [ * ]
    RENAME [ COLUMN ] column_name TO new_column_name
ALTER TABLE [ IF EXISTS ] [ ONLY ] name [ * ]
    RENAME CONSTRAINT constraint_name TO new_constraint_name
ALTER TABLE [ IF EXISTS ] name
    RENAME TO new_name
ALTER TABLE [ IF EXISTS ] name
    SET SCHEMA new_schema
```

### Supported Actions

```sql
ADD [ COLUMN ] [ IF NOT EXISTS ] column_name data_type
    [ STORAGE { PLAIN | EXTERNAL | EXTENDED | MAIN | DEFAULT } ]
ADD table_constraint_using_index
ALTER [ COLUMN ] column_name SET STORAGE { PLAIN | EXTERNAL | EXTENDED | MAIN | DEFAULT }
ALTER [ COLUMN ] column_name { SET GENERATED { ALWAYS | BY DEFAULT } |
    SET sequence_option | RESTART [ [ WITH ] restart ] } [...]
ALTER [ COLUMN ] column_name DROP IDENTITY [ IF EXISTS ]
OWNER TO { new_owner | CURRENT_ROLE | CURRENT_USER | SESSION_USER }
```

### Supported table_constraint_using_index

```sql
[ CONSTRAINT constraint_name ]
UNIQUE USING INDEX index_name
```

---

## Support Matrix

### Column Operations

| Operation | Supported | Notes |
|---|---|---|
| ADD COLUMN | ✅ | With data type and optional STORAGE clause |
| ADD COLUMN IF NOT EXISTS | ✅ | Idempotent column addition |
| DROP COLUMN | ❌ | Not in supported syntax |
| ALTER COLUMN SET DATA TYPE | ❌ | Not in supported syntax |
| ALTER COLUMN SET DEFAULT | ❌ | Not in supported syntax |
| ALTER COLUMN DROP DEFAULT | ❌ | Not in supported syntax |
| ALTER COLUMN SET NOT NULL | ❌ | Not in supported syntax |
| ALTER COLUMN DROP NOT NULL | ❌ | Not in supported syntax |
| ALTER COLUMN SET STORAGE | ✅ | PLAIN, EXTERNAL, EXTENDED, MAIN, DEFAULT |
| ALTER COLUMN SET GENERATED | ✅ | Change identity column generation mode |
| ALTER COLUMN SET sequence_option | ✅ | Modify underlying identity sequence |
| ALTER COLUMN RESTART | ✅ | Reset identity sequence counter |
| ALTER COLUMN DROP IDENTITY | ✅ | Remove identity property |
| RENAME COLUMN | ✅ | Rename a column |

### Constraint Operations

| Operation | Supported | Notes |
|---|---|---|
| ADD CONSTRAINT (CHECK) | ❌ | Not in supported syntax — add at CREATE TABLE time |
| ADD CONSTRAINT (UNIQUE) via index | ✅ | UNIQUE USING INDEX (index must be VALID) |
| ADD CONSTRAINT (PRIMARY KEY) | ❌ | Must be defined at CREATE TABLE |
| ADD CONSTRAINT (FOREIGN KEY) | ❌ | Foreign keys not supported at all |
| DROP CONSTRAINT | ❌ | Not in supported syntax |
| RENAME CONSTRAINT | ✅ | Rename an existing constraint |
| VALIDATE CONSTRAINT | ❌ | Not in supported syntax |

### Table-Level Operations

| Operation | Supported | Notes |
|---|---|---|
| RENAME TO | ✅ | Rename the table |
| SET SCHEMA | ✅ | Move table to different schema |
| OWNER TO | ✅ | Change table owner |
| ADD table_constraint_using_index | ✅ | Add UNIQUE constraint from existing index |

### Not Supported (PostgreSQL features absent from DSQL ALTER TABLE)

| Operation | Alternative |
|---|---|
| DROP COLUMN | Recreate table without the column |
| ALTER COLUMN SET DATA TYPE | Recreate table with new type (or add new column + migrate data) |
| ALTER COLUMN SET/DROP DEFAULT | Recreate table or handle defaults in application |
| ALTER COLUMN SET/DROP NOT NULL | Recreate table |
| ADD CONSTRAINT CHECK | Define at CREATE TABLE time |
| ADD CONSTRAINT PRIMARY KEY | Define at CREATE TABLE time |
| DROP CONSTRAINT | Recreate table without the constraint |
| ADD FOREIGN KEY | Not supported (use validate_fk functions) |
| ENABLE/DISABLE TRIGGER | Triggers not supported |
| ENABLE/DISABLE RULE | Rules not supported |
| SET TABLESPACE | Tablespaces not supported |
| CLUSTER ON | Not applicable (PK-ordered storage) |
| SET WITHOUT CLUSTER | Not applicable |
| SET LOGGED/UNLOGGED | All tables are durable |
| INHERIT/NO INHERIT | Inheritance not supported |
| ATTACH/DETACH PARTITION | Partitioning not supported |
| ALTER COLUMN SET STATISTICS | Not supported (automatic) |
| ALTER COLUMN SET COMPRESSION | Not in supported syntax |
| REPLICA IDENTITY | Not applicable |

---

## Migration Patterns for Unsupported Operations

### Pattern: DROP COLUMN

```sql
-- PostgreSQL
ALTER TABLE users DROP COLUMN legacy_field;

-- DSQL workaround: recreate table
-- 1. Create new table without the column
CREATE TABLE users_new (
  id uuid PRIMARY KEY,
  email varchar(255) COLLATE "C",
  -- legacy_field omitted
  created_at timestamptz
);

-- 2. Copy data
INSERT INTO users_new (id, email, created_at)
SELECT id, email, created_at FROM users;

-- 3. Swap tables
DROP TABLE users;
ALTER TABLE users_new RENAME TO users;

-- 4. Recreate indexes
CREATE INDEX ASYNC idx_users_email ON users (email);
```

### Pattern: ALTER COLUMN TYPE

```sql
-- PostgreSQL
ALTER TABLE orders ALTER COLUMN amount SET DATA TYPE numeric(12,2);

-- DSQL workaround: add new column + migrate
-- 1. Add new column with desired type
ALTER TABLE orders ADD COLUMN amount_new numeric(12,2);

-- 2. Migrate data (in batches if large table)
UPDATE orders SET amount_new = amount::numeric(12,2)
WHERE amount_new IS NULL;
-- (repeat in batches of ≤3000 rows)

-- 3. Application switches to new column
-- 4. Eventually recreate table without old column
```

### Pattern: ADD CHECK CONSTRAINT (post-creation)

```sql
-- PostgreSQL
ALTER TABLE products ADD CONSTRAINT chk_price CHECK (price > 0);

-- DSQL: CHECK must be defined at CREATE TABLE time
-- Workaround: recreate table with constraint, or enforce in application

-- If table is new/empty, just recreate:
DROP TABLE products;
CREATE TABLE products (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  price numeric(10,2) CONSTRAINT chk_price CHECK (price > 0)
);
```

### Pattern: ADD UNIQUE CONSTRAINT (post-creation)

```sql
-- DSQL supports this via existing index:
-- 1. Create the unique index first
CREATE UNIQUE INDEX ASYNC idx_users_email ON users (email);

-- 2. Wait for index to become VALID
-- SELECT indisvalid FROM pg_index WHERE indexrelid = 'idx_users_email'::regclass;

-- 3. Add constraint using the index
ALTER TABLE users ADD CONSTRAINT uq_users_email UNIQUE USING INDEX idx_users_email;
```

---

## Important Notes

1. **One DDL per transaction** — Each ALTER TABLE statement must be in its own transaction
2. **Index must be VALID** — Cannot add UNIQUE constraint using an index that's still building
3. **SET SCHEMA** — Moves table between schemas (useful for the ≤10 schema limit)
4. **OWNER TO** — Required when transferring table ownership between roles
5. **No DROP COLUMN** — This is the most impactful limitation; plan schema carefully upfront
6. **No type changes** — Column types are permanent once created; design types carefully

---

## Pre-Migration Checklist for ALTER TABLE

Before migrating, audit your PostgreSQL migration scripts for ALTER TABLE usage:

```bash
# Find all ALTER TABLE operations in your migration files
grep -rn "ALTER TABLE" migrations/ | grep -v "RENAME\|SET SCHEMA\|OWNER TO\|ADD COLUMN"
```

Any results from the above are potentially unsupported operations that need workarounds.
