# Monthly Checklist

First of the month, or first Monday.

## 1. RMAN Restore Rehearsal

- [ ] Full restore to test host in DR env.
- [ ] Verify RTO / RPO against SLA.
- [ ] Document the timing.

```
RMAN> RESTORE DATABASE PREVIEW;
RMAN> RESTORE DATABASE VALIDATE;
```

## 2. Growth Forecast

- [ ] Compare last 3 months of monthly growth.
- [ ] Project 6- and 12-month storage needs.
- [ ] Order storage if runway < 6 months.

```sql
SELECT   TRUNC(begin_interval_time,'MM') month,
         SUM(bytes_used)/1024/1024/1024 gb
FROM     dba_hist_tbspc_space_usage h
JOIN     dba_hist_snapshot s ON s.snap_id = h.snap_id
WHERE    begin_interval_time > ADD_MONTHS(SYSDATE, -6)
GROUP BY TRUNC(begin_interval_time,'MM')
ORDER BY 1;
```

## 3. Patch Currency

- [ ] Confirm all production DBs on approved RU tier.
- [ ] Plan next RU application window.

```sql
SELECT patch_id, description, action, status, action_time
FROM   dba_registry_sqlpatch
ORDER  BY action_time DESC
FETCH  FIRST 5 ROWS ONLY;
```

## 4. Security Review

- [ ] Full user/role/priv audit.
- [ ] Users not logged in > 90 days — lock.
- [ ] Password profile compliance.
- [ ] Unified audit review for anomalies.

```sql
SELECT username, last_login FROM dba_users
WHERE  (last_login < SYSDATE - 90 OR last_login IS NULL)
   AND account_status = 'OPEN'
   AND oracle_maintained = 'N';
```

## 5. Statistics Strategy

- [ ] Any tables with `NUM_ROWS = 0` but should have rows?
- [ ] Auto-stats job coverage sufficient?
- [ ] Custom stats gathering scripts running as expected?

## 6. AWR Long-Term Retention

- [ ] Retention aligned with policy (e.g., 90 days).
- [ ] Baselines defined for typical & peak periods.

```sql
SELECT * FROM dba_hist_baseline WHERE baseline_type = 'STATIC';
```

## 7. Data Guard

- [ ] Full switchover rehearsal in DR env.
- [ ] Broker configuration current.
- [ ] Verify Active DG reports still healthy.

## 8. Fabric Health

- [ ] ASM rebalances completed clean.
- [ ] OCR & voting backups (`ocrconfig -showbackup`).
- [ ] GI patch level current.

## 9. Documentation

- [ ] Update runbooks with any new incidents' lessons.
- [ ] Refresh architecture diagram if topology changed.
- [ ] Team knowledge share meeting.

## Signoff

Document. Open change tickets for planned work.

## Related

- [Daily](daily.md), [Weekly](weekly.md), [Quarterly](quarterly.md).
- [Production Health Checks](../35-production-health-checks/index.md).
