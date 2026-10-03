# Trace Files

## Overview

**Trace files** are per-process text (or binary) files Oracle writes when instructed — either explicitly by `ALTER SESSION SET EVENTS`, implicitly by `SQL_TRACE`, or automatically when a process crashes / hits an error like ORA-00600 or ORA-07445. They live under the ADR and are how DBAs turn a symptom into a root cause.

Four broad flavors:

- **User trace** — `SQL_TRACE`, `10046`, `10053`. Tune SQL.
- **Background trace** — one per background process (`PMON`, `SMON`, ...). Usually quiet.
- **Incident trace** — auto-written when Oracle logs `INCIDENT_ERROR`. ORA-00600, ORA-07445.
- **Alert log** — the special one, [Alert Log](alert-log.md).

## Location

```
$ORACLE_BASE/diag/rdbms/<db>/<inst>/trace/       -- user + background trace
$ORACLE_BASE/diag/rdbms/<db>/<inst>/incident/    -- incident traces per incident dir
$ORACLE_BASE/diag/rdbms/<db>/<inst>/cdump/       -- core dumps
```

Show current directories:

```sql
SELECT name, value FROM v$diag_info;
```

Get **your own** session's trace filename:

```sql
SELECT tracefile FROM v$process WHERE addr =
       (SELECT paddr FROM v$session WHERE sid = SYS_CONTEXT('USERENV','SID'));
```

Or with:

```sql
COLUMN name FORMAT A20
COLUMN value FORMAT A100
SELECT name, value FROM v$diag_info WHERE name = 'Default Trace File';
```

## Enabling User Trace

### `SQL_TRACE` (Simple)

Own session:

```sql
ALTER SESSION SET SQL_TRACE = TRUE;
-- ... run the SQL
ALTER SESSION SET SQL_TRACE = FALSE;
```

Another session:

```sql
EXEC DBMS_MONITOR.SESSION_TRACE_ENABLE(session_id=>&sid, serial_num=>&serial,
     waits=>TRUE, binds=>TRUE);

-- Later
EXEC DBMS_MONITOR.SESSION_TRACE_DISABLE(session_id=>&sid, serial_num=>&serial);
```

### Extended SQL Trace (`10046`)

More detail — wait events with bind values:

```sql
ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- Levels
--   1  Basic
--   4  Bind values
--   8  Wait events
--  12  Bind values + wait events (most useful)

-- Turn off
ALTER SESSION SET EVENTS '10046 trace name context off';
```

### CBO Trace (`10053`)

Shows the optimizer's cost calculations for a hard parse:

```sql
ALTER SESSION SET EVENTS '10053 trace name context forever, level 1';
EXPLAIN PLAN FOR SELECT ...;   -- or actual run for hard parse
ALTER SESSION SET EVENTS '10053 trace name context off';
```

Massive files — use for one query at a time.

### Client Identifier / Module / Action

Trace on the fly by application marker:

```sql
-- Application sets these
DBMS_APPLICATION_INFO.SET_CLIENT_INFO('order_svc_batch');
DBMS_APPLICATION_INFO.SET_MODULE('OrderService','loadOrders');

-- DBA enables trace
EXEC DBMS_MONITOR.CLIENT_ID_TRACE_ENABLE('order_svc_batch', waits=>TRUE, binds=>TRUE);
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(service_name=>'APP_SVC',
     module_name=>'OrderService', action_name=>'loadOrders',
     waits=>TRUE, binds=>TRUE);
```

Fires only when a session matches — great for intermittent problems.

## TKPROF: Turning Trace into Human-Readable

```bash
tkprof PRD1_ora_45782.trc /tmp/trace_report.txt \
       waits=yes sort=exeela,fchela sys=no
```

Options:

- `waits=yes` — include wait events.
- `sort=` — reorder queries by elapsed, CPU, etc.
- `sys=no` — hide recursive SYS SQL.
- `explain=user/pw` — run EXPLAIN PLAN on each captured SQL.

Output ranks queries by whatever `sort` you gave, with per-query row counts, CPU, elapsed, disk reads, waits.

## Incident Trace

When Oracle logs `INCIDENT_ERROR` (usually ORA-00600 / 07445), it:

1. Creates an **incident directory**: `.../incident/incdir_<incident_id>/`.
2. Writes a trace file: `<sid>_<pname>_<pid>_i<incident>.trc`.
3. Records the incident in `V$DIAG_INCIDENT`.

Query recent incidents:

```sql
SELECT   incident_id, create_time, problem_key, error_facility, error_number
FROM     v$diag_incident
WHERE    create_time > SYSDATE - 7
ORDER BY create_time DESC;
```

Package for MOS:

```bash
adrci exec="ips create package incident 234567"
adrci exec="ips add file /path/to/extra.log package 4"
adrci exec="ips generate package 4 in /tmp"
```

Produces a ZIP ready to upload to Oracle Support.

## Background Trace

Files named `<sid>_<bg>_<pid>.trc`, e.g. `PRD1_pmon_1234.trc`. Rarely of interest unless the background process crashed. `PMON`, `SMON`, `LGWR`, `DBWn` traces during a normal run are noise.

Watch for:

- `LGWR` trace with `log file sync` self-diagnosis (12c+).
- `DBWn` trace when checkpoint incomplete.
- `SMON` trace during undo cleanup.

## Trace File Naming

```
<SID>_<process>_<pid>.trc
PRD1_ora_45782.trc          # dedicated server (user session)
PRD1_lgwr_1234.trc          # LGWR
PRD1_j000_5678.trc          # Scheduler slave
PRD1_ora_45782_i234567.trc  # incident trace for incident 234567
```

## Managing Trace File Size

```sql
SHOW PARAMETER max_dump_file_size

-- Default: UNLIMITED. Cap for safety
ALTER SYSTEM SET max_dump_file_size = '100M' SCOPE=BOTH;
```

For long-running trace sessions:

```sql
ALTER SESSION SET tracefile_identifier = 'BATCH_LOAD_2026Q3';
-- Now the file is PRD1_ora_45782_BATCH_LOAD_2026Q3.trc
```

## Diagnostic Queries

```sql
-- Trace file for a specific session
SELECT s.sid, s.serial#, p.tracefile
FROM   v$session s JOIN v$process p ON p.addr = s.paddr
WHERE  s.sid = &target_sid;

-- Any active trace enabled for me
SELECT * FROM v$sess_trace;   -- 19c+

-- Incidents ready to package
SELECT problem_id, first_incident, key1_value, key2_value
FROM   v$diag_problem
ORDER  BY first_incident_time DESC;

-- Enabled DBMS_MONITOR traces
SELECT * FROM dba_enabled_traces;
```

## Common Issues

- **Trace file empty** — `SQL_TRACE=TRUE` but no queries hit yet, or session ended before trace closed. Flush with `ALTER SESSION SET SQL_TRACE=FALSE`.
- **`WARN: files truncated at max_dump_file_size`** — Raise `max_dump_file_size` or cap SQL scope.
- **Cannot find trace file** — Wrong `$ORACLE_BASE`; use `v$diag_info` to confirm.
- **Trace disks fill up** — ADR retention too high or too many active traces. Configure `SHORTP_POLICY`.
- **TKPROF error `unmapped tables`** — Old TKPROF vs new trace; use TKPROF from same $ORACLE_HOME.
- **Trace file is binary garbage** — 10046 trace under aggressive `_TRACE_FILES_PUBLIC=FALSE` + wrong shell reader. Set `+w` and reopen.

## Best Practices

1. Set `_trace_files_public = TRUE` in non-prod so DBAs can read each other's traces (never in prod).
2. Use `tracefile_identifier` on every trace session — makes files easy to find.
3. Use `DBMS_MONITOR` for other sessions, not `oradebug` (safer).
4. Cap `max_dump_file_size` to prevent runaway.
5. Delete trace files after analysis; ADR purge handles it, but bulk cleanup weekly is safer.
6. For repeatable issues, capture a **10046 level 12** AND a **10053** — one shows execution, one shows optimizer choice.
7. Compress incident directories before opening SR — Oracle Support accepts ZIPs faster than raw.
8. Never `oradebug hanganalyze` in prod without a plan — it can pin a spinlock.
9. Use SQL Plan Baseline capture for the _plan_ fix, TKPROF for the _symptom_ diagnosis.
10. For long jobs, alternate trace on/off in windows — files stay small, still informative.

## Interview Questions

1. **Q:** How do you trace another session's SQL execution with binds?
   **A:** `DBMS_MONITOR.SESSION_TRACE_ENABLE(sid, serial, waits=>TRUE, binds=>TRUE)`.

2. **Q:** What is a 10053 trace?
   **A:** Cost-Based Optimizer trace showing the optimizer's row counts, costs, and transformation decisions for a hard parse.

3. **Q:** TKPROF says the trace is "unmapped" — what's wrong?
   **A:** TKPROF binary is from a different Oracle version than the trace file. Use the same $ORACLE_HOME's TKPROF.

4. **Q:** Where does an ORA-00600 trace file end up?
   **A:** `$ORACLE_BASE/diag/rdbms/<db>/<inst>/incident/incdir_<id>/<sid>_ora_<pid>_i<id>.trc`.

5. **Q:** How do you know a session is being traced right now?
   **A:** `SELECT * FROM V$SESS_TRACE` (19c) or `V$SESSION.SQL_TRACE = 'ENABLED'`.

## References

- Oracle Database Performance Tuning Guide — Tracing
- MOS Doc ID 21154.1 — SQL_TRACE / 10046 basics
- MOS Doc ID 32615.1 — CBO 10053 trace
- MOS Doc ID 443529.1 — Alert and Trace management
- [TKPROF](../36-sql-tuning/tkprof.md) — deeper coverage
