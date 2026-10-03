# V$RECOVERY_FILE_DEST

## Purpose

State of the Fast Recovery Area (FRA) — how much space is used, reclaimable, and available.

## Key Columns

| Column              | Meaning                               |
| ------------------- | ------------------------------------- |
| `NAME`              | FRA path (`DB_RECOVERY_FILE_DEST`).   |
| `SPACE_LIMIT`       | `DB_RECOVERY_FILE_DEST_SIZE` — bytes. |
| `SPACE_USED`        | Bytes in use.                         |
| `SPACE_RECLAIMABLE` | Bytes eligible for auto-deletion.     |
| `NUMBER_OF_FILES`   | File count.                           |

## Common Queries

```sql
-- Simple percent-used
SELECT   name,
         ROUND(space_used/1024/1024/1024,2) used_gb,
         ROUND(space_limit/1024/1024/1024,2) limit_gb,
         ROUND(space_reclaimable/1024/1024/1024,2) reclaim_gb,
         ROUND(space_used*100/space_limit,2) pct_used
FROM     v$recovery_file_dest;

-- By file type
SELECT   file_type,
         percent_space_used, percent_space_reclaimable,
         number_of_files
FROM     v$flash_recovery_area_usage;
```

`V$FLASH_RECOVERY_AREA_USAGE`:

| FILE_TYPE                 |
| ------------------------- |
| `CONTROL FILE`            |
| `REDO LOG`                |
| `ARCHIVED LOG`            |
| `BACKUP PIECE`            |
| `IMAGE COPY`              |
| `FLASHBACK LOG`           |
| `FOREIGN ARCHIVED LOG`    |
| `AUXILIARY DATAFILE COPY` |

## Alerting

Consider `PCT_USED > 85%` warning, `> 95%` critical. FRA fills quickly during Data Guard resync or large backup.

## References

- Oracle Database Backup and Recovery Guide 19c — FRA
