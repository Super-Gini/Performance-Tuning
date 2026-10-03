# Monitoring

## Overview

Monitoring Data Guard means answering: is redo flowing? Is apply keeping up? Is protection mode intact? Are Standby Redo Logs healthy? Is FSFO Observer alive?

This page consolidates the metrics, views, and commands you should have on your dashboard.

## Broker Health

Fastest overall check:

```
$ dgmgrl sys/pwd@prod

DGMGRL> SHOW CONFIGURATION;
```

Output includes each database's role, transport, apply state, and any warnings/errors. Anything other than "SUCCESS" needs investigation.

```
DGMGRL> SHOW DATABASE VERBOSE 'prod_dr';
```

## Lag Metrics — `V$DATAGUARD_STATS`

```sql
SELECT name, value, unit
FROM   v$dataguard_stats
WHERE  name IN ('transport lag','apply lag','estimated startup time');
```

- **`transport lag`** — time between redo generated on primary and received on standby.
- **`apply lag`** — time between redo received on standby and applied to standby datafiles.

Sum ≈ end-to-end lag. Alert on either > 60 seconds.

## Archive Gap

```sql
-- On standby
SELECT * FROM v$archive_gap;
```

Empty result = no gap. Rows = gaps. FAL (Fetch Archive Log) automatically retrieves gaps if configured:

```sql
ALTER SYSTEM SET FAL_SERVER='prod';
ALTER SYSTEM SET FAL_CLIENT='prod_dr';
```

## MRP / Apply State

```sql
SELECT process, status, sequence#, block#, delay_mins
FROM   v$managed_standby
ORDER  BY process;
```

Look for:

- `MRP0` — should be `APPLYING_LOG` or `WAIT_FOR_LOG`.
- Any process in `ERROR` state.

## Transport State

```sql
-- On primary
SELECT dest_id, dest_name, status, target, error,
       gap_status, log_sequence, applied_seq
FROM   v$archive_dest
WHERE  status <> 'INACTIVE';
```

`STATUS = VALID` — good. Anything else — problem.

## Standby Redo Logs

```sql
-- On standby
SELECT group#, thread#, sequence#, bytes/1024/1024 AS mb, status
FROM   v$standby_log
ORDER  BY group#;
```

Rotation and `ACTIVE` status normal during apply. If all groups perpetually ACTIVE with no `UNASSIGNED`, add more SRL groups.

## Protection Mode

```sql
-- On both
SELECT protection_mode, protection_level FROM v$database;
```

`protection_level` may drop below `protection_mode` if standby unreachable — indicating a temporary downgrade.

## FSFO Observer

```sql
SELECT fs_failover_status, fs_failover_observer_present,
       fs_failover_current_target
FROM   v$database;
```

- `fs_failover_observer_present = YES` — good.
- `fs_failover_status = SYNCHRONIZED` — good.

## Log-Level Monitoring

```sql
-- V$DATAGUARD_STATUS shows recent DG messages
SELECT severity, message_num, message
FROM   v$dataguard_status
WHERE  severity IN ('Warning','Error','Fatal')
ORDER  BY timestamp DESC
FETCH FIRST 20 ROWS ONLY;
```

## Alerting Recipes

Set alerts on:

- **Broker health** — `dgmgrl show configuration` output includes anything other than SUCCESS.
- **Apply lag > 60 sec** — `V$DATAGUARD_STATS`.
- **Transport lag > 60 sec** — same view.
- **Archive gap** — `V$ARCHIVE_GAP` returns rows.
- **MRP0 not running** — `V$MANAGED_STANDBY` missing MRP0.
- **FSFO observer down** — `V$DATABASE.FS_FAILOVER_OBSERVER_PRESENT = NO`.
- **Protection downgrade** — `PROTECTION_LEVEL` < `PROTECTION_MODE`.

## Dashboard Query

Combined:

```sql
SELECT (SELECT db_unique_name FROM v$database) AS db,
       (SELECT open_mode FROM v$database) AS open_mode,
       (SELECT database_role FROM v$database) AS role,
       (SELECT value FROM v$dataguard_stats WHERE name='transport lag') AS transport_lag,
       (SELECT value FROM v$dataguard_stats WHERE name='apply lag') AS apply_lag,
       (SELECT protection_mode FROM v$database) AS protection_mode,
       (SELECT protection_level FROM v$database) AS protection_level,
       (SELECT fs_failover_observer_present FROM v$database) AS fsfo_observer
FROM dual;
```

## Common Issues

- **Apply lag rising but transport OK** — Standby apply slow. Check apply parallelism, standby I/O.
- **Transport lag rising** — Network problem, or primary generating faster than link bandwidth.
- **Gaps persistent** — FAL isn't configured, or primary can't reach standby's FAL_CLIENT.
- **Observer unreachable** — Firewall, host down, or `dgmgrl start observer` not persistent.

## Best Practices

1. **Broker `SHOW CONFIGURATION`** in cron every 5 min; alert on non-SUCCESS.
2. **Lag alerts** at 60s and 300s.
3. **Gap alerts** immediately.
4. **Observer** as systemd / auto-restart service.
5. Log `dgmgrl show configuration verbose` output daily for audit.
6. **DR test drill** quarterly (switchover; annual failover).
7. Store observer log with retention ≥ 90 days.
8. Alert on `dg_broker_start` becoming FALSE.
9. Verify Data Guard is actually protecting: match `V$LOG_HISTORY` on primary with `V$LOG_HISTORY` on standby every hour.
10. Compare RMAN backup times on primary vs standby (if standby-side backups).

## Interview Questions

1. **Q:** How do you check DG lag?
   **A:** `V$DATAGUARD_STATS` — transport lag and apply lag values.

2. **Q:** Archive gap check?
   **A:** `SELECT * FROM v$archive_gap;` on standby.

3. **Q:** Overall broker health?
   **A:** `dgmgrl SHOW CONFIGURATION`.

4. **Q:** MRP0 state?
   **A:** `V$MANAGED_STANDBY.STATUS`.

5. **Q:** FSFO monitoring?
   **A:** `V$DATABASE.FS_FAILOVER_*` columns.

6. **Q:** What if protection level < protection mode?
   **A:** Temporary degradation — standby unreachable. Investigate transport.

## References

- Oracle Data Guard Concepts and Administration 19c — Monitoring
- MOS Doc ID 220970.1 — DG Monitoring
- MOS Doc ID 1550116.1 — DG Health Checks
