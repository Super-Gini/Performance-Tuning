# V$DATAGUARD_STATS

## Purpose

High-level Data Guard health — apply lag, transport lag, estimated failover time. One row per metric.

## Key Columns

| Column          | Meaning                                                                               |
| --------------- | ------------------------------------------------------------------------------------- |
| `NAME`          | Metric — `apply lag`, `transport lag`, `apply finish time`, `estimated startup time`. |
| `VALUE`         | Value as string (usually `+00 00:00:00.000` interval).                                |
| `UNIT`          | Interval unit or seconds.                                                             |
| `TIME_COMPUTED` | When this row was updated.                                                            |
| `DATUM_TIME`    | Data source timestamp.                                                                |

## Common Queries

```sql
-- Full health
SELECT name, value, unit, time_computed
FROM   v$dataguard_stats
ORDER  BY name;

-- One-liner apply lag
SELECT name, value
FROM   v$dataguard_stats
WHERE  name IN ('apply lag','transport lag');
```

Typical output (healthy):

```
NAME                VALUE                UNIT
------------------  -------------------  -----------------
apply finish time   +00 00:00:00.000     day(2) to second(3) interval
apply lag           +00 00:00:00         day(2) to second(0) interval
estimated startup   35                   seconds
transport lag       +00 00:00:00         day(2) to second(0) interval
```

## Extra Data Guard Views

| View                  | Purpose                             |
| --------------------- | ----------------------------------- |
| `V$DATABASE`          | `DATABASE_ROLE`, `PROTECTION_MODE`. |
| `V$MANAGED_STANDBY`   | Standby processes state.            |
| `V$STANDBY_LOG`       | Standby redo logs.                  |
| `V$ARCHIVE_GAP`       | Missing archives.                   |
| `V$RECOVERY_PROGRESS` | MRP0's activity.                    |
| `V$LOGSTDBY_STATS`    | Logical standby stats.              |
| `V$DATAGUARD_STATUS`  | Recent DG log messages.             |

Common query:

```sql
-- Apply status on standby
SELECT   process, status, thread#, sequence#, block#, blocks
FROM     v$managed_standby
WHERE    process IN ('MRP0','RFS')
ORDER BY process;

-- Gaps
SELECT * FROM v$archive_gap;
```

## References

- Oracle Data Guard Concepts and Administration 19c
- [Monitoring Data Guard](../../17-data-guard/monitoring.md)
