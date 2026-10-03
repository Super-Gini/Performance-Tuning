# Statistics

## Overview

**Optimizer statistics** are the numeric descriptions of data that CBO uses to estimate cost. Every table, index, column, and system-level statistic contributes to a cardinality estimate → a cost → a plan. Wrong statistics = wrong plan = wrong performance.

Oracle collects statistics automatically nightly via the **Automatic Statistics Gathering job** (part of the Auto Maintenance framework). Manual `DBMS_STATS` calls supplement this after bulk data changes.

## Statistics Types

| Type             | What                                                  |
| ---------------- | ----------------------------------------------------- |
| **Table**        | Row count, block count, average row length            |
| **Column**       | NDV (distinct values), density, low/high, histogram   |
| **Index**        | Height, leaf blocks, clustering factor, distinct keys |
| **System**       | I/O and CPU speed — feeds cost model                  |
| **Fixed object** | Stats on X$ tables                                    |
| **Dictionary**   | Stats on SYS objects                                  |
| **Extended**     | Column groups, expression stats                       |

## `DBMS_STATS` — The Only Correct Way

`ANALYZE TABLE ... COMPUTE STATISTICS` is deprecated. Always use `DBMS_STATS`.

```sql
-- Gather stats for a table
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  ownname => 'HR',
  tabname => 'EMPLOYEES',
  estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
  method_opt => 'FOR ALL COLUMNS SIZE AUTO',
  cascade => TRUE,           -- also gather index stats
  no_invalidate => DBMS_STATS.AUTO_INVALIDATE);

-- Gather stats for a schema
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('HR');

-- Gather stats for a database
EXEC DBMS_STATS.GATHER_DATABASE_STATS;

-- Gather system stats (rare; do this once for the platform)
EXEC DBMS_STATS.GATHER_SYSTEM_STATS('START');
-- Run workload for 30-60 minutes
EXEC DBMS_STATS.GATHER_SYSTEM_STATS('STOP');
```

### Key Parameters

- **`estimate_percent`**: `DBMS_STATS.AUTO_SAMPLE_SIZE` (default; uses hash-based algorithm — very fast and accurate).
- **`method_opt`**:
  - `'FOR ALL COLUMNS SIZE AUTO'` — histograms based on usage (default).
  - `'FOR ALL COLUMNS SIZE 1'` — no histograms.
  - `'FOR COLUMNS status SIZE 254'` — histogram on specific column.
- **`cascade`**: TRUE to gather dependent index stats too.
- **`no_invalidate`**:
  - `TRUE` — don't invalidate cursors (bad plans linger).
  - `FALSE` — invalidate immediately (parse storm risk).
  - `AUTO_INVALIDATE` — rolling invalidation over ~5 hours (best of both).

### Automatic Job

Runs during **maintenance window** (weeknights 22:00–02:00, weekends 06:00–06:00). Table qualifies for gather if:

- Table is new / no stats.
- \> 10% of rows have changed since last gather (`DBA_TAB_MODIFICATIONS`).
- Table stats marked `STALE`.

Check status:

```sql
SELECT client_name, status FROM dba_autotask_client;
SELECT * FROM dba_autotask_operation
WHERE  client_name = 'auto optimizer stats collection';
```

## Column Statistics

Columns have:

- **`NUM_DISTINCT`** — distinct value count (NDV)
- **`DENSITY`** — 1/NDV usually
- **`LOW_VALUE` / `HIGH_VALUE`** — min/max (as RAW)
- **`NUM_NULLS`** — NULL count
- **`AVG_COL_LEN`** — average column length
- **Histogram** if present

Query:

```sql
SELECT column_name, num_distinct, density, num_nulls,
       histogram, sample_size, last_analyzed
FROM   dba_tab_col_statistics
WHERE  owner = 'HR' AND table_name = 'EMPLOYEES';
```

## Extended Statistics — Column Groups

For queries with correlated predicates (`gender = 'F' AND pregnant = 'Y'`), single-column stats mislead CBO. Create a column group:

```sql
DECLARE
  cg VARCHAR2(30);
BEGIN
  cg := DBMS_STATS.CREATE_EXTENDED_STATS(
    ownname => 'HR', tabname => 'EMPLOYEES',
    extension => '(GENDER, PREGNANT)');
END;
/

EXEC DBMS_STATS.GATHER_TABLE_STATS('HR','EMPLOYEES',
  method_opt=>'FOR ALL COLUMNS SIZE AUTO FOR COLUMNS (GENDER,PREGNANT) SIZE AUTO');
```

## Expression Statistics

For `WHERE UPPER(name) = 'JOHN'`:

```sql
BEGIN
  DBMS_STATS.CREATE_EXTENDED_STATS(
    ownname => 'HR', tabname => 'EMPLOYEES',
    extension => '(UPPER(name))');
END;
/
```

## Locking Statistics

Freeze stats for a table (prevent auto-gather from overwriting):

```sql
EXEC DBMS_STATS.LOCK_TABLE_STATS('HR','EMPLOYEES');
EXEC DBMS_STATS.UNLOCK_TABLE_STATS('HR','EMPLOYEES');
```

Useful for tables where you have carefully crafted stats or where stats gathering itself is expensive.

## Real-Time Statistics (19c)

Basic column statistics captured inline with DML — reduces stale stats risk between full gathers. Enabled by default when Advanced Compression licensed.

## Statistics Import / Export

Export production stats to test:

```sql
-- Create stats table
EXEC DBMS_STATS.CREATE_STAT_TABLE('HR','MY_STATS');

-- Export
EXEC DBMS_STATS.EXPORT_TABLE_STATS('HR','EMPLOYEES',NULL,'MY_STATS');

-- Move MY_STATS to test DB via Data Pump / DB link

-- Import
EXEC DBMS_STATS.IMPORT_TABLE_STATS('HR','EMPLOYEES',NULL,'MY_STATS');
```

## Pending Statistics (Preview)

Test new stats without exposing to production optimizer:

```sql
-- Enable pending stats
EXEC DBMS_STATS.SET_TABLE_PREFS('HR','EMPLOYEES','PUBLISH','FALSE');

-- Gather (now goes into pending)
EXEC DBMS_STATS.GATHER_TABLE_STATS('HR','EMPLOYEES');

-- Use pending in a session
ALTER SESSION SET optimizer_use_pending_statistics = TRUE;
-- Run test queries

-- Publish when confident
EXEC DBMS_STATS.PUBLISH_PENDING_STATS('HR','EMPLOYEES');

-- Or delete pending
EXEC DBMS_STATS.DELETE_PENDING_STATS('HR','EMPLOYEES');
```

## Diagnostic Queries

```sql
-- Table stats
SELECT owner, table_name, num_rows, blocks, avg_row_len,
       last_analyzed, stale_stats
FROM   dba_tab_statistics
WHERE  owner = 'HR'
ORDER  BY last_analyzed DESC;

-- Missing / stale stats
SELECT owner, table_name, last_analyzed, stale_stats
FROM   dba_tab_statistics
WHERE  (last_analyzed IS NULL OR stale_stats = 'YES')
   AND owner NOT IN ('SYS','SYSTEM')
ORDER  BY last_analyzed NULLS FIRST;

-- Auto-task status
SELECT client_name, status FROM dba_autotask_client
WHERE  client_name LIKE '%stats%';

-- Stats history (for retention/restore)
SELECT stats_update_time
FROM   dba_tab_stats_history
WHERE  table_name = 'EMPLOYEES' AND owner = 'HR'
ORDER  BY stats_update_time DESC;

-- Restore prior stats
EXEC DBMS_STATS.RESTORE_TABLE_STATS('HR','EMPLOYEES', SYSTIMESTAMP - INTERVAL '7' DAY);
```

## Common Issues

- **Missing stats** — CBO uses dynamic sampling; often slower or wrong.
- **Stale stats after big DML** — Manually gather after ETL loads.
- **Histogram missing on skewed data** — Add: `method_opt=>'FOR COLUMNS <col> SIZE 254'`.
- **Wrong `estimate_percent`** — Old code with 5% sample can be way off. Use AUTO_SAMPLE_SIZE.
- **Global temp tables** — 12c+ support **Session-Private Statistics**.
- **`no_invalidate=TRUE`** used everywhere — cursors keep bad plans. Use AUTO_INVALIDATE.

## Best Practices

1. **Trust the automatic job.** Do not disable unless there's a reason.
2. **Manual gather after ETL.**
3. `AUTO_SAMPLE_SIZE` and `SIZE AUTO` almost always correct.
4. Column groups for correlated predicates.
5. Lock stats on volatile tables where the auto-job produces bad snapshots (small samples of skewed data).
6. Never `ANALYZE TABLE ... COMPUTE`.
7. Gather system stats once for a platform.
8. Use pending stats for risky changes.
9. Restore stats when regression: `DBMS_STATS.RESTORE_TABLE_STATS`.
10. Monitor `V$SQL.EXECUTIONS_DELTA` after stats changes.

## Interview Questions

1. **Q:** Why do statistics matter?
   **A:** CBO cardinality estimation drives every plan choice. Bad stats → bad plans.

2. **Q:** `DBMS_STATS.AUTO_SAMPLE_SIZE`?
   **A:** Modern hash-based sampling algorithm — very fast, very accurate. Default from 11g.

3. **Q:** What's a histogram?
   **A:** Distribution of column values — allows CBO to estimate cardinality for skewed data.

4. **Q:** Column groups?
   **A:** Extended stats capturing NDV of a combination of columns — fixes correlated predicate errors.

5. **Q:** Should I set `no_invalidate=TRUE`?
   **A:** Rarely — use AUTO_INVALIDATE for rolling invalidation.

6. **Q:** How do you preview stats?
   **A:** Pending stats: `SET_TABLE_PREFS('PUBLISH','FALSE')`, gather, session `optimizer_use_pending_statistics=TRUE`.

7. **Q:** Restore old stats?
   **A:** `DBMS_STATS.RESTORE_TABLE_STATS('owner','table',timestamp)`.

## References

- Oracle Database SQL Tuning Guide 19c — Managing Optimizer Statistics
- MOS Doc ID 149560.1 — DBMS_STATS Usage
- MOS Doc ID 274529.1 — Column Groups
- MOS Doc ID 754639.1 — AUTO_SAMPLE_SIZE
