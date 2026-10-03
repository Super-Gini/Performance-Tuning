# Case: Reading an AWR Report End-to-End

## Setup

- Production 19c EE on Oracle Linux 8, `r5.8xlarge` EC2 (32 vCPU, 256 GB RAM).
- SGA_TARGET 96 GB, PGA 32 GB.
- ~800 concurrent app-tier connections.
- Complaint: "App is 30% slower this week."

## Approach

Generate AWR reports for the "good" and "bad" weeks, compare.

```sql
-- Find snapshots for a bad hour
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time BETWEEN
       TO_TIMESTAMP('2026-08-05 10:00','YYYY-MM-DD HH24:MI')
   AND TO_TIMESTAMP('2026-08-05 11:00','YYYY-MM-DD HH24:MI')
ORDER  BY snap_id;

-- Same for a good hour, one week earlier
-- Generate reports
SELECT DBMS_WORKLOAD_REPOSITORY.AWR_DIFF_REPORT_HTML(
       (SELECT dbid FROM v$database), 1, &good_start, &good_end,
       (SELECT dbid FROM v$database), 1, &bad_start, &bad_end)
FROM   dual;
```

## Reading the Comparison

Sections to focus on, in order:

### 1. Elapsed time / DB Time

```
                        Good Snap      Bad Snap
Elapsed (min):          60.0            60.0
DB Time (min):          485.7           712.3   (+47%)
```

DB Time up 47% for the same wall clock. Same load or heavier load taking longer.

### 2. Top Wait Events

```
                        Good %          Bad %
CPU time                42%             28%
db file sequential rd   28%             35%
log file sync            9%             12%
gc buffer busy release   6%             9%
```

CPU% down (proportion), IO up, `log file sync` up, `gc buffer busy release` up. Signs of storage pressure and RAC hot block.

### 3. Load Profile

```
                        Good            Bad
Redo size (MB/s):       12.5            18.7
Logical reads/s:        520k            890k
Physical reads/s:       8.2k            15.4k
User calls/s:           3200            3450
Executes/s:             8500            9100
```

- User calls up modestly (+8%).
- Executes up modestly (+7%).
- **Logical reads +71%**, **Physical reads +88%** — this is the big signal.

Something is reading way more blocks per call.

### 4. Top SQL by DB Time

```
% DB Time    Executions    Exec/DB Time (ms)    SQL_ID
15.2%        45,000        2400                 87ac...
12.8%        890            85,000              9k12...
8.9%         2,200,000     3.9                  x9zx...
```

`87ac` has moderate execution count with high time — investigate.
`9k12` is few-execution but big — probably a report.
`x9zx` is high-frequency small; probably fine.

### 5. Instance Efficiency

```
                        Good %          Bad %
Buffer Hit:             99.4            97.8
Library Hit:            99.8            99.7
Cursor Cache Hit:       80.2            81.1
Soft Parse:             98.4            98.3
```

Buffer Hit dropped from 99.4 → 97.8 = more disk reads. Correlates with physical reads spike.

### 6. Segment-Level Reads

```
Segment           Owner    Type     Physical Reads    % of Total
ORDERS_HIST       APP      TABLE    2.4 M              38%
ORDERS_HIST_PK    APP      INDEX    1.2 M              19%
```

`ORDERS_HIST` = 57% of physical reads. Table + PK. Something is scanning this.

### 7. Get the SQL

```sql
SELECT sql_id, plan_hash_value, executions, buffer_gets/executions bg_per_exec,
       elapsed_time/executions/1e6 sec_per_exec, sql_text
FROM   v$sqlarea
WHERE  sql_text LIKE '%ORDERS_HIST%'
ORDER  BY buffer_gets DESC;
```

Finds SQL `87ac...`:

```sql
SELECT * FROM orders_hist WHERE customer_id = :b1 AND status = 'ACTIVE';
```

Plan `abc123` (good week) — INDEX RANGE SCAN.
Plan `def456` (bad week) — TABLE ACCESS FULL.

### Root Cause

Plan flip. Statistics on `ORDERS_HIST` were regathered on Aug 3rd — the auto-stats job. New stats made the optimizer prefer full scan.

### Verify Root Cause

```sql
SELECT owner, table_name, last_analyzed, num_rows,
       LAG(num_rows) OVER (PARTITION BY owner, table_name ORDER BY last_analyzed) prev_rows
FROM   dba_tab_statistics
WHERE  table_name = 'ORDERS_HIST';
```

Or `DBA_HIST_SQLSTAT`:

```sql
SELECT snap_id, plan_hash_value, executions_delta,
       elapsed_time_delta/executions_delta/1e6 sec_per_exec
FROM   dba_hist_sqlstat
WHERE  sql_id = '87ac...'
ORDER  BY snap_id DESC
FETCH  FIRST 30 ROWS ONLY;
```

Shows plan_hash_value flip on Aug 3rd's afternoon.

## Fix

Load the good plan as a SQL Plan Baseline from AWR:

```sql
DECLARE
  n NUMBER;
BEGIN
  n := DBMS_SPM.LOAD_PLANS_FROM_AWR(
         begin_snap => &pre_flip_snap,
         end_snap   => &pre_flip_snap,
         basic_filter => q'[sql_id = '87ac...']');
  DBMS_OUTPUT.PUT_LINE('Loaded '||n||' plans');
END;
/
```

Next execution picks up the baseline; SQL is fast again.

## Lessons Learned

- **AWR compare** is the fastest way to spot regressions week-over-week.
- Watch **buffer hit ratio** in the Instance Efficiency section — small drops often mean plan flips.
- Top segments by physical reads narrow the culprit fast.
- **SPB** for critical SQL prevents this recurring.
- Consider `pending statistics` (`DBMS_STATS.SET_PREFS('PUBLISH','FALSE')`) on critical tables so new stats can be validated before publish.

## Related

- [AWR](../12-performance-tuning/awr.md).
- [SQL Plan Management](../11-sql-optimizer/sql-plan-management.md).
- [Statistics](../11-sql-optimizer/statistics.md).
