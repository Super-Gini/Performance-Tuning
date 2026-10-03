# Weekly Checklist

Every Monday morning (or your team's fixed weekly slot).

## 1. Backup Sanity Deep

- [ ] Restore-validate at least one datafile via `RESTORE VALIDATE`.
- [ ] Confirm retention policy still met (`REPORT OBSOLETE`).
- [ ] Check off-site copies synced.

```
RMAN> RESTORE DATABASE VALIDATE;
RMAN> REPORT OBSOLETE;
RMAN> REPORT NEED BACKUP;
```

## 2. Data Guard Deep

- [ ] Verify apply on both primary and standby.
- [ ] Run role transition rehearsal (in DR env).
- [ ] Check redo lag trend over the week.

```sql
SELECT to_char(sample_time,'YYYY-MM-DD HH24:MI') t,
       apply_lag, transport_lag
FROM   dba_hist_dataguard_stats_bh
WHERE  sample_time > SYSDATE - 7
ORDER  BY 1;
```

## 3. Space Trends

- [ ] Tablespace growth per week.
- [ ] Which schemas grew most?
- [ ] Do any need bigfile conversion soon?

```sql
SELECT   tablespace_name,
         MIN(tablespace_usedsize) start_used,
         MAX(tablespace_usedsize) end_used,
         MAX(tablespace_usedsize) - MIN(tablespace_usedsize) growth_blocks
FROM     dba_hist_tbspc_space_usage h
WHERE    snap_id BETWEEN
             (SELECT MAX(snap_id) FROM dba_hist_snapshot
              WHERE end_interval_time < SYSDATE - 7)
         AND (SELECT MAX(snap_id) FROM dba_hist_snapshot)
GROUP BY tablespace_name
ORDER BY 4 DESC;
```

## 4. Performance Baseline

- [ ] AWR compare last week vs prior week.
- [ ] Top SQL by DB time — any new entrants?
- [ ] Plan-flip candidates?

```sql
-- SQL with plan variance
SELECT sql_id, COUNT(DISTINCT plan_hash_value) plans,
       MAX(elapsed_time_delta/executions_delta/1e6) max_secs_per_exec
FROM   dba_hist_sqlstat
WHERE  snap_id BETWEEN &start_snap AND &end_snap
   AND executions_delta > 0
GROUP  BY sql_id
HAVING COUNT(DISTINCT plan_hash_value) > 1
ORDER  BY 2 DESC;
```

## 5. Statistics

- [ ] Objects with stale stats or `NUM_ROWS = 0`.
- [ ] Auto-stats job succeeded in maintenance window.
- [ ] Fixed / dictionary stats current.

```sql
SELECT owner, table_name, last_analyzed, num_rows, stale_stats
FROM   dba_tab_statistics
WHERE  stale_stats = 'YES'
   AND owner NOT IN ('SYS','SYSTEM')
ORDER  BY num_rows DESC
FETCH  FIRST 30 ROWS ONLY;
```

## 6. Security

- [ ] Failed logon spikes (`AUD$` or unified audit).
- [ ] Any DBA-role grants added?
- [ ] Password rotations due?

```sql
SELECT username, expiry_date FROM dba_users
WHERE  expiry_date BETWEEN SYSDATE AND SYSDATE + 30;

-- Recent DBA/SYSDBA grants
SELECT * FROM dba_role_privs
WHERE  granted_role IN ('DBA','SYSDBA')
   AND grantee NOT IN ('SYS','SYSTEM')
ORDER  BY grantee;
```

## 7. Log Purge

- [ ] Alert log rotated.
- [ ] Trace files > 30 days purged (ADR).
- [ ] Scheduler run history purge (`DBMS_SCHEDULER.PURGE_LOG`).

## 8. Cluster + ASM

- [ ] `crsctl check crs` all nodes OK.
- [ ] ASM DGs healthy, no rebalance pending.
- [ ] Voting / OCR backups current.

## Signoff

Log; open tickets for items needing follow-up.

## Related

- [Daily](daily.md), [Monthly](monthly.md).
- [Database Health Check](../35-production-health-checks/database-health-check.md).
