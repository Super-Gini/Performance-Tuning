# V$SESSION

## Purpose

One row per currently-connected session (user + background). The **single most useful DBA view** — starting point for "what is happening right now".

## Key Columns

| Column                                               | Meaning                                                      |
| ---------------------------------------------------- | ------------------------------------------------------------ |
| `SID`                                                | Session ID.                                                  |
| `SERIAL#`                                            | Serial — with SID uniquely identifies a session across time. |
| `PADDR`                                              | Foreign key to `V$PROCESS.ADDR`.                             |
| `USERNAME`                                           | Oracle user (NULL = background).                             |
| `OSUSER`                                             | OS user on the client.                                       |
| `MACHINE`                                            | Client hostname.                                             |
| `PROGRAM`                                            | Client program.                                              |
| `MODULE`, `ACTION`, `CLIENT_INFO`                    | Set by application via `DBMS_APPLICATION_INFO`.              |
| `CLIENT_IDENTIFIER`                                  | End-user identifier.                                         |
| `TYPE`                                               | `USER` / `BACKGROUND`.                                       |
| `STATUS`                                             | `ACTIVE` / `INACTIVE` / `KILLED` / `SNIPED`.                 |
| `STATE`                                              | `WAITING` / `WAITED KNOWN TIME` / `WAITED SHORT TIME`.       |
| `EVENT`                                              | Current wait event (or last, if not currently waiting).      |
| `WAIT_CLASS`                                         | Grouping: `User I/O`, `Concurrency`, `Idle`, etc.            |
| `SECONDS_IN_WAIT`                                    | How long in this event.                                      |
| `P1`, `P2`, `P3`                                     | Wait event parameters.                                       |
| `SQL_ID`                                             | Current SQL.                                                 |
| `PREV_SQL_ID`                                        | Previous SQL.                                                |
| `SQL_CHILD_NUMBER`                                   | Child cursor.                                                |
| `BLOCKING_SESSION`                                   | SID of the blocker (if `BLOCKING_SESSION_STATUS='VALID'`).   |
| `BLOCKING_INSTANCE`                                  | Instance # of blocker in RAC.                                |
| `ROW_WAIT_OBJ#`                                      | Object being read/locked.                                    |
| `ROW_WAIT_FILE#`, `ROW_WAIT_BLOCK#`, `ROW_WAIT_ROW#` | Precise row identification.                                  |
| `LOGON_TIME`                                         | Connection time.                                             |
| `LAST_CALL_ET`                                       | Elapsed since last call.                                     |
| `CON_ID`                                             | Container (PDB) ID.                                          |
| `SERVICE_NAME`                                       | Registered service the session belongs to.                   |
| `SCHEMANAME`                                         | Currently-parsed schema.                                     |

## Common Queries

```sql
-- Active sessions right now
SELECT sid, serial#, username, program, event, sql_id,
       blocking_session, seconds_in_wait
FROM   v$session
WHERE  status = 'ACTIVE' AND type = 'USER'
ORDER  BY seconds_in_wait DESC NULLS LAST;

-- Blocking chain (recursive)
SELECT   LEVEL lvl, sid, blocking_session, event, seconds_in_wait, sql_id
FROM     v$session
WHERE    blocking_session IS NOT NULL
CONNECT  BY PRIOR sid = blocking_session
START    WITH blocking_session IS NULL;

-- Kill a session
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;

-- Who's connected as which app user (via client_identifier)
SELECT client_identifier, COUNT(*) sessions
FROM   v$session
WHERE  type='USER'
GROUP  BY client_identifier
ORDER  BY 2 DESC;
```

## Related

- `V$PROCESS` — join via `PADDR`.
- `V$SESSION_WAIT` — same wait info; `V$SESSION` supplants in 10g+.
- `V$SESSION_LONGOPS` — long-running operations.
- `GV$SESSION` — cluster-wide.

## References

- Oracle Database Reference 19c — `V$SESSION`
