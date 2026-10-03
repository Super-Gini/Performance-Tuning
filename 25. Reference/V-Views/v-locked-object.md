# V$LOCKED_OBJECT

## Purpose

Objects currently under DML lock — joins nicely with `DBA_OBJECTS` and `V$SESSION`.

## Key Columns

| Column            | Meaning                          |
| ----------------- | -------------------------------- |
| `XIDUSN`          | Undo segment ID of transaction.  |
| `XIDSLOT`         | Rollback slot.                   |
| `XIDSQN`          | Wrap#.                           |
| `OBJECT_ID`       | Object ID (joins `DBA_OBJECTS`). |
| `SESSION_ID`      | Session.                         |
| `ORACLE_USERNAME` | Oracle user.                     |
| `OS_USER_NAME`    | OS user.                         |
| `PROCESS`         | OS process.                      |
| `LOCKED_MODE`     | Same codes as `V$LOCK.LMODE`.    |

## Common Queries

```sql
-- Who has locks on what
SELECT   lo.session_id, s.username, s.machine, s.program,
         o.owner, o.object_name, o.object_type, lo.locked_mode
FROM     v$locked_object lo
JOIN     dba_objects   o ON o.object_id = lo.object_id
JOIN     v$session     s ON s.sid       = lo.session_id
ORDER BY lo.session_id;

-- Table-level lock hotlist
SELECT   o.owner, o.object_name, COUNT(*) sessions_locking
FROM     v$locked_object lo JOIN dba_objects o ON o.object_id = lo.object_id
GROUP BY o.owner, o.object_name
ORDER BY 3 DESC;
```

## References

- Oracle Database Reference 19c — `V$LOCKED_OBJECT`
- [TM Locks](../../13-locking/tm-locks.md)
