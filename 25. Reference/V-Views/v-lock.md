# V$LOCK

## Purpose

Every lock held or requested by a session — DML locks (TM, TX) and system enqueues (CF, RO, SS, etc.).

## Key Columns

| Column    | Meaning                                                                                                                         |
| --------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `SID`     | Session holding/requesting.                                                                                                     |
| `TYPE`    | Enqueue type: `TX` (transaction), `TM` (DML), `UL` (user lock), `CF` (control file), `RO` (row-open), `SS` (sort segment), etc. |
| `ID1`     | Type-specific first identifier (for `TX` = rollback slot).                                                                      |
| `ID2`     | Type-specific second identifier (for `TX` = wrap#).                                                                             |
| `LMODE`   | Lock mode held: 0=None, 1=Null, 2=RS, 3=RX, 4=S, 5=SRX, 6=X.                                                                    |
| `REQUEST` | Lock mode requested (0 if just holding).                                                                                        |
| `CTIME`   | Seconds since lock started.                                                                                                     |
| `BLOCK`   | 1 if this lock is blocking someone else; 2 if blocking global.                                                                  |

## Lock Modes

| Code | Symbol | Meaning                       |
| ---- | ------ | ----------------------------- |
| 0    | None   | No lock                       |
| 1    | Null   | Placeholder                   |
| 2    | RS     | Row Share (SS)                |
| 3    | RX     | Row Exclusive (SX) — most DML |
| 4    | S      | Share                         |
| 5    | SRX    | Share Row Exclusive           |
| 6    | X      | Exclusive                     |

## Common Queries

```sql
-- Blocking session summary
SELECT   s.sid, s.serial#, s.username, s.status, l.type, l.lmode, l.request,
         l.id1, l.id2, l.block, l.ctime
FROM     v$lock l JOIN v$session s ON s.sid = l.sid
WHERE    l.block > 0 OR l.request > 0
ORDER BY l.block DESC, l.ctime DESC;

-- Waiter -> blocker map (via V$SESSION.BLOCKING_SESSION for quicker access)
SELECT   w.sid waiter_sid, w.username waiter_user, w.event,
         b.sid blocker_sid, b.username blocker_user,
         w.blocking_session, w.seconds_in_wait
FROM     v$session w JOIN v$session b ON b.sid = w.blocking_session
WHERE    w.blocking_session IS NOT NULL;

-- TX lock's rollback slot -> segment
SELECT   s.sid, l.type, l.id1, l.id2,
         ROUND(l.id1/POWER(2,16)) rbs, MOD(l.id1, POWER(2,16)) slot,
         l.id2 wrap#
FROM     v$lock l JOIN v$session s ON s.sid = l.sid
WHERE    l.type = 'TX' AND l.lmode > 0;

-- Global (system) enqueues right now
SELECT type, COUNT(*) FROM v$lock WHERE type NOT IN ('TX','TM') GROUP BY type;
```

## References

- Oracle Database Reference 19c — `V$LOCK`
- MOS Doc ID 62354.1 — Enqueue types
- [TM Locks](../../13-locking/tm-locks.md), [TX Locks](../../13-locking/tx-locks.md)
