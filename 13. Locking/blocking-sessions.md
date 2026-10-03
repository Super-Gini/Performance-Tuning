# Blocking Sessions

## Overview

A **blocking session** is a session holding a lock that other sessions need. The most common performance incident: a user's UPDATE forgets to commit, an application transaction lingers across user think-time, and dozens of sessions pile up waiting on `enq: TX - row lock contention`.

Resolving blockers is a routine DBA activity. Doing it safely (killing the _right_ session; not just the first one) is the skill.

## Quick Triage Query

```sql
-- Every blocked session and its blocker
SELECT s.sid, s.serial#, s.username, s.machine, s.program,
       s.blocking_session AS blocker_sid, s.event,
       s.seconds_in_wait AS wait_sec, s.sql_id
FROM   v$session s
WHERE  s.blocking_session IS NOT NULL
ORDER  BY s.seconds_in_wait DESC;
```

## Follow the Chain

The `blocking_session` might itself be blocked. Find the **ultimate blocker** (nobody blocks them):

```sql
-- Blocking tree
SELECT LPAD(' ', (LEVEL - 1) * 2) || s.sid AS "sid",
       s.serial#, s.username, s.status, s.event, s.seconds_in_wait,
       s.sql_id, s.machine
FROM   v$session s
START WITH s.blocking_session IS NULL
       AND s.sid IN (SELECT blocking_session FROM v$session WHERE blocking_session IS NOT NULL)
CONNECT BY PRIOR s.sid = s.blocking_session
ORDER SIBLINGS BY s.sid;
```

The top of the tree is the ultimate blocker. Killing it releases the entire chain.

## Gather Context Before Killing

```sql
-- What is the blocker doing?
SELECT s.sid, s.serial#, s.username, s.machine, s.program,
       s.module, s.action, s.client_info,
       s.status, s.last_call_et AS idle_sec,
       s.sql_id, s.prev_sql_id,
       (SELECT sql_text FROM v$sql WHERE sql_id = s.sql_id) AS current_sql,
       (SELECT sql_text FROM v$sql WHERE sql_id = s.prev_sql_id) AS prev_sql
FROM   v$session s
WHERE  s.sid = &blocker_sid;

-- Which locks does the blocker hold?
SELECT l.type, l.lmode, l.request, l.id1, l.id2,
       o.owner || '.' || o.object_name AS object
FROM   v$lock l LEFT JOIN dba_objects o ON o.object_id = l.id1 AND l.type = 'TM'
WHERE  l.sid = &blocker_sid;

-- OS process ID
SELECT p.spid, p.program FROM v$process p, v$session s
WHERE  s.sid = &blocker_sid AND s.paddr = p.addr;
```

Understand:

- **INACTIVE** with `last_call_et` > 3600 — a session stuck in application think-time. Safe to kill.
- **ACTIVE** running long SQL — killing may lose real work. Confirm with application owner.
- Blocker in a critical batch — investigate whether killing violates SLAs.

## Killing a Session

### Within Oracle

```sql
ALTER SYSTEM KILL SESSION '&sid,&serial#' IMMEDIATE;
```

`IMMEDIATE` tells PMON to roll back and clean up immediately (versus letting the session notice on next call).

### If Oracle-side kill hangs

```sql
-- Get OS pid
SELECT p.spid FROM v$process p, v$session s
WHERE  s.sid = &sid AND s.paddr = p.addr;

-- At OS level (as oracle user)
-- Linux:
kill -9 <spid>
```

OS kill triggers PMON cleanup. Use only if `ALTER SYSTEM KILL` doesn't work — usually because the session is stuck at OS level (network wait, deadlocked with the kernel).

### `DISCONNECT SESSION`

```sql
ALTER SYSTEM DISCONNECT SESSION '&sid,&serial#' IMMEDIATE;
```

Similar to kill but forces network disconnect. Useful when the client is hanging on a TCP call.

## Special Cases

### Blocker is SYS or an internal process

Never kill SYS or background processes. If SYS holds a lock blocking users, escalate to Oracle Support.

### `KILLED` status persists

Session shows `STATUS = KILLED` in `V$SESSION` but still there. PMON is waiting for the OS process to exit. Force with `kill -9 <spid>`.

### Blocker in a database link (distributed transaction)

Check `DBA_2PC_PENDING` for in-doubt transactions. `COMMIT FORCE '<txn_id>';` or `ROLLBACK FORCE '<txn_id>';` after verifying remote state.

### Blocker across RAC nodes

`V$SESSION.BLOCKING_SESSION` handles cross-instance blockers. `V$SESSION.BLOCKING_INSTANCE` tells you the node. Kill with `ALTER SYSTEM KILL SESSION 'sid,serial#,@instance_number'`.

## Historical Blocking (via ASH)

If the incident is over but you want to know what happened:

```sql
SELECT sample_time, session_id, blocking_session,
       event, wait_class, sql_id, current_obj#
FROM   v$active_session_history
WHERE  blocking_session IS NOT NULL
   AND sample_time BETWEEN TIMESTAMP '2026-08-06 14:00' AND TIMESTAMP '2026-08-06 14:30'
ORDER  BY sample_time;
```

## Preventing Repeats

- **Application review**: any transaction that spans user think-time is a bug.
- **`ALTER PROFILE ... IDLE_TIME`** — force disconnect of idle sessions.
- **Resource Manager `SWITCH_TIME`** — switch/kill long-running queries.
- **Session timeout on JDBC connection pool** — configure app-side.
- **Alert on blocking sessions** persistent > 60 seconds.

## Best Practices

1. **Follow the chain to the ultimate blocker.**
2. Gather context before killing — module, program, SQL, duration.
3. Prefer `ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE`.
4. OS-level kill only if Oracle-side fails.
5. In RAC, include `@instance_number` in KILL.
6. Notify the application team when killing user sessions.
7. Log the incident (sql_text, session_info) for post-mortem.
8. Alert on prolonged blocking; don't wait for user complaints.
9. Use Application Continuity for OLTP that must survive kill.
10. For queues, `SELECT ... FOR UPDATE SKIP LOCKED` to avoid unnecessary blocking.

## Interview Questions

1. **Q:** How do you find blocking sessions?
   **A:** `SELECT sid, blocking_session, event FROM v$session WHERE blocking_session IS NOT NULL;`.

2. **Q:** Follow the chain — why?
   **A:** Killing an intermediate blocker doesn't help if it's itself blocked. Kill the ultimate blocker.

3. **Q:** How to kill?
   **A:** `ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;` from Oracle; `kill -9 <spid>` at OS if needed.

4. **Q:** Session stays KILLED — why?
   **A:** PMON waiting for OS process to exit. `kill -9 <spid>`.

5. **Q:** `DISCONNECT SESSION` vs `KILL SESSION`?
   **A:** DISCONNECT drops the network; KILL marks the session and rolls back.

6. **Q:** How do you prevent long blockers?
   **A:** Application-level commit discipline, `IDLE_TIME` in profile, Resource Manager `SWITCH_TIME`.

7. **Q:** RAC — how to know which node holds the blocker?
   **A:** `V$SESSION.BLOCKING_INSTANCE`.

## References

- Oracle Database Concepts 19c — Data Concurrency
- MOS Doc ID 62354.1 — TX Enqueue
- MOS Doc ID 1020188.6 — Kill Session
- Runbook: [Blocking Sessions](../27-runbooks/blocking-sessions.md)
