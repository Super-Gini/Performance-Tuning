# Top SQL

## Overview

Every performance problem eventually points at specific SQL statements. Finding them is easy; identifying the right dimension (Elapsed? CPU? Gets? Executions?) is the key. This page covers the queries that surface top SQL and the analytical patterns for interpreting them.

## Dimensions

| Dimension               | Use When                               |
| ----------------------- | -------------------------------------- |
| **Elapsed Time**        | Any tuning; global view                |
| **CPU Time**            | CPU-bound diagnosis                    |
| **Buffer Gets**         | Memory pressure, latch contention      |
| **Physical Reads**      | I/O bottleneck                         |
| **Executions**          | Anti-pattern detection (per-row loops) |
| **Rows Processed**      | Efficiency ratio                       |
| **Version Count**       | Cursor bloat (bind vs literal)         |
| **Direct Reads/Writes** | Full scans / parallel activity         |

## Top-SQL Queries

### Right now (in-memory)

```sql
SELECT sql_id, plan_hash_value, executions,
       ROUND(elapsed_time/1e6/DECODE(executions,0,1,executions), 2) AS avg_sec,
       ROUND(cpu_time/1e6/DECODE(executions,0,1,executions), 2) AS avg_cpu_sec,
       buffer_gets/DECODE(executions,0,1,executions) AS avg_gets,
       disk_reads/DECODE(executions,0,1,executions) AS avg_reads,
       rows_processed/DECODE(executions,0,1,executions) AS avg_rows,
       SUBSTR(sql_text, 1, 60) AS sql_text
FROM   v$sql
WHERE  executions > 0
ORDER  BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;
```

### From AWR (historical)

```sql
SELECT sql_id, plan_hash_value,
       SUM(executions_delta) AS execs,
       ROUND(SUM(elapsed_time_delta)/1e6/GREATEST(SUM(executions_delta),1), 2) AS avg_sec,
       ROUND(SUM(cpu_time_delta)/1e6/GREATEST(SUM(executions_delta),1), 2) AS avg_cpu_sec,
       ROUND(SUM(buffer_gets_delta)/GREATEST(SUM(executions_delta),1)) AS avg_gets,
       ROUND(SUM(disk_reads_delta)/GREATEST(SUM(executions_delta),1)) AS avg_reads
FROM   dba_hist_sqlstat
WHERE  snap_id BETWEEN 1000 AND 1100
   AND parsing_schema_name = 'HR'
GROUP  BY sql_id, plan_hash_value
ORDER  BY SUM(elapsed_time_delta) DESC
FETCH FIRST 20 ROWS ONLY;
```

### High-execution (potential loop / N+1)

```sql
SELECT sql_id, executions,
       ROUND(elapsed_time/1e6, 1) AS total_sec,
       ROUND(elapsed_time/1e6/executions*1000, 2) AS avg_ms,
       SUBSTR(sql_text, 1, 60) AS sql
FROM   v$sql
WHERE  executions > 10000
ORDER  BY executions DESC
FETCH FIRST 20 ROWS ONLY;
```

### High version count (cursor bloat)

```sql
SELECT sql_id, version_count, loaded_versions,
       first_load_time, invalidations,
       SUBSTR(sql_text, 1, 60) AS sql
FROM   v$sqlarea
WHERE  version_count > 10
ORDER  BY version_count DESC
FETCH FIRST 20 ROWS ONLY;

-- Why did versions increase?
SELECT sql_id, address, child_number,
       unbound_cursor, sql_type_mismatch, bind_mismatch,
       language_mismatch, optimizer_mismatch, outline_mismatch,
       stats_row_mismatch, literal_mismatch, load_optimizer_stats,
       bind_uacs_diff, plsql_cmp_switchs_diff, insuff_privs,
       force_hard_parse
FROM   v$sql_shared_cursor
WHERE  sql_id = '&sql_id';
```

### Efficiency (rows returned vs gets)

```sql
SELECT sql_id, executions, rows_processed,
       ROUND(rows_processed/DECODE(executions,0,1,executions)) AS rows_per_exec,
       ROUND(buffer_gets/DECODE(rows_processed,0,1,rows_processed)) AS gets_per_row,
       SUBSTR(sql_text, 1, 60) AS sql
FROM   v$sql
WHERE  rows_processed > 0
ORDER  BY gets_per_row DESC
FETCH FIRST 20 ROWS ONLY;
```

`gets_per_row > 100` often indicates a missing index or bad plan.

## Interpreting Ratios

- **`avg_sec / avg_cpu_sec`** — Close to 1 → CPU-bound. Much higher → wait-bound.
- **`avg_gets / avg_rows`** — > 100 usually means bad plan.
- **`avg_reads / avg_gets`** — > 5% → cache miss ratio matters.
- **`version_count`** — > 10 → shared pool bloat; investigate `V$SQL_SHARED_CURSOR`.

## Per-User / Per-Module Attribution

```sql
-- Top SQL by user
SELECT parsing_schema_name AS "user",
       SUM(elapsed_time)/1e6 AS total_sec
FROM   v$sql
GROUP  BY parsing_schema_name
ORDER  BY 2 DESC;

-- Top SQL by module (application-set)
SELECT sql_id, module, action, executions,
       ROUND(elapsed_time/1e6/executions, 2) AS avg_sec
FROM   v$sql
WHERE  module IS NOT NULL
ORDER  BY elapsed_time DESC
FETCH FIRST 20 ROWS ONLY;
```

Encourage apps to set `module` and `action` via `DBMS_APPLICATION_INFO` for exactly this reason.

## ASH-Based Top SQL

More precise for time-based windows:

```sql
SELECT sql_id, plan_hash_value, COUNT(*) AS samples,
       ROUND(COUNT(*)/60, 1) AS approx_minutes,
       COUNT(DISTINCT session_id) AS sessions
FROM   v$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:00' AND TIMESTAMP '2026-08-06 15:00'
GROUP  BY sql_id, plan_hash_value
ORDER  BY samples DESC
FETCH FIRST 10 ROWS ONLY;
```

Samples × session count → total DB time.

## Common Findings

- **1 SQL, 90% of Elapsed** — Fix it first; huge payoff.
- **Many small executions dominate** — Loop / N+1 anti-pattern; batch the workload.
- **High `version_count`** — Bind mismatch or literal SQL. See `V$SQL_SHARED_CURSOR`.
- **High `gets_per_row`** — Bad plan; missing index.
- **Same SQL, different plans** — Plan flip; consider SPM baseline.

## Best Practices

1. **Rank by ELAPSED** first, then investigate why.
2. Correlate top SQL to plans (via `V$SQL_PLAN`, `V$SQL_MONITOR`).
3. Set `module` and `action` in application (`DBMS_APPLICATION_INFO.SET_MODULE`).
4. Compare current top SQL to AWR baseline for regression detection.
5. Focus on **fewer, bigger** wins rather than many tiny tunings.
6. Look at **executions × avg_ms** — many short SQLs can total more than one long one.
7. For version bloat, fix binds; for plan flip, consider baselines.

## Interview Questions

1. **Q:** How do you find top SQL right now?
   **A:** `SELECT * FROM v$sql ORDER BY elapsed_time DESC FETCH FIRST 20 ROWS ONLY;`.

2. **Q:** Historical top SQL?
   **A:** `DBA_HIST_SQLSTAT` with snap_id range.

3. **Q:** What does high `version_count` mean?
   **A:** Multiple child cursors — usually bind mismatch, literal SQL, or optimizer environment differences.

4. **Q:** How to find why child cursors don't share?
   **A:** `V$SQL_SHARED_CURSOR` — each `Y` column indicates a mismatch reason.

5. **Q:** `gets_per_row`?
   **A:** Buffer gets per row returned; > 100 often means bad plan.

6. **Q:** Time-based top SQL for a specific window?
   **A:** ASH: `V$ACTIVE_SESSION_HISTORY` with `sample_time BETWEEN ...`.

## References

- Oracle Database Performance Tuning Guide 19c — Identifying Top SQL
- MOS Doc ID 244268.1 — Diagnosing SQL Performance
- MOS Doc ID 353058.1 — V$SQL_SHARED_CURSOR
