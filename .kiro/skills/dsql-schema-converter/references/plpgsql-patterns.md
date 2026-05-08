# PL/pgSQL Transpilation Patterns

The dsql-schema-converter recognizes 10 common PL/pgSQL patterns and generates equivalent pure SQL functions that work in Aurora DSQL.

Sources:
- [Migration Guide — Application-level logic](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-migration-guide.html)
- [PostgreSQL 16 PL/pgSQL](https://www.postgresql.org/docs/16/plpgsql.html)
- [Supported SQL Features — CREATE FUNCTION](https://docs.aws.amazon.com/aurora-dsql/latest/userguide/working-with-postgresql-compatibility-supported-sql-features.html)

## Pattern 1: SET_COLUMN

**Intent:** Set a column value on UPDATE (e.g., updated_at timestamp)

**Before (PL/pgSQL trigger function):**
```sql
CREATE FUNCTION set_updated_at() RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg BEFORE UPDATE ON users
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

**After (SQL function — call from application):**
```sql
CREATE FUNCTION apply_set_updated_at_users(p_id bigint) RETURNS void
LANGUAGE sql AS $$
  UPDATE users SET updated_at = now() WHERE id = p_id;
$$;
-- Call: SELECT apply_set_updated_at_users(123) after every UPDATE
```

**App responsibility:** Call after every UPDATE on the table.

---

## Pattern 2: VALIDATION → CHECK Constraint

**Intent:** Reject invalid data on INSERT/UPDATE

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION validate_price() RETURNS TRIGGER AS $$
BEGIN
  IF NEW.price < 0 THEN
    RAISE EXCEPTION 'price must be positive';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**After (CHECK constraint — automatic enforcement):**
```sql
ALTER TABLE products ADD CONSTRAINT chk_products_price CHECK (price >= 0);
```

**App responsibility:** None — CHECK is enforced automatically by DSQL.

---

## Pattern 3: AUDIT_INSERT

**Intent:** Log changes to an audit table

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION log_change() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (table_name, action, old_data, new_data)
  VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL function):**
```sql
CREATE FUNCTION audit_log_change(p_operation text, p_old_data text, p_new_data text) RETURNS void
LANGUAGE sql AS $$
  INSERT INTO audit_log (table_name, action, old_data, new_data)
  VALUES ('orders', p_operation, p_old_data, p_new_data);
$$;
-- Call: SELECT audit_log_change('UPDATE', old_json, new_json) after DML
```

**App responsibility:** Call after INSERT/UPDATE/DELETE with the operation and data.

---

## Pattern 4: CASCADE_DML

**Intent:** Update/delete related rows when a parent changes

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION cascade_cancel() RETURNS TRIGGER AS $$
BEGIN
  UPDATE orders SET status = 'cancelled' WHERE user_id = OLD.id;
  RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL function):**
```sql
CREATE FUNCTION cascade_cascade_cancel_users(p_id bigint) RETURNS void
LANGUAGE sql AS $$
  UPDATE orders SET status = 'cancelled' WHERE user_id = p_id;
$$;
-- Call: SELECT cascade_cascade_cancel_users(user_id) before DELETE
```

**App responsibility:** Call before DELETE on the parent table.

---

## Pattern 5: FOR_LOOP → Set-Based

**Intent:** Process rows one at a time (batch update)

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION expire_old() RETURNS void AS $$
DECLARE r RECORD;
BEGIN
  FOR r IN SELECT id FROM tickets WHERE due_date < CURRENT_DATE AND NOT resolved
  LOOP
    UPDATE tickets SET resolved = TRUE WHERE id = r.id;
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL — single set-based statement):**
```sql
CREATE FUNCTION batch_expire_old() RETURNS void
LANGUAGE sql AS $$
  UPDATE tickets SET resolved = TRUE
  FROM (SELECT id FROM tickets WHERE due_date < CURRENT_DATE AND NOT resolved) AS _src
  WHERE tickets.id = _src.id;
$$;
```

**App responsibility:** None — call the function directly. Much faster than row-by-row.

---

## Pattern 6: IF_ELSE → CASE WHEN

**Intent:** Return different values based on a condition

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION get_priority(sev text) RETURNS text AS $$
BEGIN
  IF sev = 'critical' THEN RETURN 'P0';
  ELSE RETURN 'Normal';
  END IF;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL with CASE WHEN):**
```sql
CREATE FUNCTION conditional_get_priority() RETURNS text
LANGUAGE sql AS $$
  SELECT CASE WHEN sev = 'critical' THEN 'P0' ELSE 'Normal' END;
$$;
```

**App responsibility:** None — pure SQL function.

---

## Pattern 7: EXCEPTION unique_violation → ON CONFLICT

**Intent:** Insert or update (upsert)

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION upsert_pref(p_user UUID, p_key TEXT, p_val TEXT) RETURNS void AS $$
BEGIN
  INSERT INTO prefs (user_id, key, value) VALUES (p_user, p_key, p_val);
  EXCEPTION WHEN unique_violation THEN
    UPDATE prefs SET value = p_val WHERE user_id = p_user AND key = p_key;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL with ON CONFLICT):**
```sql
CREATE FUNCTION upsert_upsert_pref(p_user_id text, p_key text, p_value text) RETURNS void
LANGUAGE sql AS $$
  INSERT INTO prefs (user_id, key, value) VALUES (p_user_id, p_key, p_value)
  ON CONFLICT DO NOTHING;
$$;
```

**App responsibility:** None — ON CONFLICT is handled by DSQL.

---

## Pattern 8: Dynamic SQL → Expanded Per-Table

**Intent:** Run same DML on different tables passed as parameter

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION cleanup(tbl text, days integer) RETURNS void AS $$
BEGIN
  EXECUTE format('DELETE FROM %I WHERE created_at < now() - interval ''%s days''', tbl, days);
END;
$$ LANGUAGE plpgsql;
```

**After (one concrete function per table):**
```sql
-- For table 'orders':
CREATE FUNCTION cleanup_orders(p_value text) RETURNS void
LANGUAGE sql AS $$
  DELETE FROM orders WHERE created_at < now() - interval 'p_value days';
$$;

-- For table 'logs':
CREATE FUNCTION cleanup_logs(p_value text) RETURNS void
LANGUAGE sql AS $$
  DELETE FROM logs WHERE created_at < now() - interval 'p_value days';
$$;
-- Call the specific function from application code
```

**App responsibility:** Call the table-specific function instead of the generic one.

---

## Pattern 9: CURSOR → Set-Based

**Intent:** Process query results row by row

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION notify_users() RETURNS void AS $$
DECLARE
  cur CURSOR FOR SELECT id FROM users WHERE status = 'active' AND last_ip IS NULL;
  rec RECORD;
BEGIN
  FOR rec IN cur LOOP
    INSERT INTO notifications (user_id, message) VALUES (rec.id, 'Update profile');
  END LOOP;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL — INSERT...SELECT):**
```sql
CREATE FUNCTION cursor_replacement_notify_users() RETURNS void
LANGUAGE sql AS $$
  INSERT INTO notifications (user_id, message)
  SELECT id, 'Update profile' FROM users WHERE status = 'active' AND last_ip IS NULL;
$$;
```

**App responsibility:** None — single statement, much faster.

---

## Pattern 10: EXCEPTION no_data_found → COALESCE

**Intent:** Return NULL instead of raising error when no rows found

**Before (PL/pgSQL):**
```sql
CREATE FUNCTION safe_lookup(p_id integer) RETURNS text AS $$
DECLARE result text;
BEGIN
  SELECT name INTO result FROM orgs WHERE id = p_id;
  RETURN result;
  EXCEPTION WHEN no_data_found THEN RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

**After (SQL with COALESCE):**
```sql
CREATE FUNCTION safe_safe_lookup() RETURNS void
LANGUAGE sql AS $$
  SELECT COALESCE((SELECT name FROM orgs WHERE id = p_id), NULL);
$$;
```

**App responsibility:** None — COALESCE handles the NULL case.

---

## Unconvertible Patterns

These produce stubs with the original logic as TODO comments:

| Pattern | Why it can't be converted |
|---|---|
| PERFORM | Calls a function for side effects only, discards result. No SQL equivalent. |
| Complex ELSIF (3+ branches) | Too many conditional paths for automated CASE WHEN conversion. |

**Resolution:** Rewrite manually using CASE WHEN, move to application code, or use AWS Lambda.
