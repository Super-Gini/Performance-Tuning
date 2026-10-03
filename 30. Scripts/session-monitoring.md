# Session Monitoring Scripts

## Active Sessions Right Now

```sql
COLUMN username FORMAT A15
COLUMN program  FORMAT A25 TRUNC
COLUMN event    FORMAT A32 TRUNC
COLUMN sql_id   FORMAT A15

SELECT s.sid, s.serial#, s.username,
       s.status, s.machine,
       s.program, s.event, s.wait_class,
       s.seconds_in_wait secs_wait,
       s.sql_id,
       s.blocking_session block,
       s.last_call_et/60 mins_in_call
FROM   v$session s
WHERE  s.type = 'USER' AND s.status = 'ACTIVE'
ORDER  BY s.seconds_in_wait DESC NULLS LAST
FETCH  FIRST 30 ROWS ONLY;
```

## Session Detail by SID

```sql
COLUMN name  FORMAT A32
COLUMN value FORMAT A60 WRAP

SELECT sid, serial#, username, osuser, machine, program, module, action,
       client_identifier, service_name, resource_consumer_group,
       status, state, event, wait_class, seconds_in_wait,
       sql_id, prev_sql_id, sql_child_number, blocking_session,
       logon_time, last_call_et/60 mins_in_call
FROM   v$session
WHERE  sid = &target_sid;

-- OS side
SELECT p.spid os_pid, p.pname, p.tracefile,
       ROUND(p.pga_used_mem/1024/1024,2) used_mb,
       ROUND(p.pga_max_mem/1024/1024,2)  max_mb
FROM   v$session s JOIN v$process p ON p.addr = s.paddr
WHERE  s.sid = &target_sid;
```

## Session Counts by Program / User

```sql
SELECT COALESCE(program,'--') program, status, COUNT(*)
FROM   v$session
WHERE  type = 'USER'
GROUP  BY program, status
ORDER  BY 3 DESC;

SELECT username, status, COUNT(*)
FROM   v$session
WHERE  type = 'USER'
GROUP  BY username, status
ORDER  BY 3 DESC;
```

## Idle > N Minutes

```sql
SELECT sid, serial#, username, program, machine,
       ROUND(last_call_et/60,1) idle_mins
FROM   v$session
WHERE  type = 'USER' AND status = 'INACTIVE'
   AND last_call_et > &idle_seconds
ORDER  BY last_call_et DESC;
```

## Kill Session

```sql
-- Confirm target first!
SELECT sid, serial#, username, status FROM v$session WHERE sid = &target_sid;

ALTER SYSTEM KILL SESSION '&target_sid,&target_serial' IMMEDIATE;

-- On RAC (across instances)
ALTER SYSTEM KILL SESSION '&target_sid,&target_serial,@&target_inst' IMMEDIATE;
```

## Long-Running Sessions

```sql
SELECT sid, serial#, username, sql_id, event,
       ROUND(last_call_et/60,1) mins_in_call
FROM   v$session
WHERE  status = 'ACTIVE' AND type = 'USER'
   AND last_call_et > 3600
ORDER  BY last_call_et DESC;
```

## Session's Current SQL Text and Plan

```sql
SELECT sql_id, prev_sql_id, sql_child_number FROM v$session WHERE sid = &target_sid;

SELECT sql_fulltext FROM v$sql WHERE sql_id = '&sql_id' AND child_number = &child;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', &child, 'ALL ALLSTATS LAST'));
```

## Session's Wait History (last 10 waits)

```sql
SELECT sid, seq#, event, p1, p2, p3, wait_time
FROM   v$session_wait_history
WHERE  sid = &target_sid
ORDER  BY seq#;
```

## Session's Cumulative Wait Time (per event)

```sql
SELECT event, wait_class, total_waits,
       ROUND(time_waited/100,1) secs
FROM   v$session_event
WHERE  sid = &target_sid
   AND wait_class <> 'Idle'
ORDER  BY time_waited DESC;
```

## Related

- [V$SESSION](../25-reference/v-views/v-session.md).
- [Blocking Sessions runbook](../27-runbooks/blocking-sessions.md).
