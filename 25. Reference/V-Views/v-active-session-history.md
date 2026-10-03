# V$ACTIVE_SESSION_HISTORY

## Purpose

In-memory circular buffer of session activity samples — one snapshot per active session per second, maintained by `MMNL`. Retention: 30–60 minutes typically (buffer size = `2 MB × CPU_COUNT`).

`DBA_HIST_ACTIVE_SESS_HISTORY` is the persisted subset (every 10th sample).

## Key Columns

| Column                 | Meaning                                 |
| ---------------------- | --------------------------------------- |
| `SAMPLE_ID`            | Incrementing sample ID.                 |
| `SAMPLE_TIME`          | Wall-clock time of sample.              |
| `SESSION_ID / SERIAL#` | Session identifiers.                    |
| `USER_ID`              | User.                                   |
| `SQL_ID`               | Currently running SQL.                  |
| `SQL_CHILD_NUMBER`     | Cursor child.                           |
| `SQL_PLAN_HASH_VALUE`  | Plan.                                   |
| `SQL_PLAN_LINE_ID`     | Line in the plan we were on.            |
| `SQL_PLAN_OPERATION`   | e.g., `HASH JOIN`, `TABLE ACCESS FULL`. |
| `EVENT`                | Wait event (or NULL if on CPU).         |
| `WAIT_CLASS`           | Category.                               |
| `SESSION_STATE`        | `WAITING` or `ON CPU`.                  |
| `MODULE / ACTION`      | Application markers.                    |
| `SERVICE_HASH`         | Service.                                |
| `PGA_ALLOCATED`        | PGA at sample time.                     |
| `BLOCKING_SESSION`     | If waiting on lock.                     |
| `P1 / P2 / P3`         | Wait parameters.                        |
| `CURRENT_OBJ#`         | Object being touched.                   |
| `CON_ID`               | Container.                              |

## Common Queries

```sql
-- Top wait events in the last 5 min
SELECT   NVL(event,'ON CPU') event,
         COUNT(*) samples
FROM     v$active_session_history
WHERE    sample_time > SYSDATE - 5/1440
GROUP BY NVL(event,'ON CPU')
ORDER BY 2 DESC
FETCH FIRST 10 ROWS ONLY;

-- Top SQL by DB time in the last hour
SELECT   sql_id,
         COUNT(*) samples,       -- ~ seconds of DB time
         ROUND(COUNT(*)*10/60,1) approx_db_min
FROM     v$active_session_history
WHERE    sample_time > SYSDATE - 1/24
   AND   sql_id IS NOT NULL
GROUP BY sql_id
ORDER BY 2 DESC
FETCH FIRST 10 ROWS ONLY;

-- Session activity timeline
SELECT   sample_time, session_state, event,
         sql_id, sql_plan_operation, current_obj#
FROM     v$active_session_history
WHERE    session_id = &target_sid
   AND   sample_time > SYSDATE - 10/1440
ORDER BY sample_time DESC;

-- What was blocking session X at time Y?
SELECT   sample_time, session_id, event, blocking_session, sql_id
FROM     v$active_session_history
WHERE    blocking_session = &blocker_sid
   AND   sample_time BETWEEN &t1 AND &t2
ORDER BY sample_time;
```

## `DBA_HIST_ACTIVE_SESS_HISTORY`

Same columns, historical. Query pattern identical but with `SAMPLE_TIME` filter over days/weeks.

## References

- Oracle Database Performance Tuning Guide 19c — ASH
- [ASH](../../12-performance-tuning/ash.md)
