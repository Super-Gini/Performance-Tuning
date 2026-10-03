# Case: Standby Fell 30 Minutes Behind

## Setup

- Primary on-prem 19c EE, standby in DR region.
- ASYNC redo transport, real-time apply.
- Normal apply lag < 5 seconds.
- 09:15 UTC: monitoring alert — apply lag 30 minutes.

## Investigation

### Step 1 — Confirm on Primary

```sql
-- Primary
SELECT name, value, time_computed
FROM   v$dataguard_stats
WHERE  name IN ('apply lag','transport lag');
```

```
NAME             VALUE
apply lag        +00 00:30:14
transport lag    +00 00:00:12
```

Transport lag ~12 s (normal). Apply lag 30 min — problem is on standby.

### Step 2 — Standby State

```sql
-- Standby
SELECT database_role, open_mode FROM v$database;
-- PHYSICAL STANDBY / READ ONLY WITH APPLY

SELECT process, status, thread#, sequence#, block#, blocks, delay_mins
FROM   v$managed_standby
WHERE  process IN ('MRP0','RFS');
```

Result:

```
PROCESS  STATUS      THREAD#  SEQUENCE#  BLOCK#
MRP0     APPLYING    1        45892      120000
RFS      IDLE        1        45895
```

MRP0 is still applying (not stuck). Just slow — it's on sequence 45892 while RFS has received up to 45895.

### Step 3 — Why Slow?

```sql
-- Standby
SELECT type, item, ROUND(sofar,2) sofar, ROUND(total,2) total,
       start_time, timestamp,
       ROUND((sofar/GREATEST(total,1))*100,1) pct
FROM   v$recovery_progress
ORDER  BY start_time DESC
FETCH  FIRST 10 ROWS ONLY;
```

Result:

```
TYPE                    ITEM                            SOFAR    TOTAL   PCT
Media Recovery          Log Files                       3        5       60
Media Recovery          Active Apply Rate KB/sec         200
Media Recovery          Average Apply Rate KB/sec       220
Media Recovery          Redo Applied MB                  8500
Media Recovery          Log Files                        2        5       40
```

Apply rate ~200 KB/s. Normal is 5–10 MB/s. Something making apply slow.

### Step 4 — MRP0 Wait Events

```sql
SELECT p.pname, s.event, s.wait_class, s.p1, s.p2, s.p3, s.seconds_in_wait
FROM   v$session s JOIN v$process p ON p.addr = s.paddr
WHERE  p.pname LIKE 'MRP%' OR p.pname LIKE 'PR%';
```

Result:

```
PNAME    EVENT                     WAIT_CLASS      P1 (file#)   BLOCK#
MRP0     db file sequential read   User I/O        42           15234
PR00     db file sequential read   User I/O        42           15238
PR01     db file sequential read   User I/O        42           15242
```

Read IO — MRP0 and its parallel recovery slaves are reading blocks. Which file?

```sql
SELECT tablespace_name, file_name FROM dba_data_files WHERE file_id = 42;
```

= `USERS_DATA_LARGE`.

### Step 5 — What Is In That File

```sql
SELECT segment_name, segment_type,
       ROUND(bytes/1024/1024/1024, 2) gb
FROM   dba_segments
WHERE  tablespace_name = 'USERS_DATA_LARGE'
ORDER  BY bytes DESC;
```

```
SEGMENT_NAME       TYPE     GB
DAILY_METRICS      TABLE    120
DAILY_METRICS_PK   INDEX     18
```

`DAILY_METRICS` — a fact table. Big rebuild happened?

### Step 6 — Primary Alert Log Around 08:30

On primary:

```bash
grep -E "REBUILD|MOVE|create|CREATE" \
   $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log | tail -50
```

```
2026-08-06T08:32:15 CREATE INDEX daily_metrics_status_idx ON daily_metrics(status) TABLESPACE users_data_large PARALLEL 8
```

An index build on a 120 GB table with `PARALLEL 8`. On primary, took ~15 min with parallel workers.

On the **standby**, MRP0 is single-threaded (by default). It's replaying 15 min of parallel work as a serial stream of block writes — much slower.

### Root Cause

Large parallel operation on primary generated a redo blob standby MRP0 can't apply at the same rate. Result: apply lag grows during the operation and takes time to catch up.

## Fix — Increase Standby Apply Parallelism

```sql
-- Standby
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

ALTER DATABASE RECOVER MANAGED STANDBY DATABASE
    USING CURRENT LOGFILE
    PARALLEL 8
    DISCONNECT FROM SESSION;
```

Or via broker:

```
DGMGRL> EDIT DATABASE 'stby' SET STATE='APPLY-OFF';
DGMGRL> EDIT DATABASE 'stby' SET PROPERTY 'ApplyParallel'=8;
DGMGRL> EDIT DATABASE 'stby' SET STATE='APPLY-ON';
```

Watch progress:

```sql
SELECT process, status, sequence#, block#
FROM   v$managed_standby WHERE process LIKE 'PR%' OR process = 'MRP0';
```

Should see MRP0 + 8 PRnn slaves. Apply rate jumps to 5+ MB/s.

## Verify

```sql
SELECT name, value FROM v$dataguard_stats WHERE name = 'apply lag';
```

Lag closes over ~10 min.

## Long-Term Fix

- **Set `ApplyParallel`** on the broker configuration permanently.
- **Coordinate big ops** with off-peak / planned maintenance windows.
- Consider **Standby-First patching / operations** so heavy DML doesn't disrupt DR.
- Monitor for **apply lag > 5 min** — critical alarm.

## Lessons Learned

- Single-threaded MRP0 is a bottleneck for parallel primary operations.
- Standby needs sizing = "primary redo generation rate × safety factor".
- `PARALLEL N` on standby apply is the fastest single lever.
- Consider **read replica** for reports so apply-parallelism can be higher without impacting reporting.

## Related

- [Data Guard Architecture](../17-data-guard/architecture.md).
- [Log Apply Services](../17-data-guard/log-apply-services.md).
- [Data Guard Lag runbook](../27-runbooks/data-guard-lag.md).
