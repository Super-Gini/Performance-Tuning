# Runbook: High CPU

## Symptom

- Host `top` shows `oracle` at 90%+ CPU.
- Application slow.
- OEM Performance Home CPU wait dominant.

## Triage

### OS side

```bash
top -b -n 1 | head -30

# Threads (Linux)
top -H -p <pid>

# What Oracle processes are heavy
ps -eo pid,pcpu,pmem,rss,comm | sort -k2 -nr | head -20
```

### DB side

```sql
-- Sessions on CPU right now
SELECT   s.sid, s.serial#, s.username, s.machine, s.program,
         s.sql_id, s.status, s.event, p.spid
FROM     v$session s JOIN v$process p ON p.addr = s.paddr
WHERE    s.status = 'ACTIVE' AND s.type = 'USER'
   AND   (s.event IS NULL OR s.event LIKE 'CPU%')
ORDER BY s.last_call_et DESC;

-- Top SQL by CPU right now (V$SQL)
SELECT   sql_id, ROUND(cpu_time/1e6,1) cpu_secs, executions,
         ROUND(elapsed_time/executions/1e6,3) sec_per_exec,
         substr(sql_text,1,80)
FROM     v$sql
WHERE    last_active_time > SYSDATE - 5/1440
ORDER BY cpu_time DESC
FETCH FIRST 10 ROWS ONLY;

-- ASH-based CPU consumers (last 15 min)
SELECT   sql_id, COUNT(*) samples
FROM     v$active_session_history
WHERE    sample_time > SYSDATE - 15/1440
   AND   session_state = 'ON CPU'
   AND   sql_id IS NOT NULL
GROUP BY sql_id
ORDER BY 2 DESC
FETCH FIRST 10 ROWS ONLY;
```

## Actions

### 1. Identify the culprit

Match top SPIDs from OS with SIDs in V$SESSION:

```sql
SELECT s.sid, s.serial#, s.username, s.sql_id, s.event, p.spid
FROM   v$session s JOIN v$process p ON p.addr = s.paddr
WHERE  p.spid IN ('&spid1', '&spid2');
```

### 2. Get the SQL

```sql
SELECT sql_fulltext FROM v$sql WHERE sql_id = '&sql_id';
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id'));
```

### 3. Immediate mitigation

- **Runaway query** — kill the session.
- **Legitimate but bad plan** — see if a good plan exists in AWR:
  ```sql
  SELECT plan_hash_value, ROUND(AVG(elapsed_time_delta/executions_delta/1e6),3) avg_sec
  FROM   dba_hist_sqlstat WHERE sql_id='&sql_id' AND executions_delta > 0
  GROUP  BY plan_hash_value ORDER BY 2;
  ```
  Load a good plan via SQL Plan Baseline.
- **Legitimate load spike** — Resource Manager to throttle.

### 4. Broader saturation

If dozens of small SQLs all CPU-bound:

- Check parse rate (`hard parses/sec`) — bind variables missing.
- Check parallel execution: `SELECT COUNT(*) FROM v$px_process`.
- Check adaptive plans / SQL Monitor for large scans.

### 5. Kill runaways

```sql
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```

### 6. Throttle via Resource Manager

```sql
-- Activate a plan that caps CPU per group
ALTER SYSTEM SET resource_manager_plan = 'BATCH_PLAN' SCOPE=BOTH;
```

## Verification

```sql
-- CPU pressure released?
SELECT metric_name, value FROM v$sysmetric
WHERE  metric_name IN ('Host CPU Utilization (%)','Database Time Per Sec')
   AND  intsize_csec = 6000;
```

`top` shows healthy CPU distribution.

## Post-Mortem

- Was this a known query with a bad plan? Add SPB.
- Was it new code? App team fix.
- Was it parse-heavy (soft/hard)? Bind variables?
- Was Resource Manager active? Should it have been?

## Prevention

- SPB for critical queries.
- Bind variables enforced.
- Resource Manager: hard CPU cap for non-critical groups.
- Alert on `Host CPU Utilization` > 85%.
- AWR baselines for compare.

## Related

- [CPU Analysis](../12-performance-tuning/cpu-analysis.md).
- [ASH](../12-performance-tuning/ash.md).
- [Top SQL](../12-performance-tuning/top-sql.md).
- [Resource Manager](../09-user-management/resource-manager.md).
