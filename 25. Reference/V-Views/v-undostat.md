# V$UNDOSTAT

## Purpose

Time-series statistics about undo usage — one row per 10-minute interval. Perfect for sizing undo tablespace and undo retention.

## Key Columns

| Column                | Meaning                            |
| --------------------- | ---------------------------------- |
| `BEGIN_TIME/END_TIME` | 10-min bucket.                     |
| `UNDOTSN`             | Undo tablespace TS#.               |
| `UNDOBLKS`            | Undo blocks consumed in bucket.    |
| `TXNCOUNT`            | Number of transactions.            |
| `MAXQUERYLEN`         | Longest query duration (seconds).  |
| `MAXQUERYSQLID`       | SQL_ID of that query.              |
| `MAXCONCURRENCY`      | Max concurrent transactions.       |
| `UNXPBLKREUCNT`       | Unexpired blocks reused.           |
| `UNXPBLKRELCNT`       | Unexpired blocks released.         |
| `EXPBLKRELCNT`        | Expired blocks released.           |
| `EXPBLKREUCNT`        | Expired blocks reused.             |
| `SSOLDERRCNT`         | ORA-01555 count.                   |
| `NOSPACEERRCNT`       | ORA-30036 (undo full) count.       |
| `ACTIVEBLKS`          | Active blocks at end of bucket.    |
| `UNEXPIREDBLKS`       | Unexpired.                         |
| `EXPIREDBLKS`         | Expired.                           |
| `TUNED_UNDORETENTION` | Oracle's auto-tuned retention (s). |

## Common Queries

```sql
-- Recent activity
SELECT   TO_CHAR(begin_time,'YYYY-MM-DD HH24:MI') t,
         undoblks, txncount, maxquerylen, maxconcurrency,
         ssolderrcnt ora1555, nospaceerrcnt ora30036,
         tuned_undoretention
FROM     v$undostat
ORDER BY begin_time DESC
FETCH FIRST 20 ROWS ONLY;

-- Recommend undo tablespace size
SELECT   MAX(undoblks) max_undoblks,
         MAX(undoblks) * (SELECT value FROM v$parameter WHERE name='db_block_size')/1024/1024 max_mb_per_10min,
         MAX(tuned_undoretention) retention_s
FROM     v$undostat
WHERE    begin_time > SYSDATE - 7;

-- Any snapshot-too-old in the last week?
SELECT SUM(ssolderrcnt) FROM v$undostat WHERE begin_time > SYSDATE - 7;
```

## References

- Oracle Database Reference 19c — `V$UNDOSTAT`
- [Undo Management](../../05-undo/undo-management.md)
- [ORA-01555](../../26-errors/ora-01555.md)
