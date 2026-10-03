# Network Wait Events

## Overview

**Network** class waits — session waiting on the client, or the client waiting on Oracle net layer, or a database link waiting on a remote database.

## Events

### `SQL*Net message from client`

**Meaning**: Session is idle — waiting for the client to send another SQL. This is the **`INACTIVE`** session's dominant event and normally means nothing is wrong. It's technically classified as `Idle` but often surfaces in tuning reports.

**When it matters**: Very short bursts of `SQL*Net message from client` between round-trips is normal. Long ones = idle client. Frequent short ones between many small SQLs = "chatty app" → consider bulk fetching.

### `SQL*Net message to client`

**Meaning**: Server is writing data to the client's socket, but the client isn't reading fast enough (TCP buffer full).

**Fix**: Bigger `SDU_SIZE`, faster client, better network. Rare in modern setups.

### `SQL*Net more data from client`

Client is sending a multi-part message (large IN list, LOB write). Similar interpretation.

### `SQL*Net more data to client`

Server sending multi-part message (large result, LOB read). If long: network is slow or TCP window is small.

### `SQL*Net break/reset to client`

Client interrupted the session — Ctrl-C, connection reset.

### `SQL*Net message from dblink`

Session is waiting on a **remote database** through a DB link. This one **is** significant. Long waits mean the remote DB is slow or the network to it is.

**Diagnose**:

```sql
-- Session using a dblink
SELECT s.sid, s.serial#, s.event, s.sql_id, l.name dblink
FROM   v$session s LEFT JOIN v$dblink l ON l.owner_id = s.sid
WHERE  s.event LIKE 'SQL*Net message from dblink%';

-- Which SQL uses dblinks
SELECT sql_id, sql_text FROM v$sql WHERE sql_text LIKE '%@%';
```

### `TCP Socket (KGAS)`

Low-level TCP wait — Oracle net layer waiting on kernel socket ops.

## SDU Tuning

`SDU_SIZE` = Session Data Unit — max bytes per SQL\*Net packet. Default 8192; can raise to 65535 for fast networks with big rows.

```
-- sqlnet.ora on both sides
DEFAULT_SDU_SIZE=32767

-- listener.ora
LISTENER=(DESCRIPTION_LIST=(DESCRIPTION=(SDU=32767)(...)))

-- tnsnames.ora
PRD=(DESCRIPTION=(SDU=32767)(ADDRESS=...))
```

## Diagnostic Query

```sql
SELECT   event, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(average_wait,3) cs
FROM     v$system_event
WHERE    event LIKE 'SQL*Net%' AND time_waited > 0
ORDER BY time_waited DESC;
```

## References

- Oracle Database Net Services Reference 19c
- MOS Doc ID 33507.1 — SQL\*Net wait events
