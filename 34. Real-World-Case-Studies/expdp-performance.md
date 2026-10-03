# Case: 12-Hour Dump → 3 Hour Dump

## Setup

- 4 TB schema export from 19c EE.
- Business need: exported nightly for downstream refresh.
- Current time: 12 hours (00:00 → 12:00). Missing SLA (must finish by 06:00).

## Baseline

Existing command:

```bash
expdp system/... schemas=APP directory=DP_DUMP dumpfile=app.dmp \
      logfile=app.log
```

Symptoms during run:

- Only 1 CPU core busy.
- Disk IO ~50 MB/s.
- Single dump file grows serially.

## Investigation

### Step 1 — Check Bottleneck

```bash
top -H -p $EXPDP_PID
# Only one thread active
```

```sql
SELECT s.sid, s.username, s.program, s.event, s.sql_id
FROM   v$session s
WHERE  s.module LIKE 'Data Pump%';
```

Result:

```
SID   USERNAME   PROGRAM       EVENT
340   SYSTEM     DM00 (0)     wait for unread message on broadcast channel
420   SYSTEM     DW00 (0)     direct path write
```

Only 1 worker (`DW00`). PARALLEL wasn't set.

### Step 2 — Look at Log

```
Master table "SYSTEM"."SYS_EXPORT_SCHEMA_01" successfully loaded/unloaded
Starting "SYSTEM"."SYS_EXPORT_SCHEMA_01":  system/... schemas=APP ...
Estimate in progress using BLOCKS method...
Processing object type SCHEMA_EXPORT/USER
...
. . exported "APP"."ORDERS"                     640.5 GB   4,200,000,000 rows
. . exported "APP"."ORDER_LINES"                890.2 GB   12,000,000,000 rows
. . exported "APP"."CUSTOMERS"                  45.8 GB    50,000,000 rows
```

Massive tables. All going through 1 worker → 1 dump file → 1 disk stream.

## Fix Round 1 — Parallelize

```bash
expdp system/... schemas=APP \
      directory=DP_DUMP \
      dumpfile=app_%U.dmp \
      parallel=8 \
      filesize=8G \
      logfile=app.log \
      job_name=EXP_APP
```

Key changes:

- `parallel=8` — 8 workers.
- `dumpfile=app_%U.dmp` — pattern for multiple files.
- `filesize=8G` — cap per file, keeps parallel workers unblocked.

Re-run: **7 hours**. Better but still missing SLA.

## Investigation Round 2

`top` now shows 8 threads all busy. Disk IO ~350 MB/s. Bottleneck moved to storage.

Look at wait events:

```sql
SELECT event, COUNT(*) FROM v$session
WHERE  module LIKE 'Data Pump%' GROUP BY event;
```

```
EVENT                       COUNT
direct path write             5
db file sequential read       2
control file sequential read  1
```

Reads AND writes — the workers are reading source data as they write to dump. `direct path write` = writing to dump file.

Check storage:

```bash
iostat -x 1 10
```

```
Device   %util   await(ms)
xvdf     98%     22
xvdg     94%     18
```

98% utilized. Storage is the bottleneck.

## Fix Round 2 — Compression + Filesystem Distribution

Add compression to reduce writes:

```bash
expdp system/... schemas=APP \
      directory=DP_DUMP \
      dumpfile=app_%U.dmp \
      parallel=8 filesize=8G \
      compression=ALL \
      compression_algorithm=MEDIUM \
      logfile=app.log \
      job_name=EXP_APP
```

Compression `MEDIUM` gives ~4× shrink. So writes drop from 350 MB/s to ~90 MB/s. And CPU picks up the slack.

Re-run: **4.5 hours**. Better still.

## Fix Round 3 — Exclude Stats

The last hour of the run was gathering statistics on the exported table structures. Not needed — target will re-gather.

```bash
expdp ... exclude=STATISTICS ...
```

Also add:

```bash
exclude=DATABASE_LINK
```

DB links contain passwords; you don't want them in the dump anyway.

Re-run: **3.5 hours**.

## Fix Round 4 — Split Extra-Large Tables

`ORDER_LINES` alone was 890 GB. Its export was single-worker due to LOB serialization (some CLOB columns).

**Solution**: partition-per-worker.

`ORDER_LINES` is partitioned by month. Export per-partition:

```bash
expdp system/... \
      tables=APP.ORDER_LINES:P_2026_08,APP.ORDER_LINES:P_2026_07,... \
      parallel=8 filesize=8G \
      directory=DP_DUMP \
      dumpfile=order_lines_%U.dmp \
      compression=ALL
```

Now each partition is exported as its own unit, so 8 partitions can be worked on simultaneously.

Combined with the earlier optimizations, **total time: 2 hours 55 minutes**.

## Final Command

```bash
expdp system/... \
      schemas=APP \
      directory=DP_DUMP \
      dumpfile=app_%U.dmp \
      parallel=16 \
      filesize=8G \
      compression=ALL \
      compression_algorithm=MEDIUM \
      exclude=STATISTICS,DATABASE_LINK \
      flashback_time=SYSTIMESTAMP \
      logfile=app.log \
      job_name=EXP_APP
```

For sub-hour on 4 TB, you'd need to shard across multiple hosts / storage arrays.

## Lessons Learned

- **PARALLEL matched to dump file count** is the single biggest lever.
- **COMPRESSION** shifts bottleneck from IO to CPU; on modern CPU-rich boxes, always a win.
- **Skip statistics** — regather on target.
- **Partitioned tables** are ideal for parallel-per-partition export.
- **LOB columns don't parallelize inside a partition** — split at partition boundary.
- Always compare `iostat` before deciding it's a compute problem.

## Related

- [Data Pump Performance Tuning](../21-data-pump/performance-tuning.md).
- [EXPDP](../21-data-pump/expdp.md).
- [Migration Methods](../23-upgrade-migration/migration-methods.md).
