# Data Pump Performance Tuning

## Overview

Data Pump can be surprisingly slow out-of-the-box because defaults are conservative and every workload has a different bottleneck (CPU, network, IO, undo, LOB serialization). Tuning is about matching the number of **workers**, the number of **dump files**, and the shape of the **objects** to the resources of the source and target.

The mental model to keep: `expdp/impdp` = **parallel producer/consumer**. Producers (workers) can only go as fast as the slowest of {CPU, IO, network}. If one is saturated, adding workers won't help.

## Baseline: Measure Before Tuning

Run one full export/import and capture:

- Total elapsed time.
- Rows and bytes moved (`. . imported/exported` lines in log).
- Per-worker CPU (`v$session` → OS `top -H`).
- Waits (`v$session_wait`, `v$system_event` snapshot before/after).
- Filesystem throughput (`iostat -x 5`).

Then change one knob at a time.

## Knob 1 — `PARALLEL`

**Rule of thumb**: `PARALLEL = min(CPU_COUNT, dump_file_count)`. Setting `PARALLEL=8` with a single dump file gives you 1 worker, not 8 — because they'd serialize on file writes.

```bash
# Correct pairing
expdp ... PARALLEL=8 DUMPFILE=exp_%U.dmp FILESIZE=8G
```

At import, `PARALLEL` should also match the number of files you have.

**Ceiling**: on the source, keep total workers ≤ `CPU_COUNT * 2`; the box still needs to run its normal workload.

**LOB caveat**: LOBs don't parallelize inside a partition. If a schema is 90% one giant LOB table, adding workers barely helps.

## Knob 2 — `COMPRESSION`

Compression trades **CPU time** for **IO / space**.

| Setting                             | When to use                                    |
| ----------------------------------- | ---------------------------------------------- |
| `NONE`                              | Fast CPU-poor box, plenty of disk.             |
| `METADATA_ONLY`                     | Almost free, always helpful.                   |
| `DATA_ONLY` (Advanced Comp license) | Slow disk, capable CPU.                        |
| `ALL`                               | WAN transfer, tight disk. Typical 3–5× shrink. |

Choose `COMPRESSION_ALGORITHM=MEDIUM` — best ratio/CPU balance. `HIGH` is CPU-brutal, `LOW` gives back most of the shrink.

## Knob 3 — Skip What You Can Rebuild

Every second Data Pump spends replaying **indexes**, **statistics**, and **grants** during a load is a second the tables spend unusable. Skip and rebuild:

```bash
# Export
expdp ... EXCLUDE=STATISTICS

# Import
impdp ... EXCLUDE=STATISTICS,INDEX \
         DATA_OPTIONS=DISABLE_APPEND_HINT
```

Post-import:

```sql
-- Rebuild indexes from a pre-generated DDL file
@app_indexes.sql

-- Gather stats
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS('APP', DEGREE => 16);
END;
/
```

Empirically 3–5× faster for schemas with heavy secondary indexing.

## Knob 4 — `DISABLE_ARCHIVE_LOGGING` (19c)

Skips redo generation on `INSERT /*+ APPEND */` during import. Requires:

- Database not in `FORCE LOGGING`.
- You're OK re-doing the import if the target loses power mid-load.
- Standby databases WILL diverge — take fresh incremental after.

```bash
impdp ... TRANSFORM=DISABLE_ARCHIVE_LOGGING:Y
```

Savings: 20–40% for insert-heavy loads.

## Knob 5 — Network Mode Tuning

`NETWORK_LINK` skips the dump-file step but adds network round-trip cost. Tune:

- Set `SDU_SIZE=32767` on both `sqlnet.ora` files.
- Use a database link routing over the fastest path (private VLAN, direct connect).
- Set `NETWORK_LINK` + `PARALLEL=N` at import; each worker opens its own connection.
- `ENCRYPTION` on the DB link (via wallet) is essentially free on modern CPUs.

Network mode is IO-bound for the pipe, not disk. Measure with `iftop`.

## Knob 6 — Undo, Temp, and Redo Sizing

Import is undo- and temp-heavy:

- Undo tablespace: sized to hold the biggest concurrent transaction. LOBs blow this up; consider `UNDO_RETENTION` bump.
- TEMP: needs headroom for index rebuild (`ORA-01652`).
- Redo: NOT skipped without `DISABLE_ARCHIVE_LOGGING`. On archivelog databases, ensure archiver isn't falling behind (`v$archive_dest_status.error`).

Query these before starting a large import:

```sql
-- Undo headroom
SELECT ROUND(SUM(bytes)/1024/1024/1024,2) undo_gb
FROM   dba_data_files WHERE tablespace_name = (SELECT value
    FROM v$parameter WHERE name='undo_tablespace');

-- Temp headroom
SELECT ROUND(SUM(bytes)/1024/1024/1024,2) temp_gb
FROM   dba_temp_files;

-- Redo generation rate right now (MB/hr)
SELECT ROUND(value/1024/1024,2) mb_since_startup
FROM   v$sysstat WHERE name='redo size';
```

## Knob 7 — Direct Path vs External Table Mode

Data Pump picks between two access methods per table:

- **Direct Path** — fastest; used when possible.
- **External Table** — used when object features (fine-grained access, some triggers, certain columns) prevent direct path.

Force one:

```bash
expdp ... ACCESS_METHOD=DIRECT_PATH
expdp ... ACCESS_METHOD=EXTERNAL_TABLE
```

Check the log — Data Pump prints `Estimate in progress using DIRECT_PATH method...` for each table. If you see `EXTERNAL_TABLE` on tables you expected direct path, investigate why (often: fine-grained access policies, IOTs with mapping tables, tables with LONG columns).

## Knob 8 — Filesystem Choice

- Local NVMe/SSD → single worker often saturates it; parallelize IO, not workers.
- NFS → set `rsize=1048576, wsize=1048576, hard, nointr, tcp, actimeo=0`. Avoid over-parallelizing (packet drops).
- Object storage FUSE → often the bottleneck; stream via `NETWORK_LINK` instead.
- FRA → NEVER; RMAN backups compete.

## Diagnostic Queries During a Job

```sql
-- Per-worker CPU / IO / wait
SELECT s.sid, s.serial#, s.username, s.program,
       s.status, s.sql_id, s.event, s.wait_class,
       s.seconds_in_wait, s.p1, s.p2
FROM   v$session s
WHERE  s.module LIKE 'Data Pump%'
ORDER  BY s.sid;

-- Long-op progress
SELECT sid, opname, target, sofar, totalwork,
       ROUND(sofar/totalwork*100,1) pct,
       time_remaining
FROM   v$session_longops
WHERE  opname LIKE 'DATAPUMP%';

-- Master coordinator bottleneck check
SELECT event, total_waits, time_waited_micro/1e6 AS seconds
FROM   v$system_event
WHERE  event LIKE 'Datapump%'
ORDER  BY time_waited_micro DESC
FETCH FIRST 20 ROWS ONLY;
```

## Common Bottlenecks and Fixes

| Symptom                           | Likely Cause                       | Fix                                             |
| --------------------------------- | ---------------------------------- | ----------------------------------------------- |
| Low CPU, low IO                   | Too few workers or serial LOB      | Add PARALLEL / split LOB tables                 |
| High CPU, low IO                  | Compression = HIGH, single core    | Drop to MEDIUM, add PARALLEL                    |
| High IO wait                      | Dump file storage slow             | Move to faster FS; drop compression to NONE     |
| Waits on `resmgr:cpu quantum`     | Resource plan capping              | Move Data Pump to a group with higher `mgmt_p1` |
| `direct path write` wait dominant | Storage saturated                  | Fewer workers or faster storage                 |
| `library cache lock` on import    | DDL contention (small many tables) | Split by object type; run in chunks             |
| `log file sync` on import         | Redo behind                        | `DISABLE_ARCHIVE_LOGGING`, size redo bigger     |
| Slow LOB export                   | Serial per partition               | Partition LOB tables; export per partition      |

## A Reference Fast-Import Recipe

For a 500 GB schema migration into a fresh target:

```bash
# 1. Extract DDL only (for indexes/constraints)
impdp system/... schemas=APP directory=DP_DUMP dumpfile=app_%U.dmp \
      sqlfile=app_full_ddl.sql

# 2. Split out just the index DDL
grep -A200 'CREATE INDEX\|ALTER INDEX' app_full_ddl.sql > app_index_ddl.sql

# 3. Fast bulk import
impdp system/... schemas=APP directory=DP_DUMP dumpfile=app_%U.dmp \
      parallel=16 exclude=STATISTICS,INDEX,STATISTIC_TABLE \
      transform=DISABLE_ARCHIVE_LOGGING:Y \
      logfile=app_impdp.log job_name=IMP_APP_BULK

# 4. Rebuild indexes in parallel
sqlplus system/... @app_index_ddl.sql

# 5. Gather stats
sqlplus system/... <<EOF
BEGIN
  DBMS_STATS.GATHER_SCHEMA_STATS('APP', DEGREE=>16);
END;
/
EOF

# 6. Flip back to FORCE LOGGING if needed
ALTER DATABASE FORCE LOGGING;
```

Typical speedup: 3–5× over a naive `impdp schemas=APP parallel=4`.

## Interview Questions

1. **Q:** You set PARALLEL=8 but see one CPU busy. Why?
   **A:** Single dump file — workers serialize on writes. Add `%U` and `FILESIZE`.

2. **Q:** Import is redo-bound. What are your options?
   **A:** `TRANSFORM=DISABLE_ARCHIVE_LOGGING:Y` (19c, non-FORCE-LOGGING), grow redo logs, split load into smaller chunks with commits between.

3. **Q:** How do you speed up a LOB-heavy schema?
   **A:** Partition LOB tables and export per partition; LOBs don't parallelize inside a partition.

4. **Q:** Why is compression sometimes slower?
   **A:** CPU becomes the bottleneck; `HIGH` compression is CPU-brutal. Use `MEDIUM` or drop to `NONE` on fast disk.

5. **Q:** Which access method is faster and when isn't it available?
   **A:** Direct Path is faster; External Table is used when the table has features (LONG columns, VPD, some triggers) that block direct path.

## References

- Oracle Database Utilities 19c — Data Pump Performance
- MOS Doc ID 552424.1 — Export/Import DataPump: Slow Performance
- MOS Doc ID 1327351.1 — Data Pump Parallel Best Practices
