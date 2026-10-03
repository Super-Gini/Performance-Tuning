# Restore

## Overview

`RESTORE` is the RMAN command that copies files from backup pieces to their original (or new) locations. It's the first step of recovery — after RESTORE, you `RECOVER` to apply redo.

RMAN can restore anything it backed up: datafiles, control files, SPFILE, archive logs, tempfiles (only if included in backup), and even PDB pieces.

## Basic Syntax

```rman
RESTORE DATABASE;
RESTORE TABLESPACE users;
RESTORE DATAFILE 5;
RESTORE CONTROLFILE FROM AUTOBACKUP;
RESTORE SPFILE FROM AUTOBACKUP;
RESTORE ARCHIVELOG ALL;
RESTORE ARCHIVELOG FROM SEQUENCE 1000 UNTIL SEQUENCE 1100;
```

## Restore to a New Location

```rman
RUN {
  SET NEWNAME FOR DATAFILE 5 TO '+DATA2/PROD/DATAFILE/users_new.dbf';
  RESTORE DATAFILE 5;
  SWITCH DATAFILE 5;    -- update control file with new location
  RECOVER DATAFILE 5;
  ALTER DATABASE DATAFILE 5 ONLINE;
}
```

`SET NEWNAME` + `SWITCH` is the standard pattern for relocating datafiles during restore.

## Restore to a Different Point in Time

Preludes to PITR — see [PITR](pitr.md):

```rman
RUN {
  SET UNTIL TIME "TO_DATE('2026-08-06 14:30','YYYY-MM-DD HH24:MI')";
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}
```

## Restore Validate — Dry Run

```rman
RESTORE DATABASE VALIDATE;
RESTORE TABLESPACE users VALIDATE;
```

Reads all backup pieces to confirm they're readable without writing anything. Essential for backup verification.

## Restore Preview

```rman
RESTORE DATABASE PREVIEW SUMMARY;
```

Shows which backup pieces RMAN would use, without actually running the restore. Great for sizing recovery time estimates.

## Restore Control File

Three sources, tried in order:

1. **From catalog** (if using) — most reliable.
2. **From autobackup** — needs DBID.
3. **From explicit backup piece** — `RESTORE CONTROLFILE FROM '/backup/piece_of_ctrl.bkp'`.

Only viable in NOMOUNT (target hasn't read a control file yet).

## Restore SPFILE

Similar pattern:

```rman
RMAN> STARTUP NOMOUNT;   -- uses PFILE or default
RMAN> SET DBID 1234567890;
RMAN> RESTORE SPFILE FROM AUTOBACKUP;
```

## Restore Archive Logs

```rman
-- All available
RESTORE ARCHIVELOG ALL;

-- Range by sequence
RESTORE ARCHIVELOG FROM SEQUENCE 1000 UNTIL SEQUENCE 1100 THREAD 1;

-- Range by time
RESTORE ARCHIVELOG FROM TIME 'SYSDATE-2' UNTIL TIME 'SYSDATE-1';

-- To alternate destination
RUN {
  SET ARCHIVELOG DESTINATION TO '/staging/arch';
  RESTORE ARCHIVELOG FROM SEQUENCE 1000 UNTIL SEQUENCE 1100;
}
```

Restoring archive logs is common for standby gap resolution and PITR staging.

## Parallelism

RMAN uses channels for parallelism. Configured persistently or ad-hoc:

```rman
RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c2 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c3 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c4 DEVICE TYPE DISK;
  RESTORE DATABASE;
}
```

## Restore Encrypted Backups

```rman
-- Password-based
SET DECRYPTION IDENTIFIED BY 'BackupPassword';
RESTORE DATABASE;

-- Multiple passwords tried
SET DECRYPTION IDENTIFIED BY 'Password1','Password2';
```

Transparent (wallet) decryption: no explicit set needed if wallet open.

## Multi-Section

For very large datafiles or bigfile tablespaces, RMAN can restore sections in parallel per file:

```rman
BACKUP SECTION SIZE 32G DATAFILE 5;   -- backup side
RESTORE DATAFILE 5;                    -- restore automatically uses sections
```

## Diagnostic Queries

```sql
-- Recent restore jobs
SELECT session_key, start_time, end_time, status,
       ROUND(output_bytes/1024/1024/1024, 2) AS output_gb,
       ROUND(elapsed_seconds/60, 1) AS minutes
FROM   v$rman_status
WHERE  operation = 'RESTORE'
   AND start_time > SYSDATE - 7
ORDER  BY start_time DESC;

-- Files currently being restored
SELECT * FROM v$session_longops
WHERE  opname LIKE 'RMAN:%'
   AND totalwork > 0
ORDER  BY sofar DESC;

-- Backup pieces used
SELECT piece#, handle, tag, completion_time
FROM   v$backup_piece
WHERE  completion_time > SYSDATE - 7
ORDER  BY completion_time DESC;
```

## Common Issues

- **`RMAN-06026: some targets not found`** — Backup for the requested piece is missing. Try `LIST BACKUP` and use a different backup set.
- **`RMAN-19870` — error reading backup piece** — Media corruption. Try alternate copy.
- **Slow restore** — Add channels; check I/O to backup destination.
- **`RESTORE DATABASE` reads full backup even if some datafiles are already fine** — Use `RESTORE DATAFILE X` for targeted restore.
- **Wallet not open for encrypted backup** — Open wallet: `ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN`.

## Best Practices

1. **`RESTORE PREVIEW SUMMARY`** before actual restore — verify what pieces will be used.
2. **`VALIDATE`** as sanity check.
3. Use `SET NEWNAME` for relocation.
4. Parallelism at least 2× for large restores.
5. Multi-section backups for parallel restore of huge files.
6. Ensure wallet open before restoring encrypted backups.
7. Restore to a scratch machine periodically (test drill).
8. Alert on any failed RESTORE.
9. Store DBID and autobackup path in a well-known location (not just the DB).

## Interview Questions

1. **Q:** RESTORE vs RECOVER?
   **A:** RESTORE reads backup pieces and writes files. RECOVER applies redo.

2. **Q:** `RESTORE DATABASE PREVIEW SUMMARY`?
   **A:** Shows what backup pieces RMAN would use without executing.

3. **Q:** Restore to new location?
   **A:** `SET NEWNAME FOR DATAFILE n TO '<path>';` then `RESTORE` + `SWITCH`.

4. **Q:** Restore control file — prerequisites?
   **A:** Instance in NOMOUNT; DBID known if using autobackup.

5. **Q:** Encrypted backup restore?
   **A:** Password: `SET DECRYPTION IDENTIFIED BY 'pwd'`. Transparent: wallet must be open.

6. **Q:** Section size?
   **A:** Splits large datafile into sections; enables parallel backup and restore within a single file.

## References

- Oracle Database Backup and Recovery User's Guide 19c
- MOS Doc ID 464192.1 — Restore & Recovery
- MOS Doc ID 549174.1 — Restore Control File
