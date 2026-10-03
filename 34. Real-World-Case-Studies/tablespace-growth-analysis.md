# Case: Unplanned 400 GB/Month Growth

## Setup

- 19c EE, ecommerce app.
- DB size: 8 TB (baseline steady growth ~50 GB/month).
- Last quarter: **400 GB/month** growth. No new customers, no new tables.
- Storage team asking why.

## Investigation

### Step 1 — Which Tablespace?

```sql
SELECT   t.tablespace_id,
         ts.tablespace_name,
         ROUND(MIN(t.tablespace_usedsize)*8/1024/1024, 2) start_gb,
         ROUND(MAX(t.tablespace_usedsize)*8/1024/1024, 2) end_gb,
         ROUND((MAX(t.tablespace_usedsize)-MIN(t.tablespace_usedsize))*8/1024/1024, 2) growth_gb
FROM     dba_hist_tbspc_space_usage t
JOIN     v$tablespace ts ON ts.ts# = t.tablespace_id
WHERE    t.snap_id BETWEEN
             (SELECT MIN(snap_id) FROM dba_hist_snapshot
              WHERE end_interval_time > SYSDATE - 90)
         AND (SELECT MAX(snap_id) FROM dba_hist_snapshot)
GROUP BY t.tablespace_id, ts.tablespace_name
HAVING   (MAX(t.tablespace_usedsize)-MIN(t.tablespace_usedsize))*8/1024/1024 > 10
ORDER BY 5 DESC;
```

Result:

```
TABLESPACE_NAME          START_GB    END_GB    GROWTH_GB
APP_LOG_DATA             450         1250       800    <-- outlier
USERS_DATA               2000        2100       100
USERS_IDX                800         840        40
```

`APP_LOG_DATA` grew 800 GB in 90 days — 2× ordinary rate.

### Step 2 — Which Segments in `APP_LOG_DATA`?

```sql
SELECT   owner, segment_name, segment_type,
         ROUND(bytes/1024/1024/1024, 2) gb
FROM     dba_segments
WHERE    tablespace_name = 'APP_LOG_DATA'
ORDER BY bytes DESC
FETCH FIRST 15 ROWS ONLY;
```

Result:

```
OWNER   SEGMENT_NAME              TYPE      GB
APP     AUDIT_TRAIL_HIST          TABLE     720
APP     API_REQUEST_LOG           TABLE     380
APP     USER_ACTIVITY_LOG         TABLE     150
```

`AUDIT_TRAIL_HIST` is the giant. Let's check its history.

### Step 3 — Segment Growth via ASH

```sql
SELECT   s.owner_name, s.object_name, s.subobject_name,
         MIN(h.begin_interval_time) start_time,
         MAX(h.end_interval_time)   end_time,
         MIN(s.space_used_delta) min_delta,
         MAX(s.space_used_delta) max_delta,
         SUM(s.space_used_delta)/1024/1024/1024 total_growth_gb
FROM     dba_hist_seg_stat s
JOIN     dba_hist_snapshot h USING (snap_id)
WHERE    h.end_interval_time > SYSDATE - 90
   AND   s.object_name = 'AUDIT_TRAIL_HIST'
GROUP BY s.owner_name, s.object_name, s.subobject_name
ORDER BY 8 DESC;
```

Result:

```
OWNER_NAME  OBJECT_NAME            TOTAL_GROWTH_GB
APP         AUDIT_TRAIL_HIST       680
```

680 GB in 90 days. That's 7.5 GB/day, ~230 GB/month for this one table.

### Step 4 — Row Growth?

```sql
SELECT COUNT(*) FROM app.audit_trail_hist;
-- 4,200,000,000 rows

-- 3 months ago
SELECT   BLOCKS, ROW_STATS
FROM     dba_hist_tab_stats
WHERE    table_name = 'AUDIT_TRAIL_HIST'
   AND   snap_time > SYSDATE - 100 AND snap_time < SYSDATE - 90
ORDER BY snap_time;
```

Rows about 3 months ago: 2.6 billion. Now: 4.2 billion. 1.6 billion new rows in 90 days = ~600 million/month.

### Step 5 — Who's Writing?

Check application audit config:

```sql
SELECT VALUE FROM V$PARAMETER WHERE NAME = 'audit_trail';
-- 'DB'
SELECT VALUE FROM V$PARAMETER WHERE NAME = 'audit_sys_operations';
-- 'TRUE'
```

Hmm. Also check unified audit:

```sql
SELECT * FROM audit_unified_enabled_policies;
```

Result:

```
ORA_LOGON_FAILURES        Enabled
ORA_SECURECONFIG          Enabled
ORA_CIS_RECOMMENDATIONS   Enabled     <-- new
ORA_LOGON_LOGOFF          Enabled     <-- new
```

`ORA_LOGON_LOGOFF` was enabled 4 months ago (matches the growth window). It audits every logon and logoff.

Confirm:

```sql
SELECT event_timestamp, action_name, dbusername, os_username
FROM   unified_audit_trail
WHERE  action_name IN ('LOGON','LOGOFF')
   AND event_timestamp > SYSDATE - 1
ORDER  BY event_timestamp DESC
FETCH  FIRST 20 ROWS ONLY;

SELECT COUNT(*) FROM unified_audit_trail
WHERE  event_timestamp > SYSDATE - 1;
```

Result: 15 million rows/day just from LOGON/LOGOFF. Multiply by 30 days = 450 million rows/month.

But wait — the app has connection pooling; each user only logs in a few times a day. Where do the extra logons come from?

### Step 6 — Trace the Source

```sql
SELECT dbusername, machine, COUNT(*) audits
FROM   unified_audit_trail
WHERE  event_timestamp > SYSDATE - 1
   AND action_name = 'LOGON'
GROUP  BY dbusername, machine
ORDER  BY 3 DESC
FETCH  FIRST 20 ROWS ONLY;
```

Result:

```
DBUSERNAME       MACHINE                  AUDITS
DBSNMP           oemhost01                8,200,000    <-- OEM agent
PERFSTAT         monitoring-1             2,100,000
APP_HEALTHCHECK  loadbalancer-1           1,900,000
APP_HEALTHCHECK  loadbalancer-2           1,900,000
```

- **OEM agent** (`DBSNMP`) connects 8M times/day for metric collection.
- **Monitoring** connects 2M times/day.
- **Load balancer health check** connects 4M times/day (2 LBs × 2M each).

Each of these = 1 LOGON + 1 LOGOFF audit row.

## Root Cause

`ORA_LOGON_LOGOFF` policy enabled 4 months ago, generating audits for every monitoring connection. High-frequency machine connections dominated.

## Fix

### Option A — Exclude Monitoring Users from Audit

```sql
BEGIN
  DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    use_last_arch_timestamp => TRUE);
END;
/

CREATE AUDIT POLICY logon_policy_no_monitoring
   ACTIONS LOGON, LOGOFF
   WHEN 'SYS_CONTEXT(''USERENV'',''SESSION_USER'') NOT IN (''DBSNMP'',''PERFSTAT'',''APP_HEALTHCHECK'')'
   EVALUATE PER SESSION;

NOAUDIT POLICY ORA_LOGON_LOGOFF;
AUDIT POLICY logon_policy_no_monitoring;
```

### Option B — Reduce Retention

If compliance requires all LOGON audits, at least reduce retention:

```sql
BEGIN
  DBMS_AUDIT_MGMT.SET_LAST_ARCHIVE_TIMESTAMP(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    last_archive_time => SYSTIMESTAMP - INTERVAL '30' DAY);

  DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    use_last_arch_timestamp => TRUE);
END;
/

-- Schedule quarterly cleanup
```

### Option C — Both

Applied both. Growth rate dropped back to ~60 GB/month baseline within 30 days.

## Cleanup

```sql
-- One-time to reclaim the space
ALTER TABLE app.audit_trail_hist ENABLE ROW MOVEMENT;
ALTER TABLE app.audit_trail_hist SHRINK SPACE CASCADE;

-- Or (if partitioned) drop old partitions
ALTER TABLE app.audit_trail_hist DROP PARTITION P_2024_Q1;
ALTER TABLE app.audit_trail_hist DROP PARTITION P_2024_Q2;
```

## Lessons Learned

- **Audit policies can generate massive growth** — always check when growth doesn't match business volume.
- **Monitoring connections** are the noisiest audit source. Exclude them.
- **`dba_hist_seg_stat`** is invaluable for per-segment growth trending.
- **`dba_hist_tbspc_space_usage`** for tablespace-level trending.
- **Set retention on audit trail** — no data lives forever without policy.
- Any new audit policy = growth analysis 30 days later.

## Related

- [Auditing](../10-security/auditing.md).
- [Unified Audit](../10-security/unified-audit.md).
- [Growth Forecast (Monthly checklist)](../28-checklists/monthly.md).
