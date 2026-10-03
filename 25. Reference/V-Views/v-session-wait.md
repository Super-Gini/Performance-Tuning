# V$SESSION_WAIT

## Purpose

Wait event state per session — largely superseded by columns in `V$SESSION` since 10g, but kept for backward compat and for detailed wait event info without joining.

## Key Columns

| Column                 | Meaning                                                                        |
| ---------------------- | ------------------------------------------------------------------------------ |
| `SID`                  | Session.                                                                       |
| `SEQ#`                 | Sequence — increments each new wait.                                           |
| `EVENT`                | Wait event name.                                                               |
| `P1TEXT`, `P1`         | Parameter 1 (meaning depends on event).                                        |
| `P2TEXT`, `P2`         | Parameter 2.                                                                   |
| `P3TEXT`, `P3`         | Parameter 3.                                                                   |
| `WAIT_CLASS`           | Class (`User I/O`, `Concurrency`, `Commit`, ...).                              |
| `WAIT_TIME_MICRO`      | Duration in microseconds.                                                      |
| `TIME_REMAINING_MICRO` | For events with timeouts.                                                      |
| `STATE`                | `WAITING` / `WAITED KNOWN TIME` / `WAITED SHORT TIME` / `WAITED UNKNOWN TIME`. |

## Common Queries

```sql
-- All actively waiting sessions
SELECT sw.sid, s.username, sw.event, sw.wait_class,
       sw.p1text, sw.p1, sw.p2text, sw.p2, sw.wait_time_micro/1e6 secs
FROM   v$session_wait sw
JOIN   v$session      s ON s.sid = sw.sid
WHERE  sw.state = 'WAITING'
   AND s.status = 'ACTIVE'
ORDER  BY sw.wait_time_micro DESC;

-- What waits are most common right now (snapshot)
SELECT wait_class, event, COUNT(*)
FROM   v$session_wait
WHERE  state = 'WAITING'
GROUP  BY wait_class, event
ORDER  BY 3 DESC;
```

## When to Use V$SESSION_WAIT vs V$SESSION

`V$SESSION` includes the current wait info for actives. Use `V$SESSION_WAIT` when you want the P1/P2/P3 text descriptions without additional lookups, or on 9i where these columns weren't in `V$SESSION`.

## Related

- `V$SESSION` — includes same info.
- `V$SESSION_EVENT` — session's cumulative wait time per event.
- `V$SYSTEM_EVENT` — instance-wide cumulative.

## References

- Oracle Database Reference 19c — `V$SESSION_WAIT`
