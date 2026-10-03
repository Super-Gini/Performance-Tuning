# GV$SESSION

## Purpose

RAC-aware view of `V$SESSION` — same columns plus `INST_ID`. On single-instance databases, identical to `V$SESSION`.

## Extra Column

| Column    | Meaning                                 |
| --------- | --------------------------------------- |
| `INST_ID` | Instance number the session belongs to. |

## Common Queries

```sql
-- Sessions per instance
SELECT inst_id, COUNT(*) FROM gv$session GROUP BY inst_id;

-- Active users cluster-wide
SELECT inst_id, sid, serial#, username, event, sql_id, seconds_in_wait
FROM   gv$session
WHERE  status = 'ACTIVE' AND type = 'USER'
ORDER  BY inst_id, seconds_in_wait DESC;

-- Cross-instance blocking
SELECT   w.inst_id waiter_inst, w.sid waiter_sid,
         b.inst_id blocker_inst, b.sid blocker_sid,
         w.event, w.seconds_in_wait
FROM     gv$session w
JOIN     gv$session b ON b.sid = w.blocking_session AND b.inst_id = w.blocking_instance
WHERE    w.blocking_session IS NOT NULL;

-- Kill session on a specific instance
ALTER SYSTEM KILL SESSION '150,32458,@2' IMMEDIATE;   -- @2 = inst_id 2
```

## Related

- `V$SESSION` — this instance only.
- `GV$PROCESS`, `GV$SQL`, `GV$LOCK` — same pattern.

## Note

Every `V$` view has a `GV$` twin that adds `INST_ID`. Use `GV$` on RAC for cluster-wide views.

## References

- Oracle Database Reference 19c — `GV$SESSION`
