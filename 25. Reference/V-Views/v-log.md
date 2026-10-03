# V$LOG

## Purpose

Online redo log groups' state — one row per group.

## Key Columns

| Column          | Meaning                                                                    |
| --------------- | -------------------------------------------------------------------------- |
| `GROUP#`        | Redo log group number.                                                     |
| `THREAD#`       | Redo thread (1 on non-RAC; = instance# on RAC).                            |
| `SEQUENCE#`     | Current sequence# in this group (0 if never used).                         |
| `BYTES`         | Group size.                                                                |
| `BLOCKSIZE`     | Redo block size (512 or 4096).                                             |
| `MEMBERS`       | Number of members (multiplexed copies).                                    |
| `ARCHIVED`      | `YES`/`NO`.                                                                |
| `STATUS`        | `UNUSED`, `INACTIVE`, `ACTIVE`, `CURRENT`, `CLEARING`, `CLEARING_CURRENT`. |
| `FIRST_CHANGE#` | First SCN in this group.                                                   |
| `FIRST_TIME`    | Wall time of first SCN.                                                    |
| `NEXT_CHANGE#`  | Highest SCN (`CURRENT` = infinity marker).                                 |
| `NEXT_TIME`     | Wall time of next SCN.                                                     |
| `CON_ID`        | Container ID.                                                              |

## Common Queries

```sql
-- Layout at a glance
SELECT group#, thread#, sequence#, bytes/1024/1024 mb,
       members, archived, status
FROM   v$log
ORDER  BY thread#, group#;

-- Members
SELECT l.group#, l.status, m.member
FROM   v$log l JOIN v$logfile m USING (group#)
ORDER  BY l.group#, m.member;

-- Log switches in the last day
SELECT TO_CHAR(first_time,'YYYY-MM-DD HH24') hr, COUNT(*) switches
FROM   v$log_history
WHERE  first_time > SYSDATE - 1
GROUP  BY TO_CHAR(first_time,'YYYY-MM-DD HH24')
ORDER  BY 1;

-- Redo throughput
SELECT   TO_CHAR(first_time,'YYYY-MM-DD HH24') hr,
         COUNT(*) switches,
         ROUND(SUM(blocks * block_size)/1024/1024/1024, 2) gb
FROM     v$archived_log
WHERE    first_time > SYSDATE - 1
GROUP BY TO_CHAR(first_time,'YYYY-MM-DD HH24')
ORDER BY 1;
```

## Status Meanings

- `CURRENT` — Being written by LGWR now.
- `ACTIVE` — Full but still needed for instance recovery.
- `INACTIVE` — Full, checkpointed; safe to reuse.
- `UNUSED` — Freshly added, never used.
- `CLEARING` — `ALTER DATABASE CLEAR LOGFILE` in progress.

## References

- Oracle Database Reference 19c — `V$LOG`
- [Redo Logs](../../04-storage/redo-logs.md)
