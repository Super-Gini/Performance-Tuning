# V$ARCHIVED_LOG

## Purpose

History of archived redo logs — one row per successful archive.

## Key Columns

| Column              | Meaning                                                   |
| ------------------- | --------------------------------------------------------- |
| `SEQUENCE#`         | Redo sequence.                                            |
| `THREAD#`           | Redo thread.                                              |
| `NAME`              | Filesystem/ASM path.                                      |
| `DEST_ID`           | `LOG_ARCHIVE_DEST_n`.                                     |
| `FIRST_CHANGE#`     | First SCN in this log.                                    |
| `FIRST_TIME`        | Time of first SCN.                                        |
| `NEXT_CHANGE#`      | Last SCN.                                                 |
| `NEXT_TIME`         | Time of last SCN.                                         |
| `BLOCKS`            | Redo blocks archived.                                     |
| `BLOCK_SIZE`        | Bytes per redo block.                                     |
| `STATUS`            | `A` = available, `D` = deleted, `X` = expired.            |
| `ARCHIVED`          | `YES`/`NO`.                                               |
| `APPLIED`           | For standby — was this applied? `NO`, `YES`, `IN-MEMORY`. |
| `DELETED`           | `YES`/`NO`.                                               |
| `COMPLETION_TIME`   | When archive finished.                                    |
| `RESETLOGS_CHANGE#` | Incarnation marker.                                       |
| `RESETLOGS_TIME`    | Incarnation marker time.                                  |
| `BACKUP_COUNT`      | Times backed up by RMAN.                                  |
| `CON_ID`            | Container.                                                |

## Common Queries

```sql
-- Recent archives
SELECT   thread#, sequence#, first_time, completion_time,
         ROUND(blocks*block_size/1024/1024,2) mb,
         name
FROM     v$archived_log
WHERE    completion_time > SYSDATE - 1
ORDER BY completion_time DESC
FETCH FIRST 20 ROWS ONLY;

-- Standby lag (applied vs archived)
SELECT   MAX(sequence#) as max_arc,
         MAX(CASE WHEN applied = 'YES' THEN sequence# END) as max_applied,
         MAX(sequence#) - MAX(CASE WHEN applied = 'YES' THEN sequence# END) as gap
FROM     v$archived_log
WHERE    thread# = 1;

-- Gaps
SELECT   thread#, sequence#
FROM     v$archived_log
WHERE    (thread#, sequence#) NOT IN
         (SELECT thread#, sequence#-1 FROM v$archived_log
          WHERE sequence# > 1)
ORDER BY thread#, sequence#;

-- Space used by archives
SELECT   TO_CHAR(completion_time,'YYYY-MM-DD') day,
         ROUND(SUM(blocks*block_size)/1024/1024/1024, 2) gb
FROM     v$archived_log
WHERE    completion_time > SYSDATE - 30 AND deleted = 'NO'
GROUP BY TO_CHAR(completion_time,'YYYY-MM-DD')
ORDER BY 1 DESC;
```

## References

- Oracle Database Reference 19c — `V$ARCHIVED_LOG`
- [Archive Logs](../../04-storage/archive-logs.md)
