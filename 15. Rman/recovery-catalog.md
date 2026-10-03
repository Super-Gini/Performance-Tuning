# Recovery Catalog

## Overview

The **recovery catalog** is a separate schema (in a separate database) that stores RMAN backup metadata for one or more target databases. Without a catalog, RMAN uses the target's control file as its metadata store — limited by `CONTROL_FILE_RECORD_KEEP_TIME` (default 7 days).

For any production Oracle estate with more than a couple of databases, a recovery catalog is standard practice.

## Advantages

- **Retention beyond `CONTROL_FILE_RECORD_KEEP_TIME`** — record every backup indefinitely.
- **Central inventory** — one catalog holds metadata for many targets.
- **Stored scripts** — `CREATE SCRIPT` recallable by name.
- **Cross-database reporting** — "which databases have backups older than 7 days?".
- **Simpler `DUPLICATE`** — target's control file may not know about the backup you want to duplicate from.
- **Extended metadata** — some RMAN features work only with catalog.

## Prerequisites

- Dedicated Oracle database — small (a few GB). Same or higher version than the highest-version target.
- Tablespace for catalog data.
- Schema user (typically `RMAN`).

## Creating a Catalog

### 1. On the catalog database

```sql
-- As SYS
CREATE TABLESPACE rman_cat DATAFILE '+DATA/RMANCAT/rman.dbf' SIZE 5G AUTOEXTEND ON MAXSIZE 20G;

CREATE USER rman IDENTIFIED BY "StrongPwd"
  DEFAULT TABLESPACE rman_cat
  QUOTA UNLIMITED ON rman_cat;

GRANT RECOVERY_CATALOG_OWNER TO rman;
```

### 2. Create schema objects

```bash
rman catalog rman/pwd@rmancat
RMAN> CREATE CATALOG;
```

### 3. Register each target database

```bash
rman target sys/pwd@prod catalog rman/pwd@rmancat
RMAN> REGISTER DATABASE;
```

## Using the Catalog

Once registered, connect with both:

```bash
rman target / catalog rman/pwd@rmancat
```

All backup / recovery commands work identically; RMAN records metadata in both control file and catalog.

## Catalog Views

Query the RMAN schema (or use RMAN commands):

```sql
-- Registered databases
SELECT db_key, name, dbid FROM rman.rc_database;

-- Backup pieces per target
SELECT rc.name, bp.session_key, bp.piece#, bp.handle, bp.tag
FROM   rman.rc_database rc JOIN rman.rc_backup_piece bp ON bp.db_id = rc.dbid
ORDER  BY rc.name, bp.completion_time DESC
FETCH FIRST 20 ROWS ONLY;

-- Archive logs backed up
SELECT rc.name, rl.thread#, rl.sequence#, rl.first_time, rl.backup_count
FROM   rman.rc_database rc JOIN rman.rc_archived_log rl ON rl.db_id = rc.dbid
WHERE  rl.first_time > SYSDATE - 7
ORDER  BY rc.name, rl.first_time;
```

## Stored Scripts

```rman
CREATE SCRIPT weekly_backup {
  BACKUP AS COMPRESSED BACKUPSET
    INCREMENTAL LEVEL 0
    DATABASE PLUS ARCHIVELOG DELETE INPUT;
  DELETE NOPROMPT OBSOLETE;
}

-- Execute
RUN { EXECUTE SCRIPT weekly_backup; }

-- List
LIST SCRIPT NAMES;
```

Scripts stored per target (or global with `GLOBAL SCRIPT`).

## Resync

If catalog is disconnected during backup, resync it later:

```rman
RMAN> RESYNC CATALOG;
```

Ensures catalog knows about backups that happened while offline. Fully automatic when connected.

## Upgrading the Catalog

When you upgrade the target to a new Oracle release, may need to upgrade catalog schema:

```rman
RMAN> UPGRADE CATALOG;
-- confirmed twice
```

## Backing Up the Catalog

The catalog is itself an Oracle database — back it up like any other, but _not_ with itself. Use a smaller local RMAN backup script.

If you lose the catalog and only have control-file-based metadata for targets, re-register:

```rman
RMAN> RESET DATABASE;   -- if incarnations look off
```

## Deregistering

If a target database is decommissioned:

```rman
RMAN> CONNECT TARGET /
RMAN> CONNECT CATALOG rman/pwd@rmancat
RMAN> UNREGISTER DATABASE;
```

## Virtual Private Catalog

A single catalog database can host **VPCs** — sub-schemas that hide targets from each other:

```rman
CREATE VIRTUAL CATALOG;
```

Useful for multi-tenant DBaaS.

## Common Issues

- **`RMAN-06004: ORACLE error from recovery catalog database`** — Catalog connection down. Retry, or use `NOCATALOG`.
- **Catalog schema version mismatch** — `UPGRADE CATALOG`.
- **Registration fails** — DBID conflict (very rare); check `V$DATABASE.DBID`.
- **Catalog DB full** — Enlarge `rman_cat` tablespace.

## Best Practices

1. **Dedicated catalog database.** Small, well-protected, backed up.
2. Catalog Oracle version ≥ highest target.
3. Register every production target.
4. Use stored scripts for standard workflows.
5. Backup the catalog DB regularly.
6. Alert on `RMAN-06004` (catalog unavailable) — backups still work via control file, but you lose central metadata.
7. Retention in catalog can exceed control file window — key advantage.
8. Purge obsolete records via `DELETE OBSOLETE`.
9. Consider VPC for multi-tenant environments.
10. Keep catalog database up-to-date on patches.

## Interview Questions

1. **Q:** What is the recovery catalog?
   **A:** A separate schema/database storing RMAN backup metadata for one or more target databases.

2. **Q:** Why use it?
   **A:** Retention beyond `CONTROL_FILE_RECORD_KEEP_TIME`, central inventory, stored scripts, cross-DB reporting.

3. **Q:** Where does it live?
   **A:** Separate Oracle database, schema owned by `RMAN` user with `RECOVERY_CATALOG_OWNER` role.

4. **Q:** How to register?
   **A:** `RMAN> REGISTER DATABASE;` after connecting to both target and catalog.

5. **Q:** Backup the catalog itself?
   **A:** Yes — treat it like any Oracle DB with its own RMAN script.

6. **Q:** VPC?
   **A:** Virtual Private Catalog — sub-schema isolation for multi-tenant catalog hosting.

## References

- Oracle Database Backup and Recovery User's Guide 19c
- MOS Doc ID 388422.1 — RMAN Overview
- MOS Doc ID 132904.1 — Recovery Catalog Setup
