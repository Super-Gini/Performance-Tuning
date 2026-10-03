# EXPDP — Data Pump Export

## Overview

`expdp` is the Data Pump export client. It launches a server-side job that reads metadata + data from the source database and writes to one or more dump files in an **Oracle Directory** object. Work is parallelized across `DWnn` workers coordinated by a `DMnn` master. Because state lives in a master table on the server, you can detach (`Ctrl-C`, `exit`) and reattach later.

## Prerequisites

1. **Directory object** — an OS path registered with the DB.
2. **Privileges** — `READ, WRITE ON DIRECTORY <name>`; `DATAPUMP_EXP_FULL_DATABASE` for anything other than owned schema.
3. **Space** — dump file target must have room; consider `%U` substitution for multi-file.

```sql
CREATE OR REPLACE DIRECTORY DP_DUMP AS '/u01/oracle/dumps';
GRANT READ, WRITE ON DIRECTORY DP_DUMP TO app_admin;
GRANT DATAPUMP_EXP_FULL_DATABASE TO app_admin;
```

## Basic Command Structure

```bash
expdp <user>/<pw>@<tns>       \
      DIRECTORY=DP_DUMP        \
      DUMPFILE=app_%U.dmp     \
      LOGFILE=app_expdp.log   \
      SCHEMAS=APP              \
      PARALLEL=4               \
      COMPRESSION=ALL          \
      FILESIZE=4G              \
      JOB_NAME=EXP_APP
```

Use a **parameter file** for anything complex:

```bash
expdp system/... parfile=exp_app.par
```

`exp_app.par`:

```
DIRECTORY=DP_DUMP
DUMPFILE=app_%U.dmp
LOGFILE=app_expdp.log
SCHEMAS=APP
PARALLEL=4
COMPRESSION=ALL
FILESIZE=4G
FLASHBACK_TIME=SYSTIMESTAMP
EXCLUDE=STATISTICS
JOB_NAME=EXP_APP
```

`%U` in `DUMPFILE` — Data Pump auto-numbers files (`_01`, `_02`, ...) up to `FILESIZE`.

## Key Parameters

| Parameter                                     | Purpose                                                                                |
| --------------------------------------------- | -------------------------------------------------------------------------------------- |
| `DIRECTORY`                                   | Oracle directory object.                                                               |
| `DUMPFILE`                                    | Dump file name(s); use `%U` for auto-numbered set.                                     |
| `LOGFILE`                                     | Log file name in the same directory.                                                   |
| `SCHEMAS` / `TABLES` / `FULL` / `TABLESPACES` | Mode selector.                                                                         |
| `PARALLEL`                                    | Number of workers; match dump file count for scaling.                                  |
| `FILESIZE`                                    | Max size per dump file (e.g. `4G`).                                                    |
| `COMPRESSION`                                 | `NONE`, `METADATA_ONLY`, `DATA_ONLY`, `ALL` (needs Advanced Compression license).      |
| `COMPRESSION_ALGORITHM`                       | `BASIC`, `LOW`, `MEDIUM`, `HIGH`.                                                      |
| `ENCRYPTION`                                  | Encrypt dumps (`NONE`, `DATA_ONLY`, `METADATA_ONLY`, `ALL`, `ENCRYPTED_COLUMNS_ONLY`). |
| `ENCRYPTION_PASSWORD`                         | Password for portable encryption without TDE.                                          |
| `FLASHBACK_TIME`                              | Read-consistent as-of this timestamp.                                                  |
| `FLASHBACK_SCN`                               | Read-consistent as-of this SCN.                                                        |
| `CONSISTENT`                                  | Legacy — use FLASHBACK_TIME/SCN.                                                       |
| `EXCLUDE` / `INCLUDE`                         | Object filters (`STATISTICS`, `GRANTS`, `TRIGGERS`, `INDEXES`, etc.).                  |
| `QUERY`                                       | Row-level filter per table (`SCHEMA.TABLE:"WHERE ..."`).                               |
| `NETWORK_LINK`                                | Pull from a remote DB — no local dump file.                                            |
| `SAMPLE`                                      | Percentage sample (e.g. `10`).                                                         |
| `ESTIMATE`                                    | `BLOCKS` (default, cheap) or `STATISTICS` (accurate, expensive).                       |
| `ESTIMATE_ONLY=YES`                           | Report size, don't export.                                                             |
| `JOB_NAME`                                    | Name the job — needed to reattach.                                                     |
| `REUSE_DUMPFILES=YES`                         | Overwrite existing dump files.                                                         |

## Common Patterns

### 1. Schema Export with Consistency

```bash
expdp system/... schemas=APP directory=DP_DUMP dumpfile=app_%U.dmp \
      parallel=4 filesize=8G flashback_time=SYSTIMESTAMP \
      exclude=statistics logfile=app_expdp.log job_name=EXP_APP
```

`FLASHBACK_TIME=SYSTIMESTAMP` guarantees the dump is consistent as of job start.

### 2. Table Subset with WHERE

```bash
expdp app/... tables=APP.ORDERS,APP.ORDER_LINES \
      query='APP.ORDERS:"WHERE order_dt >= DATE ''2026-01-01''"' \
      directory=DP_DUMP dumpfile=orders_2026.dmp
```

### 3. Metadata-Only DDL Extract

```bash
expdp system/... schemas=APP content=METADATA_ONLY \
      directory=DP_DUMP dumpfile=app_ddl.dmp
```

Follow up with `impdp SQLFILE=` to render pure DDL — see [IMPDP](impdp.md).

### 4. Full Database Export

```bash
expdp system/... full=YES directory=DP_DUMP dumpfile=full_%U.dmp \
      parallel=8 compression=ALL exclude=statistics \
      flashback_time=SYSTIMESTAMP job_name=EXP_FULL
```

### 5. Network Export from Standby

```bash
expdp system/...@target_db network_link=SRC_STBY schemas=APP \
      directory=DP_DUMP dumpfile=app_via_stby.dmp
```

### 6. Encrypted Export (no TDE)

```bash
expdp system/... schemas=APP encryption=ALL \
      encryption_password=<pw> encryption_algorithm=AES256 \
      directory=DP_DUMP dumpfile=app_enc.dmp
```

## Interactive Mode / Attach

Type `Ctrl-C` during a running client to enter interactive mode:

```
Export> status
Export> stop_job=immediate
Export> parallel=6
Export> exit_client
```

Reattach later:

```bash
expdp system/... attach=EXP_APP
Export> start_job
```

## Monitoring a Running Job

```sql
SELECT owner_name, job_name, operation, job_mode, state,
       degree, attached_sessions
FROM   dba_datapump_jobs;

SELECT   sid, serial#, sofar, totalwork,
         ROUND(sofar/totalwork*100,1) pct,
         time_remaining
FROM     v$session_longops
WHERE    opname LIKE 'DATAPUMP%'
ORDER BY start_time DESC;

-- Peek at log file
SELECT * FROM DBA_DATA_PUMP_ALL_JOBS ORDER BY start_time DESC;
```

## Estimating Size

```bash
expdp system/... schemas=APP estimate_only=YES estimate=BLOCKS
```

Reports approximate dump size before you commit disk. For accuracy on partitioned/LOB-heavy schemas use `estimate=STATISTICS` (requires fresh stats).

## Common Issues

- **`ORA-31693: Table data object ... failed to load/unload`** — Usually LOB corruption or bad block. Skip with `EXCLUDE=STATISTICS` won't help; use `TABLES=` to isolate and run block recovery on the offender.
- **`ORA-39001: invalid argument value`** — Syntax/directory issue. Check the exact position.
- **`ORA-39095: Dump file space has been exhausted`** — Set `FILESIZE=` and use `%U` in `DUMPFILE`.
- **`UDE-00008: subsequent worker unable to load metadata`** — Parallel workers hit a serialization point. Reduce `PARALLEL` or split by tablespace.
- **`ORA-01555 snapshot too old`** — Long-running export exhausted undo. Increase `UNDO_RETENTION` and undo tablespace, or use `FLASHBACK_SCN` from before the workload.
- **Extremely slow** — Not enough dump files for parallel workers, no compression, or LOBs (LOBs don't parallelize per row).

## Best Practices

1. Always specify `FLASHBACK_TIME=SYSTIMESTAMP` on schema/full exports for consistency.
2. Give the job a name (`JOB_NAME=`) so you can reattach.
3. Set `PARALLEL = N` and dump files = `N` — one file per worker (via `%U` and `FILESIZE`).
4. **Skip statistics** on export (`EXCLUDE=STATISTICS`); regather on target — much faster.
5. Use `COMPRESSION=ALL` if you have Advanced Compression license; typical 3–5× shrink.
6. Encrypt sensitive dumps (`ENCRYPTION_PASSWORD` or TDE) — treat as regulated data.
7. Use `NETWORK_LINK` for direct DB-to-DB when no landing storage is needed.
8. Never export into an FRA (backups compete). Use a dedicated `/u01/oracle/dumps`.
9. Compress files after export if not using Advanced Compression: `gzip *.dmp`.
10. Automate purge of old dumps — they'll fill the filesystem otherwise.

## Interview Questions

1. **Q:** What is the difference between old `exp` and `expdp`?
   **A:** `expdp` runs server-side, is parallel, restartable, network-capable, and uses a master table for state; `exp` is a serial client-side utility.

2. **Q:** What is the master table?
   **A:** A metadata table (`SYS_EXPORT_SCHEMA_01`, etc.) that stores job state so it can be paused/resumed and workers can coordinate.

3. **Q:** How do you make an export consistent?
   **A:** `FLASHBACK_TIME=SYSTIMESTAMP` or `FLASHBACK_SCN=<n>`.

4. **Q:** How do you export DDL only?
   **A:** `CONTENT=METADATA_ONLY` (or import with `SQLFILE=` to render DDL).

5. **Q:** How do you extract data from a remote DB without a dump file?
   **A:** `NETWORK_LINK=<db_link>` on `impdp`.

## References

- Oracle Database Utilities 19c — Data Pump Export
- MOS Doc ID 553337.1 — Data Pump Master Note
- MOS Doc ID 1077784.1 — expdp/impdp troubleshooting
