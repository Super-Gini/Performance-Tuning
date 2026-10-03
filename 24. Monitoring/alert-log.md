# Alert Log

## Overview

The **alert log** is the database's authoritative event stream — every startup, shutdown, log switch, database-level ORA- error, non-default parameter, and administrative action lands there. It's the very first file a DBA opens when investigating any issue. In 11g+ it lives inside the **ADR** structure and comes in two forms: an `alert_<SID>.log` text file (human readable) and a `log.xml` XML variant (machine readable, richer).

## Location

```bash
$ORACLE_BASE/diag/rdbms/<db_name>/<instance_name>/trace/alert_<SID>.log
$ORACLE_BASE/diag/rdbms/<db_name>/<instance_name>/alert/log.xml
```

Find fast:

```bash
adrci exec="show homes"
adrci exec="show alert -tail 50"
```

Or from SQL:

```sql
SELECT value AS trace_dir
FROM   v$diag_info
WHERE  name = 'Diag Trace';
```

## What Gets Logged

Category → typical entries:

| Category               | Entries                                                                                  |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| Startup / shutdown     | `Starting ORACLE instance`, `Completed: alter database mount`, `Shutting down instance`. |
| Non-default parameters | Every non-default `init.ora` value at startup.                                           |
| Log switches           | `Thread 1 advanced to log sequence 12345`.                                               |
| Archive                | `ARC0: Beginning to archive thread 1 sequence 12345`.                                    |
| Media recovery         | `Media Recovery Log +FRA/...`, `Completed redo application`.                             |
| Errors                 | `ORA-00600 [kcbz_check_objd_typ]`, `ORA-01578 corruption`.                               |
| DDL / ADMIN            | Tablespace add/drop, ALTER DATABASE, DBID changes.                                       |
| Data Guard             | `Redo Transport Network` messages, `Media Recovery Waiting`.                             |
| Advisor / auto-task    | AWR snapshot creation.                                                                   |
| Deadlock               | `ORA-00060: Deadlock detected. See Note 60.1`.                                           |
| Session-level PROBLEMS | `KILL Session` requests, `ORA-04030 out of process memory`.                              |

## Reading Fast — `adrci`

```bash
adrci
adrci> set homepath diag/rdbms/prd/PRD1
adrci> show alert -term -tail 100
```

Filter to a search:

```bash
adrci> show alert -P "MESSAGE_TEXT LIKE '%ORA-00600%'"
```

Or with grep on the text file:

```bash
tail -f $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log | grep -E 'ORA-|Ora'
```

## Log Rotation

Alert log does **not** rotate automatically. It grows forever unless you manage it.

Recommended cron:

```bash
# Weekly Sunday 04:00
0 4 * * 0 mv $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log \
             $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log.$(date +\%Y\%m\%d) && \
             touch $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log && \
             gzip $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log.$(date +\%Y\%m\%d)
```

Oracle keeps writing to the same open file handle, so `mv + touch` is fine — the DB continues writing to the moved file until next log entry, at which point it opens the new one.

## Reading the XML Variant

The XML version has fields your monitoring can parse:

```xml
<msg time='2026-08-06T02:35:14.001+00:00'
     org_id='oracle' comp_id='rdbms'
     msg_id='opistr_real:2500:2005567130'
     type='INCIDENT_ERROR' group='Generic Internal Error'
     level='1' host_id='prd-db01' host_addr='10.0.1.4'
     module='sqlplus@prd-db01 (TNS V1-V3)' pid='45782'>
 <txt>Errors in file /u01/app/oracle/diag/.../trace/PRD1_ora_45782.trc  (incident=234567):
ORA-00600: internal error code, arguments: [ktbdchk1: bad dscn], [], [], [], [], [], [], [], [], [], [], []
Incident details in: /u01/app/oracle/diag/.../incident/incdir_234567/PRD1_ora_45782_i234567.trc
Use ADRCI or Support Workbench to package the incident.
See Note 411.1 at My Oracle Support for error and packaging details.
</txt>
</msg>
```

`type=INCIDENT_ERROR` + `msg_id=opistr_real` = trap this for paging.

## Views (Server-Side Access)

```sql
-- Recent alert entries
SELECT   originating_timestamp, message_text
FROM     v$diag_alert_ext
WHERE    originating_timestamp > SYSDATE - 1
ORDER BY originating_timestamp DESC;

-- Search
SELECT   originating_timestamp, host_id, module_id, message_text
FROM     v$diag_alert_ext
WHERE    message_text LIKE '%ORA-01578%'
   AND   originating_timestamp > SYSDATE - 30
ORDER BY originating_timestamp DESC;

-- Deadlocks in the last week
SELECT count(*) FROM v$diag_alert_ext
WHERE message_text LIKE '%ORA-00060%'
  AND originating_timestamp > SYSDATE - 7;
```

## Critical Patterns to Alert On

| Pattern                                     | Meaning / Action                               |
| ------------------------------------------- | ---------------------------------------------- |
| `ORA-00600 [kdscat_leaf_2]`                 | Index-cache corruption. Open SR immediately.   |
| `ORA-07445` (segfault)                      | Process crash. Trace file with call stack.     |
| `ORA-01578: data block corrupted`           | Physical block corruption. Run `dbverify`.     |
| `ORA-01555 snapshot too old`                | Undo too small or query too long.              |
| `ORA-04031: unable to allocate...`          | Shared pool memory exhausted.                  |
| `ORA-00060: Deadlock detected`              | App bug — trace file has the graph.            |
| `ORA-00257: archiver error`                 | FRA/archive dest full. Immediate action.       |
| `ORA-00020: maximum number of processes`    | Sessions leaked. Raise `processes` or fix app. |
| `Cannot allocate log`                       | LGWR waiting on ARC. Add / grow redo logs.     |
| `ORA-16038 log ... cannot be archived`      | Archiver blocked.                              |
| `Thread 1 cannot allocate new log`          | Redo starvation.                               |
| `Started redo application at`               | Standby / recovery event. Normal on standby.   |
| `IPC Send timeout`                          | RAC interconnect issue.                        |
| `NOTE: process ... failed to lock instance` | RAC eviction incoming.                         |

## Purging Old Diagnostic Data

ADR purge policy — set retention:

```bash
adrci
adrci> set homepath diag/rdbms/prd/PRD1
adrci> show control                         # current retention
adrci> set control (SHORTP_POLICY = 720)   # hours - trace files
adrci> set control (LONGP_POLICY  = 8760)  # hours - incident files
adrci> purge -age 4320                      # purge older than 6 months
```

Default: 720 h (30 days) short, 8760 h (365 days) long. Adjust to org policy.

## Common Issues

- **Alert log grows to 10s of GB** — No rotation policy. Add cron rotator.
- **Alert log stops updating** — Filesystem full, or the file was `rm`ed while open. `lsof | grep alert` to check.
- **`ORA-00600` recurring** — Take a full incident package and open SR.
- **Alert log full of TT00 timezone messages** — DB has TZ files loaded but no TSTZ columns; noise, ignore.
- **Log switch messages every few seconds** — Redo logs too small. See [Redo Tuning](../06-redo/redo-tuning.md).
- **Alert log missing from expected location** — Instance was started with the wrong `ORACLE_BASE`. Confirm with `v$diag_info`.

## Best Practices

1. **Monitor the alert log 24/7** — every serious DBA setup ships alert log to a SIEM.
2. Rotate weekly; keep 90 days of rotated logs compressed.
3. Alert on any `ORA-00600`, `ORA-07445`, `ORA-01578` immediately.
4. Set ADR retention to match your compliance window.
5. Prefer the XML alert log for tooling (`log.xml`) — richer metadata.
6. Use `v$diag_alert_ext` for SQL-driven queries; grep for one-off checks.
7. Never delete the current `alert.log` while the DB is open — `mv + touch`.
8. Correlate alert log entries with AWR snapshot IDs for context.
9. On RAC, monitor each instance's alert log independently.
10. On Data Guard, monitor both primary and standby alert logs — apply errors show only on standby.

## Interview Questions

1. **Q:** Where is the 19c alert log?
   **A:** `$ORACLE_BASE/diag/rdbms/<db>/<inst>/trace/alert_<SID>.log` and its XML twin in `alert/log.xml`.

2. **Q:** How would you find all ORA-01578 in the last month via SQL?
   **A:** `SELECT ... FROM v$diag_alert_ext WHERE message_text LIKE '%ORA-01578%' AND originating_timestamp > SYSDATE-30`.

3. **Q:** How is the XML alert log different?
   **A:** Structured (`type`, `msg_id`, `module`, timestamps in ISO 8601) — easier to parse for monitoring pipelines.

4. **Q:** Alert log has grown to 20 GB. Safe to delete while DB is running?
   **A:** No — Oracle holds it open. `mv` it, `touch` a new one, then compress the old one.

5. **Q:** Which command purges old ADR contents?
   **A:** `adrci -> purge -age <hours>` in the relevant home.

## References

- Oracle Database Administrator's Guide 19c — Managing Diagnostic Data
- MOS Doc ID 443529.1 — Alert Log Management
- MOS Doc ID 422893.1 — 11g Diagnosability Framework (ADR)
