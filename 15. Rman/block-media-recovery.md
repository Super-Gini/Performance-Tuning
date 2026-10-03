# Block Media Recovery

## Overview

**Block Media Recovery (BMR)** recovers individual corrupt blocks without offlining the datafile or the database. RMAN restores just the corrupt blocks from a backup and applies redo to bring them current — surgical repair.

Requires **ARCHIVELOG** mode and a backup containing the good version of the blocks.

## When BMR Applies

- Physical block corruption (`ORA-01578`, `ORA-01578`).
- Logical block corruption (`ORA-01578` + `ORA-08103`).
- User-reported specific bad rows.
- Corruption detected by RMAN `VALIDATE` or automatic block checking.

Not for widespread corruption (many blocks) — restore the datafile.

## Detecting Corruption

### Automatic (during I/O)

Set `db_block_checking = MEDIUM` and `db_block_checksum = TYPICAL`. Corrupt blocks caught on read.

Wait event: `ORA-01578: ORACLE data block corrupted (file # X, block # Y)`.

### RMAN VALIDATE

```rman
BACKUP VALIDATE CHECK LOGICAL DATABASE;
```

Populates `V$DATABASE_BLOCK_CORRUPTION` and `V$NONLOGGED_BLOCK`:

```sql
SELECT file#, block#, blocks, corruption_type
FROM   v$database_block_corruption;
```

### dbverify

Offline check on individual datafiles:

```bash
dbv file=/u02/orcl/users01.dbf blocksize=8192
```

## Performing BMR

Once you have file # and block # from `V$DATABASE_BLOCK_CORRUPTION`:

```rman
$ rman target /

RMAN> RECOVER DATAFILE 5 BLOCK 12345;
```

Multi-block:

```rman
RMAN> RECOVER DATAFILE 5 BLOCK 12345 TO 12350;
```

Multiple files:

```rman
RMAN> RECOVER
        DATAFILE 5 BLOCK 12345,
        DATAFILE 7 BLOCK 88999;
```

Auto-select from `V$DATABASE_BLOCK_CORRUPTION`:

```rman
RMAN> RECOVER CORRUPTION LIST;
```

## Auto Block Recovery on Standby (11g+)

Active Data Guard automatically fetches good blocks from the standby to fix corrupt primary blocks — no user action needed.

Requires ADG enabled and standby healthy.

## Behind the Scenes

1. RMAN identifies the last backup containing the block (`V$BACKUP_DATAFILE`).
2. Reads that block from backup.
3. Applies redo from the block's SCN forward until fully current.
4. Writes back to datafile — atomic replacement.

## Diagnostic Queries

```sql
-- Known corruption
SELECT file#, block#, blocks, corruption_type, corruption_change#
FROM   v$database_block_corruption;

-- Nonlogged blocks (from NOLOGGING ops after backup)
SELECT file#, block#, blocks
FROM   v$nonlogged_block;

-- Recent block corruption in alert log
SELECT originating_timestamp, message_text
FROM   v$diag_alert_ext
WHERE  message_text LIKE '%ORA-01578%'
ORDER  BY originating_timestamp DESC
FETCH FIRST 10 ROWS ONLY;
```

## Common Issues

- **No backup with a good block version** — Cannot BMR. Restore datafile.
- **`RMAN-06075: bad checksum in backup piece`** — Backup itself corrupt. Try alternative copy.
- **Block is a temp / undo block** — BMR limited; may need tablespace recovery.
- **Corruption in SYSTEM/UNDO** — Very serious; escalate to Oracle Support.

## Best Practices

1. Enable `db_block_checking = MEDIUM` and `db_block_checksum = TYPICAL`.
2. RMAN `VALIDATE CHECK LOGICAL DATABASE` weekly.
3. Active Data Guard for automatic block repair.
4. Alert on any block corruption.
5. Investigate root cause — hardware, storage firmware, memory error.
6. Enable **Lost Write Protection** (`db_lost_write_protect=TYPICAL`) on Data Guard configurations.
7. Keep backups covering the potential corruption time.
8. After BMR, run VALIDATE on the affected file — confirm no other corruption.
9. Consider `dbverify` cross-check during change windows.
10. For NOLOGGING-source corruption, gather redo strategies (FORCE LOGGING).

## Interview Questions

1. **Q:** What is BMR?
   **A:** Block Media Recovery — restore specific corrupt blocks from backup and recover, without full datafile restore.

2. **Q:** Command?
   **A:** `RMAN> RECOVER DATAFILE n BLOCK X;` or `RECOVER CORRUPTION LIST;`.

3. **Q:** Where is corruption tracked?
   **A:** `V$DATABASE_BLOCK_CORRUPTION`.

4. **Q:** Automatic recovery from ADG standby?
   **A:** 11g+ automatic; primary fetches good block from standby.

5. **Q:** When BMR isn't enough?
   **A:** Widespread corruption, SYSTEM/UNDO involvement, no good backup — restore datafile.

6. **Q:** How to detect corruption proactively?
   **A:** `BACKUP VALIDATE CHECK LOGICAL DATABASE;`, `db_block_checking`, `db_block_checksum`, `dbv`.

## References

- Oracle Database Backup and Recovery User's Guide 19c — Block Media Recovery
- MOS Doc ID 336133.1 — BMR
- MOS Doc ID 122183.1 — Detecting Block Corruption
- MOS Doc ID 1088018.1 — Automatic Block Repair on ADG
