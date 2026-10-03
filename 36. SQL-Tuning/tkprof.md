# TKPROF

## Overview

**TKPROF** (Transient Kernel PROFiler) converts raw 10046 trace files into human-readable performance reports. Ships with every Oracle Home: `$ORACLE_HOME/bin/tkprof`.

## Basic Usage

```bash
tkprof <input.trc> <output.txt> \
       sort=exeela,fchela \
       waits=yes \
       sys=no
```

Options:

| Option            | Meaning                                                        |
| ----------------- | -------------------------------------------------------------- |
| `sort=`           | Order SQL by these metrics (comma-separated).                  |
| `waits=yes/no`    | Include wait event breakdown per SQL.                          |
| `sys=yes/no`      | Include recursive SYS SQL (usually no).                        |
| `explain=user/pw` | Run EXPLAIN PLAN on each SQL and include.                      |
| `table=user.tab`  | Store data in an explain plan table (rarely used).             |
| `record=`         | Write SQL statements to a file for replay.                     |
| `insert=`         | Write INSERTs into `V$PARAMETER` table (for AWR-like storage). |

Sort keys:

- `prsela` — parse elapsed.
- `exeela` — execute elapsed.
- `fchela` — fetch elapsed.
- `prscpu`, `execpu`, `fchcpu` — CPU variants.
- `prsdsk`, `exedsk`, `fchdsk` — physical read.
- `prsqry`, `exeqry`, `fchqry` — consistent reads.

Most useful default: `sort=fchela,exeela` — surfaces the slow-to-fetch and slow-to-execute statements.

## Sample Report Section

```
SELECT * FROM app.orders WHERE customer_id = :b1

call     count       cpu    elapsed       disk      query    current      rows
------- ------  -------- ---------- ---------- ---------- ----------  ---------
Parse        1      0.00       0.00          0          0          0          0
Execute      1      0.00       0.00          0          0          0          0
Fetch     1500      0.42       1.85       1234      45678          0     15000
------- ------  -------- ---------- ---------- ---------- ----------  ---------
total     1502      0.42       1.85       1234      45678          0     15000

Misses in library cache during parse: 0
Optimizer mode: ALL_ROWS
Parsing schema id: 45

Rows     Row Source Operation
-------  ---------------------------------------------------
15000    TABLE ACCESS BY INDEX ROWID BATCHED ORDERS (cr=45678 pr=1234 pw=0 time=1850000 us cost=520)
15000     INDEX RANGE SCAN ORDERS_CUSTOMER_IDX (cr=100 pr=10 pw=0 time=1200 us cost=15)


Elapsed times include waiting on following events:
  Event waited on                             Times   Max. Wait  Total Waited
  ----------------------------------------   Waited  ----------  ------------
  db file sequential read                     1234        0.15         1.65
  SQL*Net message from client                 1500        0.02         0.10
  SQL*Net message to client                   1500        0.00         0.01
```

Reading:

- **call/count** — parse (1), execute (1), fetch (1500 fetches — 1500 rows batched).
- **cpu vs elapsed** — 0.42 CPU, 1.85 elapsed → 1.43 s waiting.
- **disk** — 1234 physical reads.
- **query** — 45678 consistent reads (from cache).
- **rows** — 15,000.
- Row Source Operation — actual plan with row counts.
- Event waited on — where the elapsed went. Storage IO here.

## Summary Section (End)

TKPROF also produces a summary:

```
OVERALL TOTALS FOR ALL NON-RECURSIVE STATEMENTS

call     count       cpu    elapsed       disk      query    current      rows
------- ------  -------- ---------- ---------- ---------- ----------  ---------
Parse       15      0.02       0.04          0          0          0          0
Execute     15      0.01       0.02          0          0          0          0
Fetch     6000      1.20       5.80       2500      95000          0     45000
------- ------  -------- ---------- ---------- ---------- ----------  ---------
total     6030      1.23       5.86       2500      95000          0     45000

Misses in library cache during parse: 3
Misses in library cache during execute: 0

    3  user  SQL statements in session.
    0  internal SQL statements in session.
    3  SQL statements in session.
```

## Recurring Patterns

- **High parse count** → application isn't reusing cursors. Fix: bind variables, `session_cached_cursors`.
- **CPU >> elapsed** → good; DB is doing work, not waiting.
- **Elapsed >> CPU** → waiting on something. Check waits section.
- **disk high, query low** → cold cache or full scan.
- **query high, disk low** → warm cache but many logical reads. Index?

## Common Recipes

### Just the Slow SQL

```bash
tkprof my_trace.trc slow.txt sort=fchela sys=no waits=yes
```

### Everything with EXPLAIN

```bash
tkprof my_trace.trc full.txt sort=exeela,fchela sys=no waits=yes explain=system/pw
```

### Find Parse-Heavy Statements

```bash
tkprof my_trace.trc parse_analysis.txt sort=prsela sys=no
```

## Filter Out Recursive SYS SQL

Almost always `sys=no`. Recursive SYS SQL clutters reports.

## TKPROF Alternatives

- **SQLT** (SQLT XPLAIN) — comprehensive report including trace parsing.
- **[SQLHC](sqlhc.md)** — Health Check on a specific SQL.
- **Perl / Python scripts** — some custom parsers (e.g., orasrp).
- **Method R Profiler** — commercial deep analysis.

## Common Issues

- **`unmapped tables`** — TKPROF from a different Oracle version than the trace. Use TKPROF from the same $ORACLE_HOME.
- **Wait events missing** — trace was 10046 level 1 or 4, not 8 or 12. Re-trace at level 12.
- **Bind values redacted** — some traces have `Bind privacy=1`. Only shown at level 12 in modern versions.

## Related

- [SQL Trace](sql-trace.md).
- [SQLT](sqlt.md).
- [Trace Files](../24-monitoring/trace-files.md).
