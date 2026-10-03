# Database Health Check

Full audit — architecture, config, storage, backups, security, performance, and operational readiness.

## 1. Identity & Version

```sql
SELECT instance_name, host_name, version_full, status, startup_time,
       ROUND(SYSDATE-startup_time,2) days_up
FROM   v$instance;

SELECT name, dbid, db_unique_name, database_role, open_mode, log_mode,
       flashback_on, force_logging, cdb
FROM   v$database;

SELECT patch_id, description, action_time
FROM   dba_registry_sqlpatch
ORDER  BY action_time DESC FETCH FIRST 5 ROWS ONLY;
```

Verify: current on supported RU (within 90 days); expected role/mode.

## 2. Init Parameters (Sanity)

```sql
SELECT   name, value, isdefault, ismodified
FROM     v$parameter
WHERE    ismodified <> 'FALSE'
   OR    name IN ('memory_target','sga_target','sga_max_size',
                  'pga_aggregate_target','pga_aggregate_limit',
                  'processes','sessions','open_cursors',
                  'db_files','db_block_size','db_cache_size',
                  'shared_pool_size','undo_retention','undo_tablespace',
                  'compatible','optimizer_features_enable',
                  'db_flashback_retention_target',
                  'use_large_pages','_optimizer_adaptive_statistics')
ORDER BY name;
```

Verify: ASMM (not AMM) on Linux with HugePages. Correct COMPATIBLE. No stray underscore params.

## 3. Memory Advice

```sql
SELECT * FROM v$sga_target_advice ORDER BY sga_size;
SELECT * FROM v$pga_target_advice ORDER BY pga_target_for_estimate;
SELECT * FROM v$db_cache_advice ORDER BY size_for_estimate;
```

Verify: current sizing near the "estimated response time" curve knee.

## 4. Storage

```sql
-- Tablespaces near capacity
SELECT tablespace_name, ROUND(used_percent,2) pct_used
FROM   dba_tablespace_usage_metrics
WHERE  used_percent > 80;

-- Autoextend disabled
SELECT tablespace_name, file_name FROM dba_data_files
WHERE  autoextensible='NO';

-- FRA
SELECT name, ROUND(space_used*100/space_limit,2) pct, number_of_files
FROM   v$recovery_file_dest;

-- Temp fill
SELECT tablespace_name, ROUND(bytes_used*100/(bytes_used+bytes_free),1) pct_used
FROM   v$temp_space_header;

-- Undo
SELECT ROUND(SUM(bytes)/1024/1024/1024,2) undo_gb
FROM   dba_data_files
WHERE  tablespace_name = (SELECT value FROM v$parameter WHERE name='undo_tablespace');
```

Verify: no TS > 90%, FRA < 80%, all files autoextend, undo sized for peak.

## 5. Redo Health

```sql
SELECT group#, thread#, sequence#, bytes/1024/1024 mb, members, archived, status
FROM   v$log;

-- Log switches per hour last 24h
SELECT TO_CHAR(first_time,'YYYY-MM-DD HH24') hr, COUNT(*)
FROM   v$log_history
WHERE  first_time > SYSDATE - 1
GROUP  BY TO_CHAR(first_time,'YYYY-MM-DD HH24')
ORDER  BY 1;

-- Multiplexing check
SELECT group#, COUNT(*) FROM v$logfile GROUP BY group#;
```

Verify: switch every 15–20 min at peak; multiplexed ≥ 2 members.

## 6. Backups

```sql
SELECT input_type, status, start_time, end_time,
       ROUND(input_bytes/1024/1024/1024,2) in_gb,
       ROUND(output_bytes/1024/1024/1024,2) out_gb
FROM   v$rman_backup_job_details
WHERE  start_time > SYSDATE - 14
ORDER  BY start_time DESC;

-- Last successful full
SELECT MAX(start_time) FROM v$rman_backup_job_details
WHERE  status='COMPLETED' AND input_type='DB FULL';

-- Retention
SHOW PARAMETER db_recovery_file_dest_size
```

Verify: full within retention window; no failures in 30 days; archives < 15 min behind.

## 7. Data Guard (if applicable)

```sql
SELECT name, value FROM v$dataguard_stats;

SELECT process, status, thread#, sequence# FROM v$managed_standby
WHERE  process IN ('MRP0','RFS');

SELECT * FROM v$archive_gap;

SELECT dest_id, dest_name, status, error FROM v$archive_dest_status
WHERE  status NOT IN ('INACTIVE');
```

Verify: no gaps, no errors, apply lag < SLA.

## 8. RAC (if applicable)

```bash
crsctl check cluster -all
crsctl status resource -t | head -30

srvctl status database -db PRD
srvctl status service -db PRD
```

```sql
SELECT inst_id, instance_name, host_name, status FROM gv$instance;
SELECT name, network_name FROM gv$services;
```

Verify: all nodes ONLINE, no eviction events last 30 days.

## 9. Session Limits

```sql
SELECT resource_name, current_utilization, max_utilization, limit_value
FROM   v$resource_limit
WHERE  resource_name IN ('processes','sessions','enqueue_locks',
                         'transactions','parallel_max_servers');
```

Verify: max_utilization < 80% of limit.

## 10. Wait Events (Baseline)

```sql
SELECT wait_class, ROUND(time_waited/100/60,1) mins
FROM   v$system_event
WHERE  wait_class NOT IN ('Idle')
GROUP  BY wait_class
ORDER  BY 2 DESC;

SELECT event, ROUND(time_waited/100/60,1) mins
FROM   v$system_event
WHERE  wait_class NOT IN ('Idle')
ORDER  BY time_waited DESC
FETCH  FIRST 15 ROWS ONLY;
```

Verify: dominant events match workload (User I/O for OLTP, Commit for high write, etc.).

## 11. Top SQL

```sql
SELECT sql_id, executions,
       ROUND(elapsed_time/1e6,2) elapsed_secs,
       ROUND(elapsed_time/executions/1e6,3) sec_per_exec,
       ROUND(cpu_time/1e6,2) cpu_secs,
       buffer_gets, rows_processed
FROM   v$sqlarea
WHERE  executions > 0
ORDER  BY elapsed_time DESC
FETCH  FIRST 10 ROWS ONLY;
```

Verify: nothing surprising in top-10.

## 12. Invalid Objects & Fatal Errors

```sql
SELECT owner, object_type, COUNT(*)
FROM   dba_objects
WHERE  status='INVALID'
GROUP  BY owner, object_type
ORDER  BY 3 DESC;

-- Fatal errors last 7 days
SELECT COUNT(*) FROM v$diag_alert_ext
WHERE  originating_timestamp > SYSDATE - 7
   AND (message_text LIKE '%ORA-00600%' OR message_text LIKE '%ORA-07445%'
        OR message_text LIKE '%ORA-01578%');
```

Verify: zero invalid SYS/SYSTEM objects; zero 600/7445/1578.

## 13. Statistics

```sql
-- Stale stats
SELECT owner, table_name, last_analyzed, num_rows, stale_stats
FROM   dba_tab_statistics
WHERE  stale_stats='YES' AND owner NOT IN ('SYS','SYSTEM')
ORDER  BY num_rows DESC
FETCH  FIRST 20 ROWS ONLY;

-- Auto-stats job state
SELECT client_name, status FROM dba_autotask_client;
```

Verify: auto-stats enabled and running.

## 14. Security Snapshot

```sql
-- Non-Oracle DBA-role grants
SELECT * FROM dba_role_privs
WHERE  granted_role IN ('DBA','IMP_FULL_DATABASE','EXP_FULL_DATABASE',
                        'SELECT_CATALOG_ROLE','SYSDBA')
   AND grantee NOT IN ('SYS','SYSTEM','SYSMAN');

-- Default passwords
SELECT username FROM dba_users_with_defpwd;

-- Users never logging in but OPEN
SELECT username, account_status, last_login FROM dba_users
WHERE  account_status='OPEN' AND (last_login IS NULL OR last_login < SYSDATE - 90)
   AND oracle_maintained='N';
```

Verify: least-privilege model, no default passwords.

## 15. Deliverable Template

Structure findings by severity:

| Severity | Category | Finding                              | Action                        |
| -------- | -------- | ------------------------------------ | ----------------------------- |
| CRITICAL | Backup   | No successful full backup in 8 days  | Fix RMAN failure immediately  |
| HIGH     | Space    | USERS_DATA at 92%, no autoextend     | Enable autoextend + grow      |
| HIGH     | Security | 3 default passwords found            | Change passwords              |
| MEDIUM   | Perf     | Buffer cache hit ratio 89% (was 96%) | Investigate scans, grow cache |
| MEDIUM   | Config   | `optimizer_adaptive_statistics=TRUE` | Set FALSE for 19c stability   |
| LOW      | Doc      | Missing runbook for backup failure   | Author + review               |

## Related

- [Daily Checklist](../28-checklists/daily.md).
- [RAC Health Check](rac-health-check.md), [Data Guard Health Check](data-guard-health-check.md).
- [Security Health Check](security-health-check.md).
- [Storage Health Check](storage-health-check.md).
