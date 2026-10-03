# Undo Parameters

## Overview

Init parameters that control undo (rollback) behavior. Modern setup: **automatic undo management** — Oracle picks the segments, sizes them, and manages retention.

## Parameters

| Parameter           | Default    | Purpose                                              |
| ------------------- | ---------- | ---------------------------------------------------- |
| `UNDO_MANAGEMENT`   | `AUTO`     | `AUTO` (recommended) or `MANUAL` (legacy).           |
| `UNDO_TABLESPACE`   | `UNDOTBS1` | Active undo tablespace.                              |
| `UNDO_RETENTION`    | `900` s    | Desired retention seconds. `MMON` auto-tunes upward. |
| `TEMP_UNDO_ENABLED` | `FALSE`    | 12c+ — Temp table undo goes to TEMP instead of UNDO. |

## Auto-Tuned Retention

Even if you set `UNDO_RETENTION=900`, Oracle keeps more undo when:

- Long queries are running (protects against `ORA-01555`).
- Undo tablespace has spare capacity.
- `RETENTION_GUARANTEE` is set on the tablespace.

Check what Oracle actually tuned to:

```sql
SELECT MAX(tuned_undoretention) tuned_seconds,
       MAX(maxquerylen)         longest_query
FROM   v$undostat
WHERE  begin_time > SYSDATE - 1;
```

## Sizing Undo Tablespace

Rule of thumb:

```
undo_size = (undo_generation_rate_per_second) * UNDO_RETENTION + safety_margin
```

Compute:

```sql
SELECT   ROUND(MAX(undoblks) *
              (SELECT block_size FROM dba_tablespaces
               WHERE tablespace_name = (SELECT value FROM v$parameter
                                        WHERE name='undo_tablespace'))/1024/1024, 2) mb_per_10min,
         MAX(tuned_undoretention) tuned_s
FROM     v$undostat
WHERE    begin_time > SYSDATE - 7;
```

Recommended undo tablespace = at least 2× (mb_per_10min × 6) — an hour of headroom.

## Guaranteed Retention

Prevents overwrite of unexpired undo even under pressure:

```sql
ALTER TABLESPACE UNDOTBS1 RETENTION GUARANTEE;
-- Undo Tablespace is now protected; may cause ORA-30036 (undo full) instead of ORA-01555
```

Trade-off: safer queries, riskier transactions. Use when long analytical queries must not fail.

## Temp Undo (12c+)

Redirects temporary-table undo to TEMP:

```sql
ALTER SESSION SET TEMP_UNDO_ENABLED = TRUE;
```

Reduces redo generation for global temp tables. Enable per-session for ETL that heavily uses GTTs.

## Switching Undo Tablespaces (Online)

```sql
CREATE UNDO TABLESPACE UNDOTBS2 DATAFILE '/u01/oradata/undotbs2.dbf' SIZE 20G AUTOEXTEND ON;
ALTER SYSTEM SET undo_tablespace = 'UNDOTBS2' SCOPE=BOTH;

-- Wait until old TS has no active TX
SELECT segment_name, status FROM dba_rollback_segs WHERE tablespace_name = 'UNDOTBS1';

DROP TABLESPACE UNDOTBS1 INCLUDING CONTENTS AND DATAFILES;
```

## Related

- [Undo Management](../../05-undo/undo-management.md).
- [Undo Retention](../../05-undo/undo-retention.md).
- [ORA-01555](../../26-errors/ora-01555.md).

## References

- Oracle Database Administrator's Guide 19c — Managing Undo
