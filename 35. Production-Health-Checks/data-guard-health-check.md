# Data Guard Health Check

DR-readiness audit. Run against both primary and standby.

## 1. Roles and Modes

Primary:

```sql
SELECT database_role, protection_mode, protection_level,
       open_mode, switchover_status
FROM   v$database;
```

Standby:

```sql
SELECT database_role, open_mode, protection_mode, protection_level,
       flashback_on, force_logging
FROM   v$database;
```

Verify: primary is `PRIMARY`, standby is `PHYSICAL STANDBY`, protection_mode matches expected, flashback ON on both.

## 2. Broker Health

If broker enabled:

```bash
dgmgrl sys/pw@primary <<EOF
SHOW CONFIGURATION VERBOSE;
SHOW DATABASE VERBOSE 'primary_db';
SHOW DATABASE VERBOSE 'standby_db';
SHOW FAST_START FAILOVER;
EOF
```

Verify: SUCCESS status; no warnings; FSFO enabled if planned.

## 3. Apply / Transport Lag

```sql
SELECT name, value, unit, time_computed FROM v$dataguard_stats
ORDER BY name;
```

Verify: apply lag < SLA (typically < 60 s), transport lag negligible.

## 4. Archive Gap

Primary:

```sql
SELECT thread#, MAX(sequence#) primary_seq
FROM   v$archived_log GROUP BY thread#;

SELECT dest_id, dest_name, destination, status, error, log_sequence, target
FROM   v$archive_dest_status
WHERE  status NOT IN ('INACTIVE');
```

Standby:

```sql
SELECT * FROM v$archive_gap;

SELECT thread#,
       MAX(sequence#) shipped,
       MAX(CASE WHEN applied='YES' THEN sequence# END) applied
FROM   v$archived_log
GROUP  BY thread#;
```

Verify: no gaps; standby applied within a few sequences of primary.

## 5. Standby Redo Logs

```sql
SELECT group#, thread#, sequence#, bytes/1024/1024 mb, status
FROM   v$standby_log
ORDER  BY group#;
```

Verify:

- Standby redo logs exist on both primary and standby.
- Size ≥ largest online redo log.
- Count = threads × (max_groups_per_thread + 1).

## 6. Managed Standby Processes

Standby:

```sql
SELECT process, status, thread#, sequence#, block#, blocks, delay_mins
FROM   v$managed_standby
WHERE  process IN ('MRP0','RFS','LNS','NSSn','NSAn','PR%')
ORDER  BY process;
```

Verify:

- `MRP0` in APPLYING or WAITING state.
- `RFS` receiving redo (IDLE means recent activity).

## 7. Apply Parallelism

```sql
SELECT * FROM v$recovery_progress
ORDER BY start_time DESC
FETCH FIRST 10 ROWS ONLY;

SHOW PARAMETER parallel_max_servers
```

If broker-managed:

```
DGMGRL> SHOW DATABASE 'standby_db' 'ApplyParallel';
```

Verify: apply parallelism > 1 for busy primaries.

## 8. Log Transport Config

Primary:

```sql
SELECT dest_id, destination, target, delay_mins, async_blocks,
       archiver, valid_now, valid_type, valid_role, transmit_mode, affirm
FROM   v$archive_dest
WHERE  status <> 'INACTIVE';
```

Verify:

- Standby destination configured (LOG_ARCHIVE_DEST_2 typically).
- `transmit_mode` matches design (`SYNC` or `ASYNC`).
- `affirm` = YES for SYNC.

## 9. RTO / RPO Metrics

- **RTO** = time to failover. Measure via broker: `estimated startup time` + testing.
- **RPO** = transport lag (ASYNC) or 0 (SYNC).

```sql
SELECT name, value FROM v$dataguard_stats
WHERE  name IN ('apply lag','transport lag','estimated startup time','apply finish time');
```

## 10. Flashback and GRP

Both sides:

```sql
SELECT flashback_on, ROUND(SYSDATE-oldest_flashback_time,2) days_of_flashback
FROM   v$database, v$flashback_database_log;

SELECT name, guarantee_flashback_database, storage_size/1024/1024/1024 gb
FROM   v$restore_point;
```

Verify: flashback ON on both, retention supports rollback if switchover fails.

## 11. Log Files Present

Standby:

```sql
SELECT COUNT(*) FROM v$archived_log
WHERE  deleted='NO' AND applied='YES'
   AND completion_time > SYSDATE - 7;
```

Verify: applied archives retained for troubleshooting (matching broker retention).

## 12. Recent DG-Related Alerts

```sql
SELECT originating_timestamp, message_text
FROM   v$diag_alert_ext
WHERE  originating_timestamp > SYSDATE - 30
   AND (message_text LIKE '%Data Guard%' OR message_text LIKE '%MRP%'
        OR message_text LIKE '%RFS%' OR message_text LIKE '%LNS%'
        OR message_text LIKE '%standby%')
ORDER  BY originating_timestamp DESC
FETCH  FIRST 30 ROWS ONLY;
```

Verify: no errors; only routine messages.

## 13. Switchover Readiness Test

Non-destructive:

```
DGMGRL> VALIDATE DATABASE 'standby_db';
```

Verify: "Ready for Switchover: Yes" — nothing blocking.

## 14. Fast-Start Failover Config (if enabled)

```
DGMGRL> SHOW FAST_START FAILOVER;
```

Verify:

- Observer is registered and running.
- Threshold set (30 s minimum).
- Target is the intended standby.

## 15. Recent Role Transitions

```sql
SELECT name, value FROM v$dataguard_stats WHERE name LIKE '%transition%';
```

Any unexpected transitions in last 30 days?

## 16. Deliverable

Report structure:

- Health status (green/yellow/red).
- Apply/transport lag history (30-day chart).
- Broker configuration validation.
- Switchover test result.
- Gaps between planned and actual (RTO/RPO measured).

## Related

- [Data Guard](../17-data-guard/index.md).
- [Broker](../17-data-guard/broker.md).
- [FSFO](../17-data-guard/fsfo.md).
- [Data Guard Lag runbook](../27-runbooks/data-guard-lag.md).
