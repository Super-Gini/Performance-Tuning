# Flashback Table

## Overview

**Flashback Table** rewinds an entire table (rows and dependent structures like indexes, triggers, constraints) to a past state. Unlike Flashback Query (read only), Flashback Table actually modifies data — it rolls back rows to their prior version. Works within `UNDO_RETENTION`.

Requires **row movement** enabled on the target tables.

## Prerequisites

```sql
ALTER TABLE hr.employees ENABLE ROW MOVEMENT;
```

Row movement allows Oracle to change ROWIDs — required because flashback reinserts rows at potentially new physical locations.

## Syntax

```sql
-- To a timestamp
FLASHBACK TABLE hr.employees
  TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '30' MINUTE);

-- To a specific SCN
FLASHBACK TABLE hr.employees TO SCN 1234567890;

-- To a restore point
FLASHBACK TABLE hr.employees TO RESTORE POINT before_batch;
```

## Multiple Tables Atomically

```sql
FLASHBACK TABLE hr.employees, hr.jobs, hr.job_history
  TO TIMESTAMP (SYSTIMESTAMP - INTERVAL '15' MINUTE);
```

Atomic across the list — all-or-nothing.

## FLASHBACK TABLE TO BEFORE DROP (Recycle Bin)

For dropped tables — restores from the recycle bin. See [Recycle Bin](recycle-bin.md).

```sql
FLASHBACK TABLE hr.employees TO BEFORE DROP;
FLASHBACK TABLE hr.employees TO BEFORE DROP RENAME TO employees_old;
```

## What Gets Flashed Back

- Table rows.
- Indexes on the table.
- Triggers (unless `DISABLE ALL TRIGGERS` clause).
- Constraints (revalidated).
- Referential integrity is preserved when flashback covers all involved tables.

Not flashed back:

- Materialized view logs (may need re-refresh).
- Statistics (may be stale; regather).

## Restore Point

Named SCN marker for easy referencing:

```sql
CREATE RESTORE POINT before_batch;
-- Do risky operation
FLASHBACK TABLE hr.employees TO RESTORE POINT before_batch;

DROP RESTORE POINT before_batch;
```

**Guaranteed restore point** ties into Flashback Database:

```sql
CREATE RESTORE POINT before_upgrade GUARANTEE FLASHBACK DATABASE;
```

Requires Flashback Database enabled.

## Diagnostic Queries

```sql
-- Row movement status
SELECT owner, table_name, row_movement FROM dba_tables
WHERE  owner = 'HR';

-- Restore points
SELECT name, scn, time, guarantee_flashback_database,
       storage_size, database_incarnation#
FROM   v$restore_point;

-- Undo enough for target flashback?
SELECT maxquerylen, tuned_undoretention FROM v$undostat WHERE rownum = 1;
```

## Common Issues

- **`ORA-08189: cannot flashback the table because row movement is not enabled`** — Enable row movement first.
- **`ORA-01466`** — DDL happened between target time and now.
- **`ORA-01555`** — Undo insufficient.
- **Foreign key violation on flashback** — Include child tables in the flashback statement.
- **Trigger side effects** — Consider `DISABLE ALL TRIGGERS` clause to prevent replay effects.

## Best Practices

1. **Enable row movement on tables where flashback is a possibility.**
2. Set `UNDO_RETENTION` ≥ typical incident recovery window.
3. Use **restore points** before risky batches / deployments.
4. Include **all related tables** in a single flashback statement for referential integrity.
5. **Disable triggers** if replay would cause double-processing.
6. Regather statistics on flashed tables.
7. For long-term retention, use **Flashback Data Archive** instead of relying on undo.
8. Test flashback on staging before invoking in production.
9. Communicate with app team — data changing under running queries can cause anomalies.

## Interview Questions

1. **Q:** What does Flashback Table do?
   **A:** Rewinds a table's rows to a past state, using undo.

2. **Q:** Prerequisite?
   **A:** `ENABLE ROW MOVEMENT`.

3. **Q:** Multiple tables?
   **A:** Yes — atomic across the list.

4. **Q:** Retention window?
   **A:** Within `UNDO_RETENTION` for regular flashback; longer via FDA.

5. **Q:** Restore Point?
   **A:** Named SCN marker for flashback reference.

6. **Q:** Guaranteed restore point?
   **A:** Ties to Flashback Database — DB can be flashed back to it.

7. **Q:** FLASHBACK TABLE TO BEFORE DROP?
   **A:** Restores a dropped table from the recycle bin.

## References

- Oracle Database Development Guide 19c — Flashback Table
- MOS Doc ID 258670.1 — Flashback Concepts
- MOS Doc ID 219563.1 — Flashback Table Use Cases
