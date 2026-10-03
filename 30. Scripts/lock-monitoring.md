# Lock Monitoring Scripts

## Current Blocking Chain

```sql
COLUMN tree FORMAT A50
COLUMN username FORMAT A15
COLUMN event FORMAT A30 TRUNC

SELECT LEVEL lvl,
       LPAD(' ', LEVEL*2, '.') || sid AS tree,
       serial#, username, event, seconds_in_wait,
       sql_id, blocking_session, final_blocking_session
FROM   v$session
WHERE  status = 'ACTIVE'
CONNECT BY PRIOR sid = blocking_session
START  WITH blocking_session IS NULL
       AND sid IN (SELECT DISTINCT blocking_session FROM v$session
                   WHERE blocking_session IS NOT NULL)
ORDER  SIBLINGS BY sid;
```

## Blockers Only

```sql
SELECT s.sid, s.serial#, s.username, s.status, s.last_call_et/60 mins,
       s.event, s.sql_id, s.machine, s.program
FROM   v$session s
WHERE  s.sid IN (SELECT DISTINCT blocking_session
                 FROM v$session WHERE blocking_session IS NOT NULL);
```

## Waiters and What They're Blocked On

```sql
SELECT w.sid waiter_sid, w.username waiter, w.event,
       w.seconds_in_wait wait_s,
       b.sid blocker_sid, b.username blocker,
       w.sql_id waiter_sql, b.sql_id blocker_sql
FROM   v$session w JOIN v$session b ON b.sid = w.blocking_session
WHERE  w.blocking_session IS NOT NULL;
```

## Locked Objects

```sql
COLUMN owner FORMAT A15
COLUMN object_name FORMAT A30

SELECT s.sid, s.serial#, s.username, s.machine,
       o.owner, o.object_name, o.object_type,
       DECODE(lo.locked_mode,
              2,'RS',3,'RX',4,'S',5,'SRX',6,'X', lo.locked_mode) mode_
FROM   v$locked_object lo
JOIN   dba_objects   o ON o.object_id = lo.object_id
JOIN   v$session     s ON s.sid       = lo.session_id
ORDER  BY s.sid;
```

## Recent Deadlocks

```sql
SELECT originating_timestamp, host_id, message_text
FROM   v$diag_alert_ext
WHERE  message_text LIKE '%ORA-00060%'
   AND originating_timestamp > SYSDATE - 7
ORDER  BY originating_timestamp DESC;

-- Deadlock trace files
SELECT trace_filename, modify_time
FROM   v$diag_trace_file
WHERE  trace_filename LIKE '%deadlock%'
   AND modify_time > SYSDATE - 7
ORDER  BY modify_time DESC;
```

## Enqueue Waits (Type / Count / Time)

```sql
SELECT event, wait_class, total_waits,
       ROUND(time_waited/100,1) secs,
       ROUND(average_wait,3) cs
FROM   v$system_event
WHERE  event LIKE 'enq:%'
ORDER  BY time_waited DESC;
```

## TX Locks — Rollback Slot Detail

```sql
SELECT s.sid, l.type, l.id1, l.id2,
       TRUNC(l.id1/POWER(2,16)) rbs,
       BITAND(l.id1, TO_NUMBER('ffff','xxxx'))+0 slot,
       l.id2 wrap#,
       DECODE(l.lmode, 6,'X',5,'SRX',4,'S',3,'RX',2,'RS',1,'Null', l.lmode) lmode
FROM   v$lock l JOIN v$session s ON s.sid = l.sid
WHERE  l.type = 'TX' AND l.lmode > 0;
```

## TM Lock Contention Sources

```sql
-- Missing FK indexes cause TM cascades
SELECT c.owner, c.constraint_name, c.table_name, cc.column_name,
       r.table_name r_table_name
FROM   dba_constraints c
JOIN   dba_cons_columns cc ON cc.owner=c.owner AND cc.constraint_name=c.constraint_name
JOIN   dba_constraints r ON r.owner=c.r_owner AND r.constraint_name=c.r_constraint_name
WHERE  c.constraint_type='R'
   AND NOT EXISTS (
     SELECT 1 FROM dba_ind_columns ic
     WHERE ic.table_owner = c.owner AND ic.table_name = c.table_name
       AND ic.column_name = cc.column_name AND ic.column_position = 1);
```

## Related

- [V$LOCK](../25-reference/v-views/v-lock.md).
- [Blocking Sessions runbook](../27-runbooks/blocking-sessions.md).
- [Deadlocks](../13-locking/deadlocks.md).
