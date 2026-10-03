# Flashback Database

## Overview

**Flashback Database** rewinds the entire database to a past SCN or time — as if you had done PITR, but without restoring backups. Uses a separate log stream: **flashback logs** in the FRA that capture pre-images of changed blocks. On flashback, Oracle applies those pre-images in reverse.

Flashback Database is the fastest way to recover a whole database from user error, bad batch, or bad deployment — often minutes vs hours for RMAN PITR.

## Prerequisites

- **ARCHIVELOG mode.**
- **Flashback logs** enabled in FRA.
- `db_flashback_retention_target` in minutes.
- Database in MOUNT for the flashback operation itself.

## Enabling

```sql
-- Confirm ARCHIVELOG
SELECT log_mode FROM v$database;

-- FRA sized
ALTER SYSTEM SET db_recovery_file_dest_size = 200G SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest = '+RECO' SCOPE=BOTH;

-- Retention target (minutes)
ALTER SYSTEM SET db_flashback_retention_target = 4320 SCOPE=BOTH;  -- 72h

-- Enable
ALTER DATABASE FLASHBACK ON;

SELECT flashback_on FROM v$database;
```

Flashback logs grow with change rate. Monitor:

```sql
SELECT * FROM v$flashback_database_log;
```

## Flashback Operation

```sql
-- Determine target SCN
SELECT current_scn FROM v$database;                -- capture before incident

-- Or find approximate SCN for a time
SELECT TIMESTAMP_TO_SCN(TIMESTAMP '2026-08-06 14:30:00') FROM dual;

-- Flashback
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO SCN 1234567890;
-- or
FLASHBACK DATABASE TO TIMESTAMP TIMESTAMP '2026-08-06 14:30:00';
-- or
FLASHBACK DATABASE TO RESTORE POINT before_batch;

-- Verify data
ALTER DATABASE OPEN READ ONLY;
-- Query to confirm
-- If good, close and reopen with resetlogs
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE OPEN RESETLOGS;
```

Or if verified:

```sql
ALTER DATABASE OPEN RESETLOGS;
```

`OPEN RESETLOGS` creates a new incarnation. Post-flashback backups are on the new incarnation.

## Guaranteed Restore Points

Regular flashback logs age out per `db_flashback_retention_target`. A **guaranteed restore point** pins the database's ability to flashback to that SCN, regardless of retention:

```sql
CREATE RESTORE POINT before_upgrade GUARANTEE FLASHBACK DATABASE;

-- Do the risky operation

-- Rollback if needed
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO RESTORE POINT before_upgrade;
ALTER DATABASE OPEN RESETLOGS;

-- Or drop the restore point when confident
DROP RESTORE POINT before_upgrade;
```

Guaranteed restore points can consume significant FRA space — Oracle keeps every changed block since the restore point.

## PDB Flashback (12c+)

With local UNDO enabled, individual PDBs can be flashed back:

```sql
ALTER PLUGGABLE DATABASE hrpdb CLOSE;
FLASHBACK PLUGGABLE DATABASE hrpdb TO TIMESTAMP TIMESTAMP '2026-08-06 14:30';
ALTER PLUGGABLE DATABASE hrpdb OPEN RESETLOGS;
```

## Data Guard Interaction

- Primary flashback breaks standby SCN alignment. Standby must also flashback to matching SCN or be recreated.
- Flashback the standby similarly using the same SCN.

## Common Issues

- **`ORA-19809: limit exceeded for recovery files`** — FRA full. Enlarge or clean up.
- **Cannot flash back far enough** — `db_flashback_retention_target` too short OR guaranteed restore point too old with pressure.
- **`ORA-38729: not enough flashback database log data`** — Same.
- **PDB flashback without local UNDO** — Not supported.
- **Slow flashback** — Very deep flashback = many flashback log applies. Estimate time via `V$FLASHBACK_DATABASE_STAT`.

## Diagnostic Queries

```sql
-- Flashback state
SELECT flashback_on FROM v$database;

-- Flashback log stats
SELECT oldest_flashback_scn, oldest_flashback_time,
       retention_target, flashback_size/1024/1024/1024 AS gb
FROM   v$flashback_database_log;

-- Recent flashback activity
SELECT begin_time, end_time,
       ROUND(flashback_data/1024/1024, 1) AS flashback_mb,
       ROUND(db_data/1024/1024, 1) AS db_mb,
       ROUND(redo_data/1024/1024, 1) AS redo_mb
FROM   v$flashback_database_stat
ORDER  BY begin_time DESC
FETCH FIRST 24 ROWS ONLY;

-- Restore points
SELECT name, scn, time, guarantee_flashback_database,
       storage_size/1024/1024/1024 AS gb
FROM   v$restore_point;

-- FRA usage
SELECT * FROM v$recovery_area_usage;
```

## Best Practices

1. **Enable Flashback Database** on production. RTO benefit is substantial.
2. `db_flashback_retention_target` ≥ 24 hours (1440 minutes).
3. FRA sized for flashback + archive + backups + Data Guard traffic.
4. **Create guaranteed restore point** before every major deployment / batch.
5. Drop guaranteed restore points promptly after success — they consume FRA.
6. Monitor `V$FLASHBACK_DATABASE_LOG.OLDEST_FLASHBACK_SCN` — the earliest reachable SCN.
7. Coordinate with Data Guard when flashing back primary.
8. Test flashback on staging.
9. Take a fresh L0 backup after flashback + OPEN RESETLOGS.
10. Alert on `flashback logfile write` I/O issues.

## Interview Questions

1. **Q:** Flashback Database vs PITR?
   **A:** Flashback rewinds using flashback logs (fast). PITR restores from backup + applies redo (slower).

2. **Q:** How to enable?
   **A:** ARCHIVELOG + `db_recovery_file_dest_size` + `db_flashback_retention_target` + `ALTER DATABASE FLASHBACK ON`.

3. **Q:** Guaranteed restore point?
   **A:** Pins flashback capability to a specific SCN regardless of retention target.

4. **Q:** Effect on Data Guard?
   **A:** Standby must be flashed back to same SCN or recreated.

5. **Q:** How do you flash back a PDB?
   **A:** Local UNDO enabled + `FLASHBACK PLUGGABLE DATABASE <name> TO TIMESTAMP ...`.

6. **Q:** After flashback?
   **A:** `OPEN RESETLOGS` — new incarnation. Fresh L0 backup.

## References

- Oracle Database Backup and Recovery User's Guide 19c — Flashback Database
- MOS Doc ID 565535.1 — Flashback Database
- MOS Doc ID 1550116.1 — Data Guard + Flashback
