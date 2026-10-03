# V$SQL

## Purpose

One row per **child cursor** in the shared pool. `V$SQLAREA` is one row per parent (aggregated). For per-plan and per-child troubleshooting, use `V$SQL`.

## Key Columns

| Column                  | Meaning                                   |
| ----------------------- | ----------------------------------------- |
| `SQL_ID`                | 13-char SQL fingerprint.                  |
| `SQL_TEXT`              | First 1000 chars.                         |
| `SQL_FULLTEXT`          | Complete text (CLOB).                     |
| `CHILD_NUMBER`          | Child cursor number.                      |
| `PLAN_HASH_VALUE`       | Execution plan hash.                      |
| `EXECUTIONS`            | Number of runs.                           |
| `PARSE_CALLS`           | Parse count.                              |
| `FETCHES`               | Fetch calls.                              |
| `ROWS_PROCESSED`        | Total rows.                               |
| `BUFFER_GETS`           | Logical reads.                            |
| `DISK_READS`            | Physical reads.                           |
| `CPU_TIME`              | CPU (μs).                                 |
| `ELAPSED_TIME`          | Elapsed (μs).                             |
| `USER_IO_WAIT_TIME`     | User IO waits (μs).                       |
| `CLUSTER_WAIT_TIME`     | RAC waits (μs).                           |
| `APPLICATION_WAIT_TIME` | Application class waits (μs).             |
| `CONCURRENCY_WAIT_TIME` | Concurrency class waits (μs).             |
| `LOADS`                 | Times loaded into shared pool.            |
| `INVALIDATIONS`         | Times invalidated.                        |
| `FIRST_LOAD_TIME`       | First loaded.                             |
| `LAST_ACTIVE_TIME`      | Last executed.                            |
| `MODULE / ACTION`       | Application marker.                       |
| `PARSING_SCHEMA_NAME`   | Schema that parsed it.                    |
| `OPTIMIZER_MODE`        | `ALL_ROWS`, `FIRST_ROWS`, etc.            |
| `IS_BIND_SENSITIVE`     | ACS eligible.                             |
| `IS_BIND_AWARE`         | ACS currently maintaining per-bind plans. |

## Common Queries

```sql
-- Top SQL by CPU in the shared pool right now
SELECT   sql_id, executions,
         ROUND(cpu_time/1e6,2) cpu_secs,
         ROUND(elapsed_time/executions/1e6,3) sec_per_exec,
         buffer_gets, rows_processed
FROM     v$sql
WHERE    executions > 0
ORDER BY cpu_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Top SQL by elapsed
SELECT   sql_id, ROUND(elapsed_time/1e6,2) elapsed_secs,
         executions, ROUND(elapsed_time/executions/1e6,3) sec_per_exec,
         sql_text
FROM     v$sql
WHERE    executions > 0
ORDER BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Full text of a specific SQL
SELECT sql_fulltext FROM v$sql WHERE sql_id = '&sql_id';

-- Multiple children of same parent (why?)
SELECT sql_id, child_number, plan_hash_value, is_bind_sensitive, is_bind_aware
FROM   v$sql
WHERE  sql_id = '&sql_id'
ORDER  BY child_number;

-- Cursors for a session's SQL
SELECT s.sql_id, sq.child_number, sq.plan_hash_value, sq.executions
FROM   v$session s
JOIN   v$sql sq USING (sql_id)
WHERE  s.sid = &target_sid;
```

## `V$SQL_SHARED_CURSOR`

Why do multiple children exist for one SQL_ID? Query this — it has a Y/N column for every reason (bind mismatch, optimizer mismatch, ACS, etc.).

## References

- Oracle Database Reference 19c — `V$SQL`
- MOS Doc ID 296377.1 — Reason for multiple child cursors
