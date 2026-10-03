# V$SYSTEM_EVENT

## Purpose

Cumulative wait event totals since instance startup. One row per event; not sampled — every wait is counted.

## Key Columns

| Column              | Meaning                                     |
| ------------------- | ------------------------------------------- |
| `EVENT`             | Event name.                                 |
| `WAIT_CLASS`        | Category.                                   |
| `TOTAL_WAITS`       | Number of waits.                            |
| `TOTAL_TIMEOUTS`    | Waits that timed out.                       |
| `TIME_WAITED`       | Total wait time (cs, hundredths of second). |
| `TIME_WAITED_MICRO` | Microseconds (higher precision).            |
| `AVERAGE_WAIT`      | `TIME_WAITED / TOTAL_WAITS` (cs).           |

## Common Queries

```sql
-- Top waits since startup
SELECT   event, wait_class, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(average_wait,3) avg_cs
FROM     v$system_event
WHERE    wait_class NOT IN ('Idle')
ORDER BY time_waited DESC
FETCH FIRST 20 ROWS ONLY;

-- By wait class
SELECT   wait_class, ROUND(SUM(time_waited)/100/60,1) minutes
FROM     v$system_event
WHERE    wait_class <> 'Idle'
GROUP BY wait_class
ORDER BY 2 DESC;
```

## Snapshot for Comparison

Take a baseline before a test:

```sql
CREATE TABLE t_events AS SELECT * FROM v$system_event;

-- run workload

SELECT   e.event, e.wait_class,
         e.total_waits - t.total_waits AS d_waits,
         ROUND((e.time_waited - t.time_waited)/100, 1) AS d_secs
FROM     v$system_event e JOIN t_events t USING (event)
WHERE    e.time_waited - t.time_waited > 0
   AND   e.wait_class <> 'Idle'
ORDER BY 4 DESC
FETCH FIRST 20 ROWS ONLY;
```

## References

- Oracle Database Reference 19c — `V$SYSTEM_EVENT`
- MOS Doc ID 61998.1 — Wait event reference
