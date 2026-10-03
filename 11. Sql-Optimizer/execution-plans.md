# Execution Plans

## Overview

An **execution plan** is a tree of operations the optimizer chose to execute a SQL statement. Every row in a plan shows: the operation (join method, access path), the target object, estimated cardinality, estimated cost, and — at runtime — actual row counts. Reading plans is the single most important skill for SQL tuning.

## Getting a Plan

### 1. Estimated plan (before execution)

```sql
EXPLAIN PLAN FOR SELECT * FROM hr.employees WHERE department_id = 90;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

Note: shows the optimizer's _guess_. Reality can differ (bind peeking, adaptive plans).

### 2. Actual plan of a running / completed cursor

```sql
-- If you know the sql_id
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', NULL,
  'ALLSTATS LAST +ADAPTIVE'));
```

`ALLSTATS LAST` requires `STATISTICS_LEVEL='ALL'` at session level or `/*+ GATHER_PLAN_STATISTICS */` hint on the SQL — collects per-operation runtime stats.

### 3. Real-Time SQL Monitoring (EE + Tuning Pack)

```sql
SELECT DBMS_SQLTUNE.REPORT_SQL_MONITOR(
  sql_id => '&sql_id',
  type => 'TEXT') FROM dual;
```

Best for long-running SQL — shows time-per-operation live.

### 4. Historical plan from AWR

```sql
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_AWR('&sql_id', null, null,
  'ALL +ADAPTIVE'));
```

### 5. From tkprof (via 10046 trace)

See [SQL Trace](../36-sql-tuning/sql-trace.md).

## Reading a Plan

```
-----------------------------------------------------------------------------------
| Id | Operation                        | Name      | Rows  | Bytes | Cost | Time |
-----------------------------------------------------------------------------------
|  0 | SELECT STATEMENT                 |           |       |       |    3 |      |
|  1 |  NESTED LOOPS                    |           |    10 |   210 |    3 | 00:01|
|* 2 |   INDEX RANGE SCAN               | EMP_DEPT  |    10 |   130 |    1 | 00:01|
|  3 |   TABLE ACCESS BY INDEX ROWID    | EMPLOYEES |     1 |     8 |    1 | 00:01|
-----------------------------------------------------------------------------------

Predicate Information (identified by operation id):
   2 - access("DEPARTMENT_ID"=90)
```

Read **inside-out, top-down**:

1. **INDEX RANGE SCAN on EMP_DEPT** — first operation. Finds ROWIDs.
2. **TABLE ACCESS BY INDEX ROWID on EMPLOYEES** — for each ROWID, fetch the row.
3. **NESTED LOOPS** — combines (drives the loop).
4. **SELECT STATEMENT** — root.

Key columns:

- **Rows** — estimated cardinality.
- **Cost** — optimizer's arbitrary cost unit.
- **A-Rows** (with `ALLSTATS LAST`) — actual rows.
- **A-Time** — actual time per operation.
- **Buffers** — logical reads per op.
- **Predicate Information** — WHERE clauses at each step.

## Estimated vs Actual (the Key Diagnostic)

```sql
ALTER SESSION SET STATISTICS_LEVEL = 'ALL';
SELECT /*+ GATHER_PLAN_STATISTICS */ ...
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

Look for **E-Rows vs A-Rows** — if orders of magnitude off, cardinality misestimate is your root cause. Fix statistics or the SQL.

## Adaptive Plan Indicator

Plan shows `- adaptive` marker when adaptive plans engaged. Both original and fallback plans visible with `+ADAPTIVE`.

## Common Operations to Recognize

| Op                                                              | Meaning                       |
| --------------------------------------------------------------- | ----------------------------- |
| `TABLE ACCESS FULL`                                             | Full table scan               |
| `TABLE ACCESS BY INDEX ROWID`                                   | Rows fetched via ROWID        |
| `INDEX UNIQUE SCAN`                                             | Single-row PK / UNIQUE lookup |
| `INDEX RANGE SCAN`                                              | Range lookup                  |
| `INDEX FAST FULL SCAN`                                          | Read all of index, no order   |
| `INDEX FULL SCAN`                                               | Read index in order           |
| `INDEX SKIP SCAN`                                               | Skip leading column values    |
| `NESTED LOOPS`                                                  | Outer × inner                 |
| `HASH JOIN`                                                     | Build/probe                   |
| `SORT MERGE JOIN`                                               | Sort both, merge              |
| `HASH GROUP BY` / `SORT GROUP BY`                               | Aggregation                   |
| `HASH UNIQUE` / `SORT UNIQUE`                                   | DISTINCT                      |
| `PX COORDINATOR` / `PX SEND/RECEIVE`                            | Parallel exec                 |
| `PARTITION RANGE`/`LIST`/`HASH` (+ `SINGLE`, `ITERATOR`, `ALL`) | Partition pruning             |
| `INLIST ITERATOR`                                               | IN-list evaluation            |
| `FILTER`                                                        | Post-filter                   |
| `VIEW` / `VIEW PUSHED PREDICATE`                                | View decomposition            |
| `MERGE` / `MERGE JOIN CARTESIAN`                                | Cartesian (often unintended)  |

## Diagnostic Queries

```sql
-- Find sql_id from text
SELECT sql_id, plan_hash_value, executions,
       elapsed_time/1000000/DECODE(executions,0,1,executions) AS avg_sec,
       buffer_gets/DECODE(executions,0,1,executions) AS avg_gets,
       sql_text
FROM   v$sql
WHERE  sql_text LIKE '%CUSTOMERS%'
   AND upper(sql_text) NOT LIKE '%V$SQL%'
ORDER  BY elapsed_time DESC
FETCH FIRST 10 ROWS ONLY;

-- All plans currently in memory for a SQL
SELECT plan_hash_value, child_number, executions, elapsed_time
FROM   v$sql
WHERE  sql_id = '&sql_id';

-- Historical plan changes
SELECT plan_hash_value, MIN(snap_id) AS first_seen, MAX(snap_id) AS last_seen
FROM   dba_hist_sqlstat
WHERE  sql_id = '&sql_id'
GROUP  BY plan_hash_value
ORDER  BY last_seen DESC;

-- Compare execution stats per plan
SELECT plan_hash_value,
       SUM(executions_delta) AS execs,
       SUM(elapsed_time_delta)/GREATEST(SUM(executions_delta),1) AS avg_us,
       SUM(buffer_gets_delta)/GREATEST(SUM(executions_delta),1) AS avg_gets
FROM   dba_hist_sqlstat
WHERE  sql_id = '&sql_id'
GROUP  BY plan_hash_value
ORDER  BY avg_us;
```

## Common Issues

- **Estimated vs Actual off by 10× or more** — Cardinality issue.
- **NL where HJ would be better** — Under-estimated row count.
- **HJ where NL would be better** — Over-estimated row count on outer.
- **Sort spilling to disk** — See `direct path write temp`.
- **Adaptive plan bailout** — Plan started NL, switched to HJ mid-execution — check "Note" section.
- **Partition pruning not happening** — Predicate on non-partition column, or function around partition column.

## Best Practices

1. Get **actual** plans, not just `EXPLAIN PLAN`.
2. Always use `ALLSTATS LAST` + `GATHER_PLAN_STATISTICS`.
3. Trust A-Rows over E-Rows.
4. Read plans inside-out for join order intuition.
5. Watch for cartesian, unpartitioned full scans, and index skip scans (often signals).
6. Compare plans over time with `DBA_HIST_SQL_PLAN` — plan flips are diagnostic.
7. Use SQL Monitor for long-running SQL.
8. Save known-good plans as SPM baselines.

## Interview Questions

1. **Q:** How do you get the actual execution plan?
   **A:** `DBMS_XPLAN.DISPLAY_CURSOR('sql_id',NULL,'ALLSTATS LAST +ADAPTIVE')` with `STATISTICS_LEVEL=ALL` or `GATHER_PLAN_STATISTICS` hint.

2. **Q:** E-Rows vs A-Rows?
   **A:** Estimated by CBO vs actual. Large discrepancy = cardinality misestimate = probable bad plan.

3. **Q:** How do you read a plan?
   **A:** Inside-out — leaf operations execute first; parents combine their outputs.

4. **Q:** What is SQL Monitor?
   **A:** Real-Time SQL Monitoring showing live per-operation time for long-running or parallel SQL. EE + Tuning Pack.

5. **Q:** Difference between `TABLE ACCESS FULL` and `INDEX FAST FULL SCAN`?
   **A:** Full: scans the table. Index Fast Full: scans the index (much smaller), useful when the index covers the columns needed.

## References

- Oracle Database SQL Tuning Guide 19c — Execution Plans
- MOS Doc ID 235530.1 — DBMS_XPLAN Usage
- MOS Doc ID 262687.1 — SQL Monitoring
