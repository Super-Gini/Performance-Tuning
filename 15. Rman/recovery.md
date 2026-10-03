# Recovery

## Overview

"Recovery" in Oracle is the process of bringing a database (or datafile / tablespace / block) forward from a backup to a consistent state, applying redo. This page covers **complete recovery** — restore + apply all archive logs and online redo — to bring the DB fully up to date.

Point-in-time recovery is [PITR](pitr.md). Tablespace-level is [TSPITR](tspitr.md). Block-level is [Block Media Recovery](block-media-recovery.md).

## When to Use Complete Recovery

- Media loss (datafile / tablespace gone).
- Corruption localized to a datafile.
- Restore for testing.
- Recovery from full backup after disaster.

## Prerequisites

- Valid backup covering the datafile(s).
- All archive logs from backup SCN to current.
- ARCHIVELOG mode.

## Full Database Recovery

Loss of many datafiles or entire database:

```
$ rman target /
RMAN> STARTUP MOUNT;
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN;
```

`RESTORE` copies datafiles from backup to their original locations (or new via `SET NEWNAME`). `RECOVER` applies redo from the backup SCN forward.

## Selective Datafile Recovery

Database can stay OPEN if the affected datafile isn't in SYSTEM or UNDO:

```
RMAN> SQL 'ALTER DATABASE DATAFILE 5 OFFLINE';
RMAN> RESTORE DATAFILE 5;
RMAN> RECOVER DATAFILE 5;
RMAN> SQL 'ALTER DATABASE DATAFILE 5 ONLINE';
```

If the datafile is in SYSTEM, database must be in MOUNT.

## Tablespace Recovery

```
RMAN> SQL 'ALTER TABLESPACE users OFFLINE IMMEDIATE';
RMAN> RESTORE TABLESPACE users;
RMAN> RECOVER TABLESPACE users;
RMAN> SQL 'ALTER TABLESPACE users ONLINE';
```

## Instance Recovery vs Media Recovery

- **Instance recovery** — after crash/ABORT; SMON applies redo from online logs. No RMAN involved.
- **Crash recovery** — subset of instance recovery, on startup.
- **Media recovery** — after restore from backup; RMAN or SQL\*Plus applies archive logs.

## Recovery Modes

- **Full** — apply all redo.
- **Incomplete** — stop at a specified SCN, time, or sequence (this is PITR).

## Diagnostic Queries

```sql
-- Which files need recovery?
SELECT file#, error, online, change# FROM v$recover_file;

-- Which redo logs needed for a datafile?
SELECT df.file#, df.name, dh.checkpoint_change# AS df_scn,
       (SELECT MAX(rl.next_change#) FROM v$archived_log rl
        WHERE  rl.next_change# > dh.checkpoint_change#) AS max_scn_available
FROM   v$datafile df JOIN v$datafile_header dh USING (file#)
WHERE  file# IN (SELECT file# FROM v$recover_file);

-- Archive logs available
SELECT thread#, sequence#, first_time, next_time, name
FROM   v$archived_log
ORDER  BY first_time DESC
FETCH FIRST 20 ROWS ONLY;
```

## The Recovery Process Under the Hood

1. **RESTORE** — RMAN reads backup pieces, decompresses / decrypts if needed, writes to datafile location.
2. **Datafile header SCN** = backup SCN.
3. **RECOVER**:
   - RMAN identifies archive logs needed (from `V$ARCHIVED_LOG`).
   - Applies them, one after another.
   - Applies online redo if available.
   - Datafile SCNs march forward.
4. **OPEN** — normal database open.

## Sample Full Restore + Recovery

```
$ rman target /
RMAN> STARTUP NOMOUNT;      -- new instance
RMAN> SET DBID 1234567890;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> ALTER DATABASE MOUNT;

RMAN> RESTORE DATABASE;

RMAN> RECOVER DATABASE;    -- applies all available archive logs

RMAN> ALTER DATABASE OPEN;   -- or OPEN RESETLOGS if incomplete
```

`OPEN RESETLOGS` is required after any incomplete recovery.

## Common Issues

- **`RMAN-06054: media recovery requesting unknown log`** — Missing archive log. Restore it or accept incomplete recovery.
- **`ORA-01113: file N needs media recovery`** — File not fully recovered. Continue recover or restore fresh.
- **`ORA-19870: error reading backup piece`** — Backup piece corrupted or missing. Try alternative copy.
- **Long recovery time** — Insufficient parallelism or large archive log volume. Increase channels, add BCT for future incrementals.

## Best Practices

1. **Always ARCHIVELOG.**
2. Multiplex archive logs to two destinations (FRA + off-site).
3. `CONFIGURE ARCHIVELOG DELETION POLICY` prevents unbacked logs from being deleted.
4. Store the DBID off-DB.
5. Test full recovery to a scratch host quarterly.
6. Retention ≥ RTO + margin.
7. Monitor `V$RECOVER_FILE`.
8. RMAN parallelism: `CONFIGURE DEVICE TYPE DISK PARALLELISM 4`.
9. Use `SECTION SIZE` for parallel restore of large datafiles.
10. Enable BCT for fast future incrementals (backup side; but recovery benefits too since fewer redo).

## Interview Questions

1. **Q:** What is complete recovery?
   **A:** Restore backup + apply all archive + online redo to bring datafile fully current.

2. **Q:** Restore vs Recover?
   **A:** RESTORE copies from backup. RECOVER applies redo forward.

3. **Q:** After incomplete recovery — what next?
   **A:** `ALTER DATABASE OPEN RESETLOGS` — resets log sequence.

4. **Q:** Can DB stay OPEN for datafile recovery?
   **A:** Yes if datafile not in SYSTEM/UNDO. Offline, restore, recover, online.

5. **Q:** How does RMAN know which archive logs to apply?
   **A:** From the datafile header SCN forward, via `V$ARCHIVED_LOG`.

6. **Q:** OPEN RESETLOGS behavior?
   **A:** Discards older redo, starts new incarnation. Required after incomplete recovery.

## References

- Oracle Database Backup and Recovery User's Guide 19c
- MOS Doc ID 388422.1 — RMAN
- MOS Doc ID 388422.1 — Recovery Overview
