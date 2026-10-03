# DBA_DATA_FILES

## Purpose

All permanent datafiles.

## Key Columns

| Column               | Meaning                                         |
| -------------------- | ----------------------------------------------- |
| `FILE_NAME`          | OS path or ASM name.                            |
| `FILE_ID`            | Absolute file# (unique DB-wide).                |
| `RELATIVE_FNO`       | Per-tablespace file#.                           |
| `TABLESPACE_NAME`    | TS this file belongs to.                        |
| `BYTES`              | Current size.                                   |
| `BLOCKS`             | In blocks.                                      |
| `STATUS`             | `AVAILABLE`, `INVALID`.                         |
| `AUTOEXTENSIBLE`     | `YES`/`NO`.                                     |
| `MAXBYTES`           | Autoextend upper bound (0 = unbounded / unset). |
| `MAXBLOCKS`          | Same in blocks.                                 |
| `INCREMENT_BY`       | Autoextend increment in blocks.                 |
| `USER_BYTES`         | Usable bytes (excludes header).                 |
| `ONLINE_STATUS`      | `ONLINE`, `OFFLINE`, `SYSOFF`, `RECOVER`.       |
| `LOST_WRITE_PROTECT` | Lost-write protection setting.                  |
| `CON_ID`             | Container.                                      |

## Common Queries

```sql
-- All datafiles with autoextend info
SELECT tablespace_name, file_name,
       ROUND(bytes/1024/1024/1024,2) size_gb,
       ROUND(maxbytes/1024/1024/1024,2) max_gb,
       autoextensible,
       ROUND(increment_by * (SELECT block_size FROM dba_tablespaces t
                             WHERE t.tablespace_name = df.tablespace_name)
             /1024/1024,2) inc_mb
FROM   dba_data_files df
ORDER  BY 1, 2;

-- Files near their max
SELECT tablespace_name, file_name,
       ROUND(bytes*100/maxbytes,2) pct_of_max,
       ROUND(bytes/1024/1024/1024,2) size_gb,
       ROUND(maxbytes/1024/1024/1024,2) max_gb
FROM   dba_data_files
WHERE  autoextensible='YES' AND maxbytes > 0
   AND bytes*100/maxbytes > 80
ORDER  BY 3 DESC;

-- Non-autoextend files (surprise!)
SELECT tablespace_name, file_name FROM dba_data_files WHERE autoextensible='NO';
```

## Related

- `DBA_TEMP_FILES` — temp files.
- `V$DATAFILE` — from control file, includes SCN state.
- `V$DATAFILE_HEADER` — read from file header on disk.

## References

- Oracle Database Reference 19c — `DBA_DATA_FILES`
- [Datafiles](../../04-storage/datafiles.md)
