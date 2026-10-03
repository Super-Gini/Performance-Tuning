# Daily Checklist

Every morning, first hour. If any item is red, escalate.

## 1. Instance & Cluster Health

- [ ] All production instances `OPEN`.
- [ ] No unplanned restart overnight (`v$instance.startup_time`).
- [ ] RAC — all nodes ONLINE (`crsctl status resource -t`).
- [ ] Data Guard — apply lag < 15 min on standby(s).

```sql
SELECT instance_name, host_name, status, startup_time,
       ROUND(SYSDATE - startup_time, 2) days_up
FROM   v$instance;

SELECT name, value FROM v$dataguard_stats WHERE name IN ('apply lag','transport lag');
```

## 2. Alert Log Sweep

- [ ] No `ORA-00600` in last 24 h.
- [ ] No `ORA-07445`.
- [ ] No `ORA-01578` (block corruption).
- [ ] No `ORA-01555`.
- [ ] No `ORA-00060` (deadlocks).
- [ ] No `ORA-00257` (archiver).
- [ ] No `ORA-04031`.

```sql
SELECT COUNT(*), MIN(message_text)
FROM   v$diag_alert_ext
WHERE  originating_timestamp > SYSDATE - 1
   AND (message_text LIKE '%ORA-00600%' OR message_text LIKE '%ORA-07445%'
        OR message_text LIKE '%ORA-01578%' OR message_text LIKE '%ORA-00060%'
        OR message_text LIKE '%ORA-00257%' OR message_text LIKE '%ORA-04031%');
```

## 3. Backup Verification

- [ ] Last night's full/incremental backup succeeded.
- [ ] Archive log backups current (< 1 h old).
- [ ] Backup size in expected range.

```sql
SELECT   input_type, status, start_time,
         ROUND(input_bytes/1024/1024/1024,2) input_gb,
         ROUND((end_time-start_time)*24*60,1) mins
FROM     v$rman_backup_job_details
WHERE    start_time > SYSDATE - 1
ORDER BY start_time DESC;
```

## 4. Space

- [ ] No tablespace > 90%.
- [ ] TEMP < 80%.
- [ ] FRA < 80%.
- [ ] Filesystems < 85%.

```sql
SELECT tablespace_name, ROUND(used_percent,1) pct
FROM   dba_tablespace_usage_metrics
WHERE  used_percent > 80
ORDER  BY 2 DESC;

SELECT name, ROUND(space_used*100/space_limit,1) pct FROM v$recovery_file_dest;
```

## 5. Sessions

- [ ] Session count < 80% of `PROCESSES`.
- [ ] No sessions blocked > 15 min.
- [ ] No sessions active > 4 h (unless expected batch).

```sql
SELECT resource_name, current_utilization, max_utilization, limit_value
FROM   v$resource_limit
WHERE  resource_name IN ('processes','sessions');

SELECT COUNT(*) FROM v$session
WHERE  blocking_session IS NOT NULL AND seconds_in_wait > 900;
```

## 6. Job Failures

- [ ] All Scheduler jobs green.
- [ ] Any `BROKEN` state investigated.

```sql
SELECT owner, job_name, state, failure_count, last_start_date
FROM   dba_scheduler_jobs
WHERE  state IN ('FAILED','BROKEN') AND enabled='TRUE';
```

## 7. Data Guard Detailed (if applicable)

- [ ] Apply lag < SLA.
- [ ] No archive gaps.
- [ ] Broker state = SUCCESS.

```sql
SELECT * FROM v$archive_gap;
```

## 8. AWR Sanity

- [ ] AWR snapshots collected every hour last 24 h.
- [ ] No abnormal DB Time spike vs prior day.

```sql
SELECT snap_id, begin_interval_time
FROM   dba_hist_snapshot
WHERE  begin_interval_time > SYSDATE - 1
ORDER  BY snap_id DESC;
```

## 9. Standby Sync Verification (Data Guard)

- [ ] Latest sequence# on primary + standby match within tolerance.

```sql
-- Primary
SELECT MAX(sequence#) primary_seq FROM v$archived_log WHERE thread# = 1;
-- Standby
SELECT MAX(sequence#) standby_seq FROM v$archived_log WHERE thread# = 1 AND applied='YES';
```

## Signoff

Log the daily check with results and escalations opened.

## Related

- [Weekly](weekly.md), [Monthly](monthly.md).
- [Alert Log](../24-monitoring/alert-log.md).
- [Scripts](../30-scripts/index.md).
