# V$DATABASE

## Purpose

One row describing the **database** (as opposed to the instance). Contains DBID, name, mode, log/flashback config, SCN, Data Guard role, control file info.

## Key Columns

| Column                         | Meaning                                                                  |
| ------------------------------ | ------------------------------------------------------------------------ |
| `DBID`                         | Unique DB identifier — critical for RMAN.                                |
| `NAME`                         | DB name (short, ≤ 8 chars).                                              |
| `DB_UNIQUE_NAME`               | Full unique name (Data Guard: primary vs standby names differ).          |
| `CREATED`                      | When DB was created.                                                     |
| `RESETLOGS_CHANGE#`            | SCN at last RESETLOGS.                                                   |
| `RESETLOGS_TIME`               | Timestamp of last RESETLOGS.                                             |
| `LOG_MODE`                     | `ARCHIVELOG` / `NOARCHIVELOG`.                                           |
| `OPEN_MODE`                    | `READ WRITE` / `READ ONLY` / `MOUNTED` / `READ ONLY WITH APPLY`.         |
| `DATABASE_ROLE`                | `PRIMARY` / `PHYSICAL STANDBY` / `LOGICAL STANDBY` / `SNAPSHOT STANDBY`. |
| `PROTECTION_MODE`              | `MAXIMUM PROTECTION` / `MAXIMUM AVAILABILITY` / `MAXIMUM PERFORMANCE`.   |
| `PROTECTION_LEVEL`             | Actual (may differ from mode during failover).                           |
| `FLASHBACK_ON`                 | `YES` / `NO`.                                                            |
| `CURRENT_SCN`                  | Current DB SCN.                                                          |
| `CONTROLFILE_TYPE`             | `CURRENT` / `STANDBY` / `BACKUP` / `CLONE`.                              |
| `CONTROLFILE_SEQUENCE#`        | Control file sequence.                                                   |
| `PLATFORM_ID`, `PLATFORM_NAME` | Endianness/platform.                                                     |
| `DB_UNIQUE_NAME`               | For Data Guard.                                                          |
| `CDB`                          | `YES` / `NO`.                                                            |

## Common Queries

```sql
-- Identity + role
SELECT name, db_unique_name, dbid, database_role, open_mode, log_mode
FROM   v$database;

-- Data Guard protection
SELECT protection_mode, protection_level, database_role, switchover_status
FROM   v$database;

-- Flashback enabled?
SELECT flashback_on FROM v$database;

-- Current SCN
SELECT current_scn FROM v$database;

-- RESETLOGS history
SELECT resetlogs_change#, resetlogs_time
FROM   v$database;
```

## Related

- `V$INSTANCE` — instance metadata.
- `V$CONTAINERS` — PDBs in a CDB.
- `V$PDBS` — PDB info.

## References

- Oracle Database Reference 19c — `V$DATABASE`
