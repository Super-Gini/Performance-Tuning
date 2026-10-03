# Control File Repository

## Overview

When RMAN runs without a recovery catalog, it uses the **target's control file** as its metadata store. All backup pieces, backup sets, image copies, and archive log records live in the control file's reusable records section. Bounded by `CONTROL_FILE_RECORD_KEEP_TIME` (default 7 days) — records older than that may be overwritten.

For small standalone databases this is fine. For production estates with retention needs beyond a week, use a [recovery catalog](recovery-catalog.md).

## Control File Sections Used by RMAN

- Backup Set
- Backup Piece
- Backup Archive Log
- Backup Datafile
- Backup Spfile
- Backup Redolog (rare)
- Image Copy
- Deleted Object
- Proxy Copy
- Copy Corruption

Check current usage:

```sql
SELECT type, records_total, records_used,
       ROUND(records_used/DECODE(records_total,0,1,records_total)*100, 1) AS pct
FROM   v$controlfile_record_section
WHERE  records_total > 0
ORDER  BY records_used DESC;
```

## `CONTROL_FILE_RECORD_KEEP_TIME`

```sql
ALTER SYSTEM SET control_file_record_keep_time = 31 SCOPE=BOTH;
```

Days to keep reusable records before allowing overwrite. Should equal or exceed your retention policy — otherwise the control file may age out records for backups you're still counting on.

## Autobackup

Even without a catalog, always enable:

```
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '+RECO/%d/autobackup/%F';
```

After every backup, structural change, or `BACKUP` command, RMAN writes an autobackup of the control file + SPFILE. Essential for disaster recovery when both control files are lost.

Autobackup format `%F` yields `c-<DBID>-<YYYYMMDD>-<SEQ>`. RMAN can find and restore autobackups by DBID even without knowing the exact filename.

## Recovering With Only the Control File

Without a catalog, if you lose all control files you need:

1. **DBID** — pre-record it. `SELECT dbid FROM v$database;`
2. **Autobackup location** — where autobackups live (usually FRA).

Then:

```rman
rman target /
RMAN> SET DBID 1234567890;
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> ALTER DATABASE MOUNT;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

## Diagnostic Queries

```sql
-- Records approaching capacity
SELECT type, record_size, records_total, records_used,
       ROUND(records_used/DECODE(records_total,0,1,records_total)*100, 1) AS pct
FROM   v$controlfile_record_section
WHERE  records_used > records_total * 0.7
ORDER  BY pct DESC;

-- Backup pieces recorded in control file
SELECT bp.piece#, bp.handle, bp.tag, bp.completion_time,
       bs.set_stamp, bs.set_count
FROM   v$backup_piece bp JOIN v$backup_set bs
       ON bp.set_stamp = bs.set_stamp AND bp.set_count = bs.set_count
WHERE  bp.completion_time > SYSDATE - 7
ORDER  BY bp.completion_time DESC;
```

## When It's Enough

Control-file-only is enough when:

- Single or few databases.
- Retention ≤ 30 days (bounded by control file size).
- No cross-database reporting needs.
- No stored scripts requirement.
- No PDB unplug/plug where old backup metadata may be needed.

## When to Adopt a Catalog

Adopt when:

- Retention > 30 days.
- Multiple databases (5+).
- Need for stored scripts.
- Regulatory retention audits.
- Frequent duplicates from old backups.

## Best Practices

1. **`CONTROLFILE AUTOBACKUP ON`** — mandatory.
2. `CONTROL_FILE_RECORD_KEEP_TIME` ≥ retention policy days.
3. Multiplex control files.
4. Record and safeguard the DBID.
5. Store autobackups off-site.
6. Monitor `V$CONTROLFILE_RECORD_SECTION` for records approaching capacity.
7. Adopt a catalog when the estate grows.
8. Test control file recovery quarterly.

## Interview Questions

1. **Q:** RMAN without catalog?
   **A:** Uses target's control file as metadata store — bounded by `CONTROL_FILE_RECORD_KEEP_TIME`.

2. **Q:** What limits control-file-based repository?
   **A:** Fixed sections and `CONTROL_FILE_RECORD_KEEP_TIME`. Records age out.

3. **Q:** How to recover from complete control file loss?
   **A:** `SET DBID`, `STARTUP NOMOUNT`, `RESTORE CONTROLFILE FROM AUTOBACKUP`, mount, recover, open resetlogs.

4. **Q:** Why enable autobackup?
   **A:** Provides a recoverable copy of the control file even without a catalog.

5. **Q:** When adopt a catalog?
   **A:** Multiple databases, long retention, stored scripts, regulatory needs.

## References

- Oracle Database Backup and Recovery User's Guide 19c
- MOS Doc ID 549174.1 — Restore Control File From Autobackup
- MOS Doc ID 388422.1 — RMAN Repository
