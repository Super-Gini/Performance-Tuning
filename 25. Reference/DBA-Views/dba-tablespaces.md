# DBA_TABLESPACES

## Purpose

All tablespaces in the database.

## Key Columns

| Column                      | Meaning                                    |
| --------------------------- | ------------------------------------------ |
| `TABLESPACE_NAME`           | Name.                                      |
| `BLOCK_SIZE`                | Bytes.                                     |
| `INITIAL_EXTENT`            | For dictionary-managed (legacy).           |
| `NEXT_EXTENT`               | For dictionary-managed.                    |
| `MIN_EXTENTS / MAX_EXTENTS` | Storage.                                   |
| `PCT_INCREASE`              | Storage growth %.                          |
| `MIN_EXTLEN`                | Minimum extent length.                     |
| `STATUS`                    | `ONLINE`, `OFFLINE`, `READ ONLY`.          |
| `CONTENTS`                  | `PERMANENT`, `TEMPORARY`, `UNDO`.          |
| `LOGGING`                   | `LOGGING`, `NOLOGGING`.                    |
| `FORCE_LOGGING`             | `YES`/`NO`.                                |
| `EXTENT_MANAGEMENT`         | `LOCAL`, `DICTIONARY`.                     |
| `ALLOCATION_TYPE`           | `SYSTEM`, `UNIFORM`, `USER`.               |
| `SEGMENT_SPACE_MANAGEMENT`  | `MANUAL`, `AUTO` (ASSM).                   |
| `BIGFILE`                   | `YES`/`NO`.                                |
| `ENCRYPTED`                 | `YES`/`NO` — TDE.                          |
| `COMPRESS_FOR`              | Table compression default.                 |
| `DEF_TAB_COMPRESSION`       | `ENABLED`/`DISABLED`.                      |
| `RETENTION`                 | For undo — `GUARANTEE`/`NOGUARANTEE`.      |
| `SHARED`                    | `SHARED`, `LOCAL_ON_LEAF`, `LOCAL_ON_ALL`. |

## Common Queries

```sql
-- Basic inventory
SELECT tablespace_name, contents, status, extent_management,
       segment_space_management, bigfile, encrypted
FROM   dba_tablespaces
ORDER  BY tablespace_name;

-- Undo tablespaces
SELECT tablespace_name, status, retention
FROM   dba_tablespaces
WHERE  contents='UNDO';

-- With size (join to DBA_DATA_FILES)
SELECT   ts.tablespace_name, ts.contents,
         ROUND(SUM(df.bytes)/1024/1024/1024, 2) size_gb,
         ROUND(SUM(df.maxbytes)/1024/1024/1024, 2) max_gb
FROM     dba_tablespaces ts JOIN dba_data_files df USING (tablespace_name)
GROUP BY ts.tablespace_name, ts.contents
ORDER BY 3 DESC;

-- Temp tablespaces
SELECT   tablespace_name,
         ROUND(SUM(bytes)/1024/1024/1024,2) size_gb
FROM     dba_temp_files
GROUP BY tablespace_name;

-- Free space
SELECT   tablespace_name,
         ROUND(SUM(bytes)/1024/1024/1024,2) free_gb
FROM     dba_free_space
GROUP BY tablespace_name
ORDER BY 2 DESC;
```

## Fill % (Common Alert Query)

```sql
SELECT   df.tablespace_name,
         ROUND(SUM(df.bytes)/1024/1024/1024, 2) alloc_gb,
         ROUND((SUM(df.bytes) - NVL(SUM(fs.bytes),0))/1024/1024/1024, 2) used_gb,
         ROUND((SUM(df.bytes) - NVL(SUM(fs.bytes),0))*100/SUM(df.bytes), 1) pct_used
FROM     dba_data_files df
LEFT JOIN (SELECT tablespace_name, SUM(bytes) bytes FROM dba_free_space
           GROUP BY tablespace_name) fs USING (tablespace_name)
GROUP BY df.tablespace_name
HAVING   ROUND((SUM(df.bytes) - NVL(SUM(fs.bytes),0))*100/SUM(df.bytes), 1) > 80
ORDER BY 4 DESC;
```

## References

- Oracle Database Reference 19c — `DBA_TABLESPACES`
- [Tablespaces](../../04-storage/tablespaces.md)
