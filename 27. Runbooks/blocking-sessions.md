# Runbook: Blocking Sessions

## Symptom

- App reports "hanging" queries.
- Wait event: `enq: TX - row lock contention`, `enq: TM - contention`.
- ADDM / OEM: blocking chain visible.

## Triage

```sql
-- Blocking chain
SELECT   LEVEL lvl,
         LPAD(' ', LEVEL*2, ' ') || sid AS session_tree,
         sid, serial#, username, event, seconds_in_wait,
         sql_id, blocking_session
FROM     v$session
WHERE    blocking_session IS NOT NULL OR sid IN (
             SELECT blocking_session FROM v$session WHERE blocking_session IS NOT NULL)
CONNECT  BY PRIOR sid = blocking_session
START    WITH blocking_session IS NULL AND sid IN (
             SELECT blocking_session FROM v$session WHERE blocking_session IS NOT NULL)
ORDER    SIBLINGS BY sid;

-- Details on the root blocker
SELECT s.sid, s.serial#, s.username, s.status, s.event,
       s.last_call_et/60 min_since_call,
       s.sql_id, s.program, s.machine, s.osuser
FROM   v$session s
WHERE  s.sid IN (SELECT DISTINCT final_blocking_session FROM v$session
                 WHERE final_blocking_session IS NOT NULL);

-- Locked objects
SELECT s.sid, o.owner, o.object_name, o.object_type, lo.locked_mode
FROM   v$locked_object lo JOIN dba_objects o ON o.object_id = lo.object_id
JOIN   v$session s ON s.sid = lo.session_id
WHERE  s.sid IN (SELECT blocking_session FROM v$session WHERE blocking_session IS NOT NULL);
```

## Actions

### 1. Contact the blocker (if human)

If the blocking session's `PROGRAM` is `sqlplus`, `Toad`, etc. — reach the user. They may have an open transaction.

### 2. Kill the blocker (if safe)

```sql
-- With serial# from v$session
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;

-- In RAC (across instances)
ALTER SYSTEM KILL SESSION 'sid,serial#,@inst_id' IMMEDIATE;
```

`IMMEDIATE` = rollback + close session; will complete when the session's cleanup finishes.

### 3. If session won't die

```sql
-- Get OS PID
SELECT s.sid, p.spid FROM v$session s JOIN v$process p ON p.addr = s.paddr
WHERE  s.sid = &blocker_sid;

-- OS-level (last resort)
!kill -9 <spid>
```

### 4. Prevent recurrence in the current window

If it's a runaway user query:

```sql
-- Resource Manager cancel_sql after N minutes
ALTER SYSTEM SET resource_manager_plan = 'CANCEL_LONG_QUERIES';
```

## Verification

```sql
-- Confirm chain cleared
SELECT COUNT(*) FROM v$session WHERE blocking_session IS NOT NULL;
```

Application: transactions resume.

## Post-Mortem

- Who was the root blocker? Human or app process?
- What was the blocked SQL? Was it critical?
- Missing FK index causing TM cascade?
- Application transaction pattern issue?

## Prevention

- Application transaction discipline (short TX, commit promptly).
- FK indexes always.
- Long-running "SELECT for review" sessions blocked by profile IDLE_TIME.
- Resource Manager plan with `SWITCH_GROUP='CANCEL_SQL'` for reports.
- Monitoring for blocking chains > 5 minutes.

## Related

- [Blocking Sessions](../13-locking/blocking-sessions.md).
- [Deadlocks](../13-locking/deadlocks.md).
- [ORA-00060](../26-errors/ora-00060.md).
