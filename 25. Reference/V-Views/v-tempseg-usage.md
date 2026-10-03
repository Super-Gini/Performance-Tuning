# V$TEMPSEG_USAGE (V$SORT_USAGE in older versions)

## Purpose

Temporary segments in use — sort area, hash join spills, temp tables, index rebuilds. One row per active temp segment per session.

## Key Columns

| Column              | Meaning                                      |
| ------------------- | -------------------------------------------- |
| `USERNAME`          | Oracle user.                                 |
| `SID / SERIAL#`     | Session.                                     |
| `TABLESPACE`        | Temp tablespace.                             |
| `CONTENTS`          | `TEMPORARY`, `PERMANENT`.                    |
| `SEGTYPE`           | `SORT`, `HASH`, `DATA`, `INDEX`, `LOB_DATA`. |
| `SEGFILE#`          | File number.                                 |
| `SEGBLK#`           | First block.                                 |
| `EXTENTS`           | Extents allocated.                           |
| `BLOCKS`            | Blocks allocated.                            |
| `SEGRFNO#`          | Relative file#.                              |
| `SQL_ID`            | SQL causing the temp allocation.             |
| `SQLADDR / SQLHASH` | Legacy identifiers.                          |

## Common Queries

```sql
-- Who's eating TEMP right now
SELECT   s.sid, s.username, s.machine, s.program, tu.segtype,
         ROUND(tu.blocks * (SELECT block_size FROM dba_tablespaces
                            WHERE tablespace_name = tu.tablespace)/1024/1024, 2) mb,
         tu.sql_id
FROM     v$tempseg_usage tu JOIN v$session s
             ON s.sid = tu.session_addr    -- some 19c builds use session_id
ORDER BY mb DESC;

-- Alternative modern join (via session_num)
SELECT   s.sid, s.username, ss.username tem_user,
         ROUND(ss.blocks * ts.block_size/1024/1024,2) mb, ss.sql_id
FROM     v$sort_usage ss JOIN v$session s ON s.saddr = ss.session_addr
JOIN     dba_tablespaces ts ON ts.tablespace_name = ss.tablespace
ORDER BY mb DESC;

-- Temp file utilization
SELECT   tablespace_name, ROUND(bytes_used/1024/1024,2) used_mb,
         ROUND(bytes_free/1024/1024,2) free_mb
FROM     v$temp_space_header;

-- Session that generated the biggest temp usage in the last hour (ASH)
SELECT   session_id, sql_id, SUM(temp_space_allocated) bytes_allocated
FROM     v$active_session_history
WHERE    sample_time > SYSDATE - 1/24 AND temp_space_allocated > 0
GROUP BY session_id, sql_id
ORDER BY bytes_allocated DESC
FETCH FIRST 10 ROWS ONLY;
```

## References

- Oracle Database Reference 19c — `V$TEMPSEG_USAGE`
- [Tempfiles](../../04-storage/tempfiles.md)
