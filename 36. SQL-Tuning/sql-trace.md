# SQL Trace

## Overview

**SQL Trace** captures per-session runtime detail: SQL executed, wait events, bind values, execution stats. Multiple flavors, from simple to deep:

- `SQL_TRACE = TRUE` — basic (deprecated).
- `10046` event — modern; levels 1/4/8/12.
- `DBMS_MONITOR` — trace by session / client_id / service / module.

Written to trace files under `$ORACLE_BASE/diag/rdbms/<db>/<sid>/trace/`.

## Levels

`10046` levels combine bit flags:

| Level | Includes                   |
| ----- | -------------------------- |
| 1     | Basic (SQL + basic stats)  |
| 4     | + Bind values              |
| 8     | + Wait events              |
| 12    | Bind + waits (most useful) |

Level 12 is the common answer for "trace a session".

## Enabling — Own Session

```sql
ALTER SESSION SET tracefile_identifier = 'MY_TRACE';

ALTER SESSION SET EVENTS '10046 trace name context forever, level 12';

-- run SQL
SELECT ...;

ALTER SESSION SET EVENTS '10046 trace name context off';
```

Find your trace file:

```sql
SELECT value FROM v$diag_info WHERE name = 'Default Trace File';
```

## Enabling — Another Session (`DBMS_MONITOR`)

Preferred over `SYS.DBMS_SYSTEM.SET_EV`:

```sql
BEGIN
  DBMS_MONITOR.SESSION_TRACE_ENABLE(
    session_id => &sid,
    serial_num => &serial,
    waits      => TRUE,
    binds      => TRUE);
END;
/

-- Later
BEGIN
  DBMS_MONITOR.SESSION_TRACE_DISABLE(session_id => &sid, serial_num => &serial);
END;
/
```

Trace file naming: `<SID>_ora_<pid>_<identifier>.trc`.

## Enabling by Client ID / Module

Trace only sessions matching an application marker:

```sql
-- Client ID (set by app via DBMS_APPLICATION_INFO)
EXEC DBMS_MONITOR.CLIENT_ID_TRACE_ENABLE(
       client_id => 'batch_load_2026Q3',
       waits => TRUE, binds => TRUE);

-- Later
EXEC DBMS_MONITOR.CLIENT_ID_TRACE_DISABLE(client_id => 'batch_load_2026Q3');

-- Service + module + action
EXEC DBMS_MONITOR.SERV_MOD_ACT_TRACE_ENABLE(
       service_name => 'APP_SVC',
       module_name  => 'OrderService',
       action_name  => 'loadOrders',
       waits => TRUE, binds => TRUE);
```

Best for intermittent problems.

## Reading Trace Files

Never open a raw 10046 trace file — use TKPROF or a visual tool.

See [TKPROF](tkprof.md).

Quick preview:

```bash
head -100 $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/PRD1_ora_12345_MY_TRACE.trc
```

Key lines:

- `PARSING IN CURSOR` — starts a SQL block.
- `PARSE` — parse phase stats.
- `EXEC` — execute phase stats.
- `FETCH` — fetch phase stats.
- `WAIT` — wait event with duration.
- `BINDS` — bind variable values.
- `STAT` — plan step stats.

## 10053 — CBO Trace

Different from 10046. Shows the **optimizer's cost calculations** for a hard parse:

```sql
ALTER SESSION SET tracefile_identifier = 'CBO';
ALTER SESSION SET EVENTS '10053 trace name context forever, level 1';

EXPLAIN PLAN FOR SELECT ...;

ALTER SESSION SET EVENTS '10053 trace name context off';
```

Massive file. Read for:

- Row-source cardinality estimates.
- Access path costs.
- Join order enumeration.
- Transformations considered.

## Tracing Currently-Running SQL

If you know the SPID (OS pid):

```bash
# oradebug — SYSDBA only
sqlplus / as sysdba
oradebug setospid 12345
oradebug event 10046 trace name context forever, level 12
-- wait
oradebug event 10046 trace name context off
oradebug tracefile_name
```

## Trace File Size Limits

`MAX_DUMP_FILE_SIZE` caps in blocks (default `UNLIMITED`). Set to bounds for busy traces:

```sql
ALTER SYSTEM SET max_dump_file_size = '100M' SCOPE=BOTH;
```

## Turning Trace On for New Sessions

Rare, dangerous, but useful for reproducing at logon:

```sql
-- Triggering event on logon
CREATE OR REPLACE TRIGGER trace_batch_user
AFTER LOGON ON DATABASE
BEGIN
  IF USER = 'BATCH_USER' THEN
    EXECUTE IMMEDIATE 'ALTER SESSION SET EVENTS ''10046 trace name context forever, level 12''';
  END IF;
END;
/
```

Drop the trigger after collecting.

## Best Practices

1. Prefer `DBMS_MONITOR` over legacy alter session for other sessions.
2. Always set `tracefile_identifier` — makes files findable.
3. Cap `max_dump_file_size` before enabling in prod.
4. Trace short windows; big traces are hard to parse.
5. Use 10046 level 12 for waits + binds; simpler when possible.
6. Use 10053 for CBO investigation; separate from 10046.
7. Compress trace files after collection (they compress ~10x).
8. Delete trace files after analysis (ADR retention helps).

## Related

- [TKPROF](tkprof.md).
- [Trace Files](../24-monitoring/trace-files.md).
- [Bind Peeking](bind-peeking.md).
