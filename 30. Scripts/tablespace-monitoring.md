# Tablespace Monitoring Scripts

## Fill Percentage (Sorted by Danger)

```sql
COLUMN tablespace_name FORMAT A30

SELECT   df.tablespace_name,
         ROUND(SUM(df.bytes)/1024/1024/1024, 2) alloc_gb,
         ROUND((SUM(df.bytes)-NVL(SUM(fs.bytes),0))/1024/1024/1024, 2) used_gb,
         ROUND(NVL(SUM(fs.bytes),0)/1024/1024/1024, 2) free_gb,
         ROUND((SUM(df.bytes)-NVL(SUM(fs.bytes),0))*100/SUM(df.bytes), 2) pct_used,
         ROUND(SUM(df.maxbytes)/1024/1024/1024, 2) max_gb
FROM     dba_data_files df
LEFT JOIN (SELECT tablespace_name, SUM(bytes) bytes FROM dba_free_space
           GROUP BY tablespace_name) fs USING (tablespace_name)
GROUP BY df.tablespace_name
ORDER BY pct_used DESC;
```

## Autoextend Health

```sql
SELECT tablespace_name, file_name,
       ROUND(bytes/1024/1024/1024,2) size_gb,
       ROUND(maxbytes/1024/1024/1024,2) max_gb,
       autoextensible,
       ROUND(increment_by *
             (SELECT block_size FROM dba_tablespaces t
              WHERE t.tablespace_name = d.tablespace_name)/1024/1024, 2) inc_mb
FROM   dba_data_files d
ORDER  BY tablespace_name, file_name;

-- Non-autoextend datafiles (surprises)
SELECT tablespace_name, file_name FROM dba_data_files
WHERE  autoextensible = 'NO'
ORDER  BY 1, 2;
```

## TEMP Usage

```sql
SELECT tablespace_name,
       ROUND(bytes_used/1024/1024/1024, 2) used_gb,
       ROUND(bytes_free/1024/1024/1024, 2) free_gb,
       ROUND(bytes_used*100/(bytes_used+bytes_free), 1) pct_used
FROM   v$temp_space_header;

-- Who's using TEMP right now?
SELECT s.sid, s.username, s.sql_id,
       ROUND(su.blocks * ts.block_size/1024/1024, 2) mb,
       su.segtype, su.tablespace
FROM   v$sort_usage su
JOIN   v$session s ON s.saddr = su.session_addr
JOIN   dba_tablespaces ts ON ts.tablespace_name = su.tablespace
ORDER  BY mb DESC;
```

## Undo Usage

```sql
-- Undo tablespace state
SELECT tablespace_name,
       ROUND(SUM(bytes)/1024/1024/1024, 2) size_gb
FROM   dba_data_files
WHERE  tablespace_name IN (SELECT value FROM v$parameter WHERE name = 'undo_tablespace')
GROUP  BY tablespace_name;

-- Undo stats
SELECT   TO_CHAR(begin_time,'YYYY-MM-DD HH24:MI') t,
         undoblks, txncount, maxquerylen,
         ssolderrcnt ora_1555, nospaceerrcnt ora_30036,
         tuned_undoretention/60 tuned_min
FROM     v$undostat
ORDER BY begin_time DESC
FETCH FIRST 20 ROWS ONLY;
```

## Historical Growth (7 days)

```sql
SELECT   h.tablespace_id, ts.tablespace_name,
         MIN(h.tablespace_usedsize)*8/1024 min_used_mb,
         MAX(h.tablespace_usedsize)*8/1024 max_used_mb,
         (MAX(h.tablespace_usedsize)-MIN(h.tablespace_usedsize))*8/1024 growth_mb
FROM     dba_hist_tbspc_space_usage h
JOIN     dba_tablespaces ts ON ts.tablespace_name = (
             SELECT tablespace_name FROM dba_tablespaces
             WHERE ts# = h.tablespace_id AND ROWNUM=1)
WHERE    h.snap_id BETWEEN
             (SELECT MAX(snap_id) FROM dba_hist_snapshot
              WHERE end_interval_time < SYSDATE - 7)
         AND (SELECT MAX(snap_id) FROM dba_hist_snapshot)
GROUP BY h.tablespace_id, ts.tablespace_name
HAVING   (MAX(h.tablespace_usedsize)-MIN(h.tablespace_usedsize))*8/1024 > 0
ORDER BY 5 DESC;
```

## Top 20 Largest Segments

```sql
SELECT   owner, segment_name, partition_name, segment_type,
         tablespace_name,
         ROUND(bytes/1024/1024/1024, 2) gb
FROM     dba_segments
ORDER BY bytes DESC
FETCH FIRST 20 ROWS ONLY;
```

## Datafile IO Hotspots

```sql
SELECT   df.tablespace_name, df.file_name,
         fs.phyrds, fs.phywrts,
         ROUND(fs.readtim/GREATEST(fs.phyrds,1),3) avg_read_cs,
         ROUND(fs.writetim/GREATEST(fs.phywrts,1),3) avg_write_cs
FROM     v$filestat fs JOIN dba_data_files df ON df.file_id = fs.file#
ORDER BY fs.phyrds+fs.phywrts DESC
FETCH FIRST 15 ROWS ONLY;
```

## Related

- [DBA_TABLESPACES](../25-reference/dba-views/dba-tablespaces.md).
- [DBA_DATA_FILES](../25-reference/dba-views/dba-data-files.md).
- [Tablespace Full runbook](../27-runbooks/tablespace-full.md).
