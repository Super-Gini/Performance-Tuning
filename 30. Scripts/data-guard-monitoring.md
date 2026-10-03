# Data Guard Monitoring Scripts

## Overall Health (One-Shot)

```sql
-- Role, mode
SELECT database_role, protection_mode, protection_level, open_mode,
       switchover_status FROM v$database;

-- Broker DG stats
SELECT name, value, unit, time_computed
FROM   v$dataguard_stats;
```

## Apply Lag

```sql
SELECT name, value FROM v$dataguard_stats WHERE name IN ('apply lag','transport lag');
```

## Latest Sequences — Primary vs Standby

```sql
-- On primary
SELECT thread#, MAX(sequence#) FROM v$archived_log GROUP BY thread#;

-- On standby
SELECT thread#,
       MAX(sequence#) shipped,
       MAX(CASE WHEN applied='YES' THEN sequence# END) applied,
       MAX(sequence#) - MAX(CASE WHEN applied='YES' THEN sequence# END) gap
FROM   v$archived_log
GROUP  BY thread#;
```

## Archive Gap

```sql
-- On standby
SELECT * FROM v$archive_gap;

-- Detailed: what should apply but hasn't
SELECT thread#, sequence#, applied
FROM   v$archived_log
WHERE  applied = 'NO'
ORDER  BY thread#, sequence#;
```

## Standby Processes (RFS, MRP)

```sql
-- On standby
SELECT process, status, thread#, sequence#, block#, blocks, delay_mins
FROM   v$managed_standby
WHERE  process IN ('MRP0','RFS','ARCH','LNS','NSA0','NSS0')
ORDER  BY process;
```

## Archive Destination Health (Primary)

```sql
SELECT dest_id, dest_name, destination, status, error,
       log_sequence, target
FROM   v$archive_dest_status
WHERE  status <> 'INACTIVE'
ORDER  BY dest_id;

-- Any destination in ERROR?
SELECT dest_id, status, error FROM v$archive_dest_status WHERE error IS NOT NULL;
```

## Redo Transport Rate

```sql
SELECT   thread#, sequence#,
         first_change#, next_change#,
         first_time, completion_time,
         ROUND((completion_time - first_time)*24*60, 1) mins,
         ROUND(blocks*block_size/1024/1024, 2) mb
FROM     v$archived_log
WHERE    completion_time > SYSDATE - 1/24
ORDER BY completion_time DESC
FETCH FIRST 15 ROWS ONLY;
```

## Broker Configuration (DGMGRL)

```bash
dgmgrl sys/pw
DGMGRL> SHOW CONFIGURATION VERBOSE;
DGMGRL> SHOW DATABASE VERBOSE 'primary_db';
DGMGRL> SHOW DATABASE VERBOSE 'standby_db';
```

## Recover Progress (Standby)

```sql
SELECT * FROM v$recovery_progress
ORDER  BY start_time DESC
FETCH  FIRST 20 ROWS ONLY;
```

## Historical Lag

```sql
SELECT ROUND(EXTRACT(DAY FROM value)*24*60 + EXTRACT(HOUR FROM value)*60 +
             EXTRACT(MINUTE FROM value), 2) apply_lag_min,
       TO_CHAR(time_computed,'YYYY-MM-DD HH24:MI') t
FROM   dba_hist_dataguard_stats_bh
WHERE  name = 'apply lag' AND time_computed > SYSDATE - 7
ORDER  BY time_computed DESC
FETCH  FIRST 100 ROWS ONLY;
```

## Related

- [Data Guard Architecture](../17-data-guard/architecture.md).
- [Log Apply Services](../17-data-guard/log-apply-services.md).
- [Data Guard Lag runbook](../27-runbooks/data-guard-lag.md).
- [V$DATAGUARD_STATS](../25-reference/v-views/v-dataguard-stats.md).
