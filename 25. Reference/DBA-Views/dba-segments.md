# DBA_SEGMENTS

## Purpose

All storage segments — tables, indexes, LOBs, undo, temp, cluster. One row per segment.

## Key Columns

| Column                        | Meaning                                                                                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `OWNER`                       | Schema.                                                                                                                   |
| `SEGMENT_NAME`                | Segment name (table, index, LOB name).                                                                                    |
| `PARTITION_NAME`              | Partition/subpartition.                                                                                                   |
| `SEGMENT_TYPE`                | `TABLE`, `TABLE PARTITION`, `INDEX`, `LOBSEGMENT`, `LOBINDEX`, `TYPE2 UNDO`, `TEMPORARY`, `CLUSTER`, `NESTED TABLE`, etc. |
| `TABLESPACE_NAME`             | TS.                                                                                                                       |
| `BYTES`                       | Total bytes allocated.                                                                                                    |
| `BLOCKS`                      | In blocks.                                                                                                                |
| `EXTENTS`                     | Extent count.                                                                                                             |
| `INITIAL_EXTENT`              | First extent size.                                                                                                        |
| `NEXT_EXTENT`                 | Next extent size.                                                                                                         |
| `MIN_EXTENTS / MAX_EXTENTS`   | Storage.                                                                                                                  |
| `PCT_INCREASE`                | Growth %.                                                                                                                 |
| `FREELISTS / FREELIST_GROUPS` | Legacy freelist config.                                                                                                   |
| `LOGGING`                     | `YES`/`NO`.                                                                                                               |
| `BUFFER_POOL`                 | `DEFAULT`, `KEEP`, `RECYCLE`.                                                                                             |
| `HEADER_FILE / HEADER_BLOCK`  | Segment header location.                                                                                                  |

## Common Queries

```sql
-- Top 20 biggest segments
SELECT   owner, segment_name, segment_type, tablespace_name,
         ROUND(bytes/1024/1024/1024,2) gb
FROM     dba_segments
ORDER BY bytes DESC
FETCH FIRST 20 ROWS ONLY;

-- Schema footprint
SELECT   owner, ROUND(SUM(bytes)/1024/1024/1024,2) gb, COUNT(*) segments
FROM     dba_segments
GROUP BY owner
ORDER BY 2 DESC;

-- LOB usage per table
SELECT   l.owner, l.table_name, l.column_name, s.segment_name,
         ROUND(s.bytes/1024/1024/1024,2) gb
FROM     dba_lobs l
JOIN     dba_segments s ON s.owner = l.owner AND s.segment_name = l.segment_name
ORDER BY 5 DESC
FETCH FIRST 20 ROWS ONLY;

-- Partitioned tables' partition sizes
SELECT   owner, segment_name, partition_name,
         ROUND(bytes/1024/1024/1024,2) gb
FROM     dba_segments
WHERE    segment_type = 'TABLE PARTITION'
   AND   owner = '&owner'
   AND   segment_name = '&table'
ORDER BY bytes DESC;

-- Growth over last week (via ASH-adjacent)
-- Use DBA_HIST_SEG_STAT for historical growth
```

## Related

- `DBA_TABLES / DBA_INDEXES / DBA_LOBS` — logical objects.
- `DBA_EXTENTS` — extent-level detail.
- `DBA_HIST_SEG_STAT` — segment-level historical stats.

## References

- Oracle Database Reference 19c — `DBA_SEGMENTS`
- [Segments](../../04-storage/segments.md)
