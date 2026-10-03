# V$RMAN_STATUS

## Purpose

Status of RMAN operations — running, completed, failed. Same info as `LIST` output in RMAN.

## Key Columns

| Column             | Meaning                                                  |
| ------------------ | -------------------------------------------------------- |
| `SID / RECID`      | Identifiers.                                             |
| `OPERATION`        | `BACKUP`, `RESTORE`, `RECOVER`, `DUPLICATE`, etc.        |
| `OBJECT_TYPE`      | e.g., `DB FULL`, `ARCHIVELOG`, `DATAFILE`.               |
| `STATUS`           | `RUNNING`, `RUNNING WITH ERRORS`, `COMPLETED`, `FAILED`. |
| `MBYTES_PROCESSED` | Bytes done.                                              |
| `START_TIME`       | Start.                                                   |
| `END_TIME`         | End (or NULL if running).                                |
| `INPUT_BYTES`      | Read bytes.                                              |
| `OUTPUT_BYTES`     | Written bytes.                                           |
| `INPUT_TYPE`       | e.g., `DB INCR`, `ARCHIVELOG`.                           |

## Common Queries

```sql
-- Recent backups
SELECT   session_key, input_type, status,
         ROUND(input_bytes/1024/1024/1024,2) in_gb,
         ROUND(output_bytes/1024/1024/1024,2) out_gb,
         start_time, end_time,
         ROUND((end_time-start_time)*24*60,1) mins
FROM     v$rman_backup_job_details
WHERE    start_time > SYSDATE - 7
ORDER BY start_time DESC;

-- Currently running
SELECT session_key, operation, object_type, status,
       mbytes_processed, start_time
FROM   v$rman_status
WHERE  status LIKE 'RUNNING%'
ORDER  BY start_time;

-- Errors
SELECT session_key, operation, object_type, status, start_time
FROM   v$rman_status
WHERE  status LIKE '%ERROR%' OR status = 'FAILED'
ORDER  BY start_time DESC
FETCH  FIRST 20 ROWS ONLY;

-- Backup throughput
SELECT   TO_CHAR(start_time,'YYYY-MM-DD') day,
         ROUND(SUM(input_bytes)/1024/1024/1024,2) input_gb,
         ROUND(SUM(output_bytes)/1024/1024/1024,2) output_gb,
         SUM(EXTRACT(HOUR FROM (end_time-start_time)*24 HOUR)) hours
FROM     v$rman_backup_job_details
WHERE    start_time > SYSDATE - 30
GROUP BY TO_CHAR(start_time,'YYYY-MM-DD')
ORDER BY 1 DESC;
```

## Related Views

- `V$RMAN_BACKUP_JOB_DETAILS` — job-level summary (preferred).
- `V$RMAN_BACKUP_SUBJOB_DETAILS`.
- `V$BACKUP_SET` — per backup set.
- `V$BACKUP_PIECE` — per file.

## References

- Oracle Database Backup and Recovery Reference 19c — `V$RMAN_STATUS`
