# MMON — Manageability Monitor

## Purpose

Runs periodic **management tasks**: AWR snapshot every hour, ADDM run after snapshot, alert threshold checks, and space advisor.

## Behavior

- Wakes on schedule (default hourly for AWR).
- Delegates real-time sampling to `MMNL`.
- Fires `DBMS_JOB` / `DBMS_SCHEDULER` triggers when thresholds cross.
- Space monitoring — flags 85%/97% tablespace fill.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(MMON)%';

-- AWR snapshot cadence
SELECT snap_interval, retention FROM dba_hist_wr_control;

-- Recent snapshots
SELECT snap_id, begin_interval_time, end_interval_time
FROM   dba_hist_snapshot
ORDER  BY snap_id DESC
FETCH  FIRST 10 ROWS ONLY;

-- Threshold events
SELECT * FROM v$alert_types WHERE state='STATEFUL' FETCH FIRST 20 ROWS ONLY;
SELECT * FROM v$threshold_types;
```

## Related Views

- `DBA_HIST_SNAPSHOT` — MMON's output.
- `V$THRESHOLD_TYPES` — MMON's rules.
- `V$ALERT_TYPES`.

## Common Issues

- **AWR snapshots not being taken** — MMON stuck or hung. Bounce instance or force `EXEC DBMS_WORKLOAD_REPOSITORY.CREATE_SNAPSHOT`.
- **MMON high CPU** — Statspack legacy noise, or a bug. Check `V$SESSION_WAIT` for the MMON session.
- **AWR retention exceeded** — Snapshots pile up. Change with `DBMS_WORKLOAD_REPOSITORY.MODIFY_SNAPSHOT_SETTINGS(retention => 43200)`.

## References

- Oracle Database Performance Tuning Guide 19c — AWR
- [AWR](../../12-performance-tuning/awr.md)
