# Duplicate Database

## Overview

`RMAN DUPLICATE` creates a copy of a database — same data, potentially different DBID, name, and file layout — from RMAN backups or directly from the source (**active database duplication**). Used for:

- **Refresh non-prod** from a recent prod backup.
- **Create a Data Guard standby** (a specialized duplicate).
- **Point-in-time clone** to extract data as it was.
- **Move to new hardware** with minimal source downtime.

## Two Modes

- **Backup-based** — Duplicate from existing RMAN backup pieces on disk (or a shared mount).
- **Active database** — Duplicate directly over network from a running source. No prior backup needed.

## Backup-Based Duplicate

```rman
$ rman target sys/pwd@prod auxiliary sys/pwd@aux

RUN {
  DUPLICATE TARGET DATABASE TO auxdb
    UNTIL TIME "SYSDATE - 1"
    NOFILENAMECHECK
    DB_FILE_NAME_CONVERT ('/prod/', '/aux/')
    LOG_FILE_NAME_CONVERT ('/prod/', '/aux/');
}
```

## Active Database Duplicate

Requires network path between target and auxiliary; auxiliary must exist as NOMOUNT instance.

```rman
$ rman target sys/pwd@prod auxiliary sys/pwd@aux

RUN {
  DUPLICATE TARGET DATABASE TO auxdb
    FROM ACTIVE DATABASE
    NOFILENAMECHECK
    DB_FILE_NAME_CONVERT ('/prod/', '/aux/')
    LOG_FILE_NAME_CONVERT ('/prod/', '/aux/');
}
```

Options:

- `USING BACKUPSET` (12.2+) — send backupsets over network (compressed, encrypted); more efficient than image copies.
- `USING COMPRESSED BACKUPSET` — compress on-the-fly.

## Duplicate for Data Guard Standby

```rman
DUPLICATE TARGET DATABASE FOR STANDBY
  FROM ACTIVE DATABASE
  DORECOVER
  SPFILE
    SET DB_UNIQUE_NAME='prod_dr'
    SET LOG_ARCHIVE_DEST_2='SERVICE=prod ASYNC VALID_FOR=(ONLINE_LOGFILE,PRIMARY_ROLE)'
  NOFILENAMECHECK;
```

`FOR STANDBY` skips DBID reset (standby uses same DBID as primary). See [Data Guard](../17-data-guard/index.md).

## Preparing the Auxiliary Instance

1. Create the auxiliary directory structure.
2. Create a minimal PFILE for the auxiliary:

   ```
   db_name=auxdb
   db_unique_name=auxdb
   db_block_size=8192
   compatible=19.0.0
   memory_target=4G
   control_files='/aux/orcl/control01.ctl','/aux/orcl/control02.ctl'
   ```

3. Add password file: `orapwd file=$ORACLE_HOME/dbs/orapwauxdb password=x entries=5`.
4. Create tnsnames entries for both target and auxiliary.
5. Start auxiliary NOMOUNT: `startup nomount pfile=/aux/init.ora`.

## Point-in-Time Duplicate

For extracting data from a historical time:

```rman
DUPLICATE TARGET DATABASE TO auxdb
  UNTIL SCN 1234567890
  FROM ACTIVE DATABASE
  NOFILENAMECHECK;
```

## Skip Tablespace

Skip large tablespaces you don't need in the duplicate:

```rman
DUPLICATE TARGET DATABASE TO auxdb
  SKIP TABLESPACE staging, temp_reports
  FROM ACTIVE DATABASE;
```

Skipped tablespaces come back as OFFLINE/UNUSABLE in the duplicate.

## Duplicate a PDB

12.2+:

```rman
DUPLICATE PLUGGABLE DATABASE hrpdb TO auxdb
  FROM ACTIVE DATABASE
  UNTIL SCN 1234567890
  NOFILENAMECHECK;
```

## Rename / Convert Filenames

```rman
DB_FILE_NAME_CONVERT ('/prod/datafile/','/aux/datafile/')
LOG_FILE_NAME_CONVERT ('/prod/redo/','/aux/redo/')
```

Or explicit:

```rman
SET NEWNAME FOR DATAFILE 5 TO '/aux/users_new.dbf';
```

## Diagnostic Queries

Between duplicate runs, check auxiliary:

```sql
-- After DUPLICATE, connect to auxiliary
SELECT open_mode, database_role, dbid, name, db_unique_name
FROM   v$database;

-- Datafile status
SELECT file#, name, status FROM v$datafile;
```

## Common Issues

- **`ORA-17629: Cannot connect to the remote database server`** — Auxiliary can't reach target. Check TNS and firewall.
- **`RMAN-05541: no archived logs found in target database`** — Target hasn't archived enough logs for the requested recovery point.
- **Space in auxiliary destination** — Ensure enough disk for full DB size.
- **Auxiliary parameter mismatch** — `memory_target` too small for source's actual memory need.
- **`NOFILENAMECHECK` omitted, same filesystem** — RMAN objects to identical file names; either use CONVERT clauses or `NOFILENAMECHECK`.
- **PDB DUPLICATE fails on local undo** — Ensure local UNDO enabled.

## Best Practices

1. **Active duplicate over network** for freshness.
2. Use `USING BACKUPSET` for network efficiency (12.2+).
3. `SKIP TABLESPACE` for non-essential large tablespaces.
4. Automate refresh cycles: nightly / weekly.
5. Duplicate for DR standby with `FOR STANDBY` + `DORECOVER`.
6. Test with `PREVIEW` where possible.
7. Have enough archive logs on target during duplicate.
8. Auxiliary DBID may differ (RMAN sets new DBID by default).
9. Isolate auxiliary from production network segments to avoid mistakes.
10. Post-duplicate: apply data masking scripts for non-prod.

## Interview Questions

1. **Q:** What is RMAN DUPLICATE?
   **A:** Creates a copy of the database from backup or active source.

2. **Q:** Backup-based vs active duplication?
   **A:** Backup-based needs existing backup pieces. Active connects to running source over network.

3. **Q:** Duplicate for standby?
   **A:** `DUPLICATE ... FOR STANDBY DORECOVER` preserves DBID and prepares as physical standby.

4. **Q:** File name conversion?
   **A:** `DB_FILE_NAME_CONVERT` and `LOG_FILE_NAME_CONVERT` clauses, or `SET NEWNAME`.

5. **Q:** Skip large tablespaces?
   **A:** `SKIP TABLESPACE <name>` — comes back OFFLINE in duplicate.

6. **Q:** Prerequisites for active duplicate?
   **A:** Network path, auxiliary NOMOUNT with PFILE + password file + TNS entries.

## References

- Oracle Database Backup and Recovery User's Guide 19c — Duplicating Databases
- MOS Doc ID 452868.1 — RMAN DUPLICATE
- MOS Doc ID 1935365.1 — PDB DUPLICATE
