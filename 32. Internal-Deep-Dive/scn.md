# SCN — System Change Number

## Overview

The System Change Number (SCN) is Oracle's monotonically-increasing logical clock. Every commit, every block modification, every redo record, every consistent read snapshot, and every RAC message carries an SCN. It's the fabric that makes MVCC, recovery, Data Guard, Flashback, and distributed transactions work.

This page covers SCN structure at the byte level, generation mechanisms, propagation between DBs, and the SCN-related failure modes DBAs encounter.

## SCN Format

An SCN is **48 bits** wide (as of 12c+; historically discussed as 64-bit "SCN Base + SCN Wrap"):

- **SCN Base** — low-order 32 bits.
- **SCN Wrap** — high-order 16 bits (the 32-bit wrap column exists but only 16 bits are actually used before rollover).

Display formats:

- **Compact**: single decimal number up to ~281 trillion.
- **Wrap.Base**: `WRAP.BASE` two-part display; older tooling.
- **Timestamp**: converted approximation via `SMON_SCN_TIME` map.

Get SCN as compact and as wrap.base:

```sql
SELECT DBMS_FLASHBACK.GET_SYSTEM_CHANGE_NUMBER  compact_scn,
       current_scn                              via_v$database
FROM   v$database;

-- Wrap.Base decoding
SELECT current_scn,
       TRUNC(current_scn / POWER(2,32)) wrap,
       MOD(current_scn, POWER(2,32))    base
FROM   v$database;
```

## Where SCN Is Stored

Every block:

- **Block header** carries the SCN of the last change to the block (`RDBA`, `CSCN` field).
- **ITL entries** carry the transaction's commit SCN when cleaned out.

Every redo record:

- **Redo record header** — SCN of the change.
- Sub-vectors within — SCN references for consistency.

Every buffer:

- `X$BH.SCN_BAS` + `X$BH.SCN_WRP` — block's current SCN.

Every session:

- Session-level "snapshot SCN" for read consistency.

## SCN Generation Mechanisms

Three internal functions:

- **`kcmgcs`** — Get Current SCN. Returns without advancing. Used by SELECTs for snapshot capture.
- **`kcmgas`** — Get Advanced SCN. Advances SCN by 1 and returns. Used by every commit.
- **`kcmgss`** — Set Session SCN (from external source, e.g., distributed transaction).

Commit sequence:

1. Session prepares commit.
2. `kcmgas` advances SCN.
3. LGWR writes redo including that commit SCN.
4. Commit SCN stamped into block ITL entries (delayed for most).
5. Signal client.

Read sequence:

1. `kcmgcs` captures current SCN → session's snapshot SCN.
2. Every block read: compare block SCN with snapshot SCN. If block newer, build a CR clone using undo.

## SCN Rate — The Concern

SCN is limited: 48 bits ≈ 2.8 × 10^14.

Oracle enforces a **soft SCN rate limit**: nominally 16,384 SCN increments per second (16k/s). If sustained, the SCN could exhaust ~2035 given the epoch of 1988. Modern Oracle (12c+) raises the soft limit dynamically based on activity — `_max_reasonable_scn_rate`.

Practical: monitor SCN growth.

```sql
SELECT metric_name, ROUND(value, 2) value_per_sec, end_time
FROM   v$sysmetric
WHERE  metric_name = 'SCN growth per second'
   AND intsize_csec = 6000;
```

Typical: 100–2000 SCN/sec. If sustained > 8000, investigate what's driving commits (batch code, chatty apps, tight loops).

## SCN Synchronization Across DBs (DB Links)

When a session on DB-A queries via DB link to DB-B:

1. DB-A sends its current SCN in the request.
2. DB-B's SCN is compared. If lower, DB-B **advances** its SCN to DB-A's.
3. Response returned.
4. DB-A never lowers.

Consequence: SCN "chases" the highest DB in your DB-link mesh. A single high-SCN DB (from a bad clone, time-travel, wrong parameter setting) can drag the whole fleet.

**Bug 12371955** (fixed 12.1) — an artificially high SCN could get planted via distributed TX. Post-fix: `_external_scn_rejection_threshold_hours` (default 24) — an incoming SCN more than N hours ahead of local is rejected.

Query:

```sql
SHOW PARAMETER _external_scn_rejection_threshold_hours

-- Recent rejected SCNs (11g+)
SELECT * FROM x$kcmscn WHERE scnbas > 0 FETCH FIRST 20 ROWS ONLY;
```

## SMON_SCN_TIME — SCN ↔ Timestamp Map

SMON populates a rolling window of (SCN, timestamp) pairs every 5 minutes into `SMON_SCN_TIME`. Retention: 5 days (older rows purged).

```sql
SELECT   scn, TO_CHAR(time_dp, 'YYYY-MM-DD HH24:MI:SS') scn_time
FROM     smon_scn_time
ORDER BY scn DESC
FETCH FIRST 10 ROWS ONLY;
```

Conversion functions use this:

```sql
SELECT SCN_TO_TIMESTAMP(12345678901) FROM dual;
SELECT TIMESTAMP_TO_SCN(SYSTIMESTAMP - INTERVAL '1' HOUR) FROM dual;
```

Because entries are 5-min-bucketed, conversion is approximate — precision ± 5 min for old SCNs.

Common failure — `ORA-08181: specified number is not a valid system change number` — the SCN is outside `SMON_SCN_TIME`'s window (older than 5 days).

## SCN in Redo — Change Vector SCN

Every redo change vector (CV) has:

- **SCN** — SCN this change was assigned.
- **DBA** — the block affected.
- **OP-code** — what kind of change.
- **Change data** — the actual bytes.

At log switch, LGWR flushes redo up through some SCN. `V$LOG.NEXT_CHANGE#` records the max SCN in that log.

```sql
SELECT group#, sequence#, first_change#, next_change#, status
FROM   v$log
ORDER  BY group#;
```

## SCN in Data Guard

Primary's `V$DATABASE.CURRENT_SCN` should be equal to standby's `V$DATABASE.RECOVER_SCN` under real-time apply (within a few thousand).

```sql
-- Primary
SELECT current_scn FROM v$database;

-- Standby
SELECT recover_scn FROM v$database;
```

Difference = apply lag in SCN terms.

## SCN in RAC

All instances share one SCN counter, coordinated via GES. The `kcmgas` call on instance A obtains the next SCN by requesting from the SCN manager (in RAC, distributed via LMS).

`_gcs_scn_broadcast_interval_ms` controls how often instances broadcast SCN. Under-tuning causes visible cross-instance ordering issues; default (100ms) is fine.

## SCN Headroom Monitoring

Check "SCN age" — how far into the theoretical 2035 exhaustion your DB is:

```sql
SELECT   name, dbid,
         current_scn,
         ROUND(current_scn / POWER(2,48) * 100, 8) pct_of_max
FROM     v$database;
```

Typical values: `pct_of_max` = 0.00000X.

MOS Doc ID 1376995.1 covers SCN headroom monitoring for old DBs at risk.

## Practical Use — Flashback

Every flashback feature accepts SCN:

```sql
-- Query
SELECT * FROM orders AS OF SCN 12345678 WHERE id = 100;

-- Table
FLASHBACK TABLE app.orders TO SCN 12345678;

-- Database
FLASHBACK DATABASE TO SCN 12345678;
```

Or as timestamp (converted via `SMON_SCN_TIME`):

```sql
SELECT * FROM orders AS OF TIMESTAMP SYSDATE - INTERVAL '1' HOUR;
```

## Practical Use — Recovery

```
RMAN> RUN {
  SET UNTIL SCN 12345678;
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}
```

Or PITR to time:

```
RMAN> SET UNTIL TIME "TO_DATE('2026-08-06 12:00','YYYY-MM-DD HH24:MI')";
```

RMAN converts time to SCN using primary's SMON_SCN_TIME during the RESTORE.

## RESETLOGS SCN

`RESETLOGS` doesn't reset SCN — it creates a **new incarnation** with `RESETLOGS_CHANGE#` and `RESETLOGS_TIME`:

```sql
SELECT resetlogs_change#, resetlogs_time, resetlogs_id
FROM   v$database;

SELECT * FROM v$database_incarnation ORDER BY incarnation#;
```

Backups belong to a specific incarnation. RMAN needs `RESET DATABASE TO INCARNATION N` before restoring from a previous incarnation's backup.

## Common Failure Modes

- **`ORA-08181: specified number is not a valid system change number`** — SCN older than SMON_SCN_TIME window.
- **`ORA-19706: invalid SCN`** — RMAN restore beyond available redo.
- **`ORA-01555: snapshot too old`** — read consistency: undo for SCN gone. Not really SCN — undo.
- **`ORA-30052: invalid lower limit snapshot expression`** — flashback query with wrong SCN.
- **SCN drift after cross-DB link** — a bad remote DB pulled yours forward.
- **`kcmgas: SCN increment throttled`** — soft rate limit hit; investigate commit rate.

## Views for SCN Investigation

| View                                           | Purpose                              |
| ---------------------------------------------- | ------------------------------------ |
| `V$DATABASE.CURRENT_SCN`                       | Current DB SCN.                      |
| `V$DATAFILE.CHECKPOINT_CHANGE#`                | Per-datafile SCN of last checkpoint. |
| `V$LOG.FIRST_CHANGE#/NEXT_CHANGE#`             | SCN range per online redo log.       |
| `V$ARCHIVED_LOG.FIRST_CHANGE#`                 | SCN range per archived log.          |
| `V$SYSMETRIC / metric='SCN growth per second'` | Rate.                                |
| `SMON_SCN_TIME`                                | SCN↔time map.                        |
| `V$RESTORE_POINT`                              | Named restore points with SCNs.      |
| `V$DATABASE_INCARNATION`                       | SCN boundaries between incarnations. |
| `X$KCMSCN`                                     | Rejected external SCNs.              |

## Interview Framing

> "What is the SCN?"

Oracle's global, monotonically-increasing logical clock. Every commit advances it; every read captures it. Used for MVCC, recovery, DG, Flashback, distributed TX ordering.

> "How does DBLINK affect SCN?"

Both DBs sync — the higher-SCN DB pulls the lower up. This means one DB with an artificially high SCN can propagate across a mesh via DB links.

> "How would you troubleshoot ORA-08181?"

SCN older than `SMON_SCN_TIME`'s 5-day window. Either the query is using a very old SCN, or you're doing flashback to a time too far back.

## Related

- [Consistent Read](../05-undo/consistent-read.md).
- [Undo & CR Internals](undo-cr-internals.md).
- [Redo Internals](redo-internals.md).
- [Data Guard Architecture](../17-data-guard/architecture.md).
- [Recovery](../15-rman/recovery.md).
- [Flashback](../16-flashback/index.md).
- [Block Format & ITL](block-format-itl.md).
- [MOS Doc ID 1376995.1 — SCN Headroom](https://support.oracle.com).
