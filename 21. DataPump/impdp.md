# IMPDP — Data Pump Import

## Overview

`impdp` is the Data Pump import client. It reads a dump file (or pulls over a network link) and re-creates the objects and data on the target. Like `expdp`, it runs server-side with a master table and parallel workers, and is restartable.

Import has more knobs than export because you often want to **remap** (`REMAP_SCHEMA`, `REMAP_TABLESPACE`), **transform** (`TRANSFORM=`), **filter** (`EXCLUDE`, `INCLUDE`), or **preview** (`SQLFILE=`) before applying.

## Prerequisites

1. Directory object with `READ, WRITE`.
2. Target schemas / tablespaces exist (or use `REMAP_SCHEMA` / `REMAP_TABLESPACE`).
3. Import user privileges — typically `DATAPUMP_IMP_FULL_DATABASE`.
4. `Enough` UNDO and TEMP; imports can be undo-intensive.

## Basic Command

```bash
impdp system/...@target      \
      DIRECTORY=DP_DUMP       \
      DUMPFILE=app_%U.dmp    \
      LOGFILE=app_impdp.log  \
      SCHEMAS=APP             \
      PARALLEL=4              \
      JOB_NAME=IMP_APP
```

## Key Parameters (delta from expdp)

| Parameter                    | Purpose                                                                        |
| ---------------------------- | ------------------------------------------------------------------------------ |
| `TABLE_EXISTS_ACTION`        | `SKIP` (default when no `CONTENT=DATA_ONLY`), `APPEND`, `TRUNCATE`, `REPLACE`. |
| `REMAP_SCHEMA=SRC:TGT`       | Rename schema.                                                                 |
| `REMAP_TABLESPACE=SRC:TGT`   | Rename tablespace.                                                             |
| `REMAP_TABLE=SCHEMA.SRC:TGT` | Rename table.                                                                  |
| `REMAP_DATAFILE=OLD:NEW`     | Rename datafile paths (transportable / full).                                  |
| `REMAP_DIRECTORY=OLD:NEW`    | Rename directory objects.                                                      |
| `TRANSFORM=`                 | On-the-fly DDL transform (see below).                                          |
| `SQLFILE=out.sql`            | Don't import — write DDL to file.                                              |
| `CONTENT`                    | `ALL`, `METADATA_ONLY`, `DATA_ONLY`.                                           |
| `INCLUDE` / `EXCLUDE`        | Object filters.                                                                |
| `DATA_OPTIONS`               | `DISABLE_APPEND_HINT`, `SKIP_CONSTRAINT_ERRORS`.                               |
| `PARTITION_OPTIONS`          | `NONE`, `DEPARTITION`, `MERGE`.                                                |
| `DISABLE_ARCHIVE_LOGGING=Y`  | Turn off redo generation during import (19c; requires FORCE LOGGING off).      |

## TRANSFORM Options

`TRANSFORM=<option>:<value>[:<object_type>]`

| Transform                  | Value                                | Effect                                         |
| -------------------------- | ------------------------------------ | ---------------------------------------------- |
| `SEGMENT_ATTRIBUTES`       | `Y` \| `N`                           | Preserve or strip STORAGE, TABLESPACE clauses. |
| `STORAGE`                  | `Y` \| `N`                           | Preserve or strip STORAGE clause only.         |
| `OID`                      | `Y` \| `N`                           | Preserve or generate new OIDs on object types. |
| `PCTSPACE`                 | `N%`                                 | Scale storage percentages.                     |
| `TABLE_COMPRESSION_CLAUSE` | `NONE` \| `COMPRESS` \| `NOCOMPRESS` | Rewrite compression clause.                    |
| `LOB_STORAGE`              | `SECUREFILE` \| `BASICFILE`          | Convert LOB storage.                           |
| `DISABLE_ARCHIVE_LOGGING`  | `Y`                                  | Import in NOLOGGING (19c only).                |

Example — strip TABLESPACE clauses and put everything into USERS:

```bash
impdp system/... schemas=APP directory=DP_DUMP dumpfile=app.dmp \
      remap_tablespace=USERS_DATA:USERS,USERS_IDX:USERS \
      transform=segment_attributes:N
```

## Common Patterns

### 1. Schema Rename

```bash
impdp system/... schemas=APP remap_schema=APP:APP_QA \
      remap_tablespace=USERS_DATA:QA_DATA \
      directory=DP_DUMP dumpfile=app_%U.dmp table_exists_action=SKIP \
      logfile=app_impdp.log
```

### 2. DDL-Only Preview

```bash
impdp system/... schemas=APP \
      directory=DP_DUMP dumpfile=app_%U.dmp \
      sqlfile=app_ddl.sql
```

Nothing is imported; DDL is written to `app_ddl.sql` inside the directory.

### 3. Data-Only Reload

```bash
impdp system/... schemas=APP content=DATA_ONLY \
      table_exists_action=TRUNCATE \
      directory=DP_DUMP dumpfile=app_%U.dmp
```

### 4. Table Subset with Rename

```bash
impdp system/... tables=APP.ORDERS \
      remap_table=APP.ORDERS:ORDERS_ARCHIVE \
      remap_schema=APP:HIST directory=DP_DUMP dumpfile=orders_2020.dmp
```

### 5. Network Import (No Dump File)

```bash
impdp system/...@target network_link=SRC_DB schemas=APP \
      remap_schema=APP:APP_NEW parallel=4
```

`SRC_DB` is a database link created on the target that points at the source.

### 6. Import Excluding Statistics + Indexes (Faster, Rebuild Later)

```bash
impdp system/... schemas=APP directory=DP_DUMP dumpfile=app_%U.dmp \
      exclude=statistics,index parallel=8 \
      logfile=app_impdp.log

-- After import completes
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS('APP', DEGREE => 16);
END;
/

-- Rebuild indexes from a pre-extracted DDL file
@app_index_ddl.sql
```

For very large loads, this "load naked table, index after" pattern is often 3–5x faster.

### 7. Full Import into a New Database

```bash
impdp system/... full=YES directory=DP_DUMP dumpfile=full_%U.dmp \
      parallel=8 \
      exclude=statistics \
      remap_tablespace=SOURCE_TS:USERS \
      logfile=full_impdp.log
```

## Monitoring

```sql
SELECT owner_name, job_name, operation, state,
       degree, attached_sessions
FROM   dba_datapump_jobs;

SELECT   sid, serial#, sofar, totalwork,
         ROUND(sofar/totalwork*100,1) pct
FROM     v$session_longops
WHERE    opname LIKE 'DATAPUMP%'
ORDER BY start_time DESC;
```

Attach an already-running job to change parallelism or watch status:

```bash
impdp system/... attach=IMP_APP
Import> status
Import> parallel=8
Import> continue_client
```

## Reading a Log Fast

```
Master table "SYSTEM"."IMP_APP" successfully loaded/unloaded
Starting "SYSTEM"."IMP_APP":  ...
Processing object type SCHEMA_EXPORT/USER
Processing object type SCHEMA_EXPORT/TABLE/TABLE
. . imported "APP"."CUST"                              4.2 GB   28,340,192 rows
Processing object type SCHEMA_EXPORT/TABLE/INDEX/INDEX
```

Look for lines starting with `ORA-` for errors; `. . imported` for row counts; `Processing object type` shows phase.

## Common Issues

- **`ORA-39002: invalid operation`** — Usually mode mismatch; e.g., `SCHEMAS=` with a full-mode dump. Match the mode used at export.
- **`ORA-39083: object type ... failed to create`** — DDL error (missing tablespace, mismatched types). Fix DDL and rerun with `SQLFILE=` first.
- **`ORA-39082: object created with compilation warnings`** — PL/SQL objects imported before their dependencies. Recompile with `UTL_RECOMP.RECOMP_SERIAL` after import.
- **`ORA-31684: Object type ... already exists`** — `TABLE_EXISTS_ACTION=SKIP` (default). Change to `APPEND`, `TRUNCATE`, or `REPLACE`.
- **`ORA-01950: no privileges on tablespace 'USERS'`** — Target user has no quota. `ALTER USER ... QUOTA UNLIMITED ON USERS`.
- **Import wildly slow** — Indexes and constraints being built row-by-row; use `exclude=INDEX,STATISTICS`, then rebuild after.
- **Transport tablespace fails on endian mismatch** — Use RMAN `CONVERT` or `CONVERT DATAFILE` first.

## Best Practices

1. Always dry-run with `SQLFILE=` on a schema-mode import to inspect DDL before applying.
2. Use `EXCLUDE=INDEX,STATISTICS,GRANTS` for the bulk load, restore them afterward.
3. Set `PARALLEL` = number of dump files (`%U` files at export time).
4. Use `REMAP_TABLESPACE` + `TRANSFORM=SEGMENT_ATTRIBUTES:N` when the target has a simpler tablespace layout.
5. `TABLE_EXISTS_ACTION=TRUNCATE` for iterative reloads during development.
6. Gather stats **after** import (`DBMS_STATS.GATHER_SCHEMA_STATS`).
7. `DISABLE_ARCHIVE_LOGGING=Y` (19c) shaves 20–40% off large loads when archivelog isn't required for the operation.
8. Test the network mode timing before scheduling — WAN latency dominates.
9. Preserve the export log alongside the import log — audit trail.
10. Never import into production without a full RMAN backup of the target first.

## Interview Questions

1. **Q:** What does `TABLE_EXISTS_ACTION=APPEND` do?
   **A:** Insert dump rows into the existing table without touching its structure.

2. **Q:** How do you convert LOBs from BasicFile to SecureFile during import?
   **A:** `TRANSFORM=LOB_STORAGE:SECUREFILE`.

3. **Q:** How do you preview the DDL a dump would create?
   **A:** `SQLFILE=out.sql` — no rows imported, DDL written to file.

4. **Q:** Import is slow. What are the first three things you check?
   **A:** PARALLEL matches file count, indexes/constraints excluded from bulk phase, archivelog + FORCE LOGGING implications on redo.

5. **Q:** How do you import into a differently-named tablespace?
   **A:** `REMAP_TABLESPACE=OLD:NEW` (plus `TRANSFORM=SEGMENT_ATTRIBUTES:N` to strip other storage clauses).

## References

- Oracle Database Utilities 19c — Data Pump Import
- MOS Doc ID 553337.1 — Data Pump Master Note
- MOS Doc ID 1354063.1 — TRANSFORM options
