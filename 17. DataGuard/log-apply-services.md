# Log Apply Services

## Overview

**Log Apply Services** is the standby-side machinery that consumes redo from the Standby Redo Logs and applies it to standby datafiles. For a **physical standby**, this is the **Managed Recovery Process (MRP0)**. For a **logical standby**, it's the Logical Standby Process (LSP0) — mostly deprecated in favor of GoldenGate.

## Physical Standby Apply — MRP0

MRP0 reads redo from SRLs (or archive logs if there's a gap) and applies it as if the standby were a running instance in recovery.

Modes:

- **Managed Recovery** — MRP0 automatically consumes redo as it arrives.
- **Real-Time Apply** — MRP0 reads directly from Standby Redo Logs as they're written (rather than waiting for a log switch).
- **Cascaded Apply** — Redo forwarded from one standby to another.

Real-time apply is essential for **Active Data Guard** and low-lag failover.

## Enabling Real-Time Apply

```sql
-- On standby
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

`USING CURRENT LOGFILE` = read from SRLs directly = real-time apply.

Broker equivalent: real-time apply is the default when SRLs exist.

## Stopping Apply

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
```

## Recovery Slaves

Apply is parallelized via **recovery slaves** (`PR0N`), spawned by MRP0:

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE PARALLEL 16 USING CURRENT LOGFILE DISCONNECT;
```

Default is CPU count. Higher parallelism helps on busy DBs.

## Delay

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DELAY 60;
```

Applies redo N minutes behind primary. Provides a "safety window" against user error (bad batch on primary can be flashed away from standby before applied). Note: SRLs still receive redo in real time — the delay is only in apply.

## Cascaded Standbys

A cascaded standby receives redo from another standby, not the primary. Reduces primary → WAN traffic if you have multiple downstream standbys in different regions.

```sql
-- On the intermediate standby
ALTER DATABASE SET LOG_ARCHIVE_DEST_3=
  'SERVICE=downstream_standby ASYNC ...';
```

## Diagnostic Queries

```sql
-- Apply state
SELECT process, status, sequence#, block#, delay_mins
FROM   v$managed_standby
ORDER  BY process;

-- Or via V$DATAGUARD_PROCESS (12c+)
SELECT name, pid, role, action, client_role, sequence#
FROM   v$dataguard_process
WHERE  role IN ('log apply','media recovery');

-- Apply lag
SELECT name, value, unit FROM v$dataguard_stats
WHERE  name = 'apply lag';

-- Recovery progress
SELECT * FROM v$recovery_progress;

-- Which log is being applied?
SELECT thread#, sequence#, first_time, applied
FROM   v$archived_log
WHERE  applied IN ('YES','IN-MEMORY')
ORDER  BY first_time DESC
FETCH FIRST 20 ROWS ONLY;
```

## Common Issues

- **Apply stuck** — `V$MANAGED_STANDBY.STATUS = 'WAITING'` while redo waiting. Check RFS, network, and SRLs.
- **`ORA-16145: archival for log sequence N not yet completed`** — Primary not yet done with a log. Wait.
- **`ORA-16016: archived log for thread N sequence M unavailable`** — Gap. Use FAL to fetch or manually copy.
- **Slow apply** — Increase parallelism. Check standby I/O.
- **Apply lag growing during heavy DML** — Standby storage too slow, or apply parallelism insufficient.

## Best Practices

1. **Real-Time Apply always** — `USING CURRENT LOGFILE`.
2. **SRLs sized to match primary** — one more group than primary, same size.
3. Standby storage as fast as primary (or faster for apply).
4. Set apply parallelism to CPU count or higher.
5. Delayed apply for user-error protection when RPO permits.
6. Monitor `V$DATAGUARD_STATS.apply lag` — alert > 60 seconds.
7. Cascaded standbys for multi-region topologies.
8. Match Oracle versions exactly between primary and standby.
9. Alert on MRP0 not running.
10. Reduce primary NOLOGGING operations (or set FORCE LOGGING).

## Interview Questions

1. **Q:** What is MRP0?
   **A:** Managed Recovery Process — reads redo on standby and applies to datafiles.

2. **Q:** Real-Time Apply?
   **A:** MRP0 reads SRLs as they're written, not waiting for log switch. Enabled by `USING CURRENT LOGFILE`.

3. **Q:** Apply delay use case?
   **A:** Protect against user error — bad batch on primary can be flashed away from standby before applied.

4. **Q:** Cascaded standby?
   **A:** Downstream standby receives redo from another standby, reducing primary WAN load.

5. **Q:** How to see apply state?
   **A:** `V$MANAGED_STANDBY`, `V$DATAGUARD_STATS`, `V$DATAGUARD_PROCESS`.

## References

- Oracle Data Guard Concepts and Administration 19c
- MOS Doc ID 220970.1 — Log Apply Troubleshooting
- MOS Doc ID 1550116.1 — Redo Apply Performance
