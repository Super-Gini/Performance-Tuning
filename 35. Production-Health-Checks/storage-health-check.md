# Storage Health Check

Storage subsystem audit — ASM, IO, capacity, growth.

## 1. Tablespace Capacity

```sql
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

Verify: no TS > 90%; enough MAXBYTES for 6 months growth.

## 2. Datafile Configuration

```sql
-- Non-autoextend files (surprise!)
SELECT tablespace_name, file_name FROM dba_data_files
WHERE  autoextensible='NO';

-- Files near their max
SELECT tablespace_name, file_name,
       ROUND(bytes*100/maxbytes,2) pct_of_max
FROM   dba_data_files
WHERE  autoextensible='YES' AND maxbytes > 0
   AND bytes*100/maxbytes > 80
ORDER  BY 3 DESC;

-- Small autoextend increments (frequent extends slow)
SELECT tablespace_name, file_name,
       increment_by *
       (SELECT block_size FROM dba_tablespaces t WHERE t.tablespace_name = d.tablespace_name)/1024/1024 inc_mb
FROM   dba_data_files d
WHERE  autoextensible='YES'
   AND increment_by *
       (SELECT block_size FROM dba_tablespaces t WHERE t.tablespace_name = d.tablespace_name) < 100*1024*1024
ORDER  BY 3;
```

Verify: all autoextend on, increment >= 100 MB.

## 3. Bigfile Candidates

Small-file tablespaces above 32 GB with many files:

```sql
SELECT tablespace_name, COUNT(*) file_count,
       ROUND(SUM(bytes)/1024/1024/1024, 2) total_gb
FROM   dba_data_files
GROUP  BY tablespace_name
HAVING COUNT(*) > 4 AND SUM(bytes)/1024/1024/1024 > 100
ORDER  BY 3 DESC;

-- Bigfile in use
SELECT tablespace_name, bigfile FROM dba_tablespaces WHERE bigfile='YES';
```

## 4. FRA

```sql
SELECT name, ROUND(space_used*100/space_limit,2) pct,
       ROUND(space_reclaimable*100/space_limit,2) pct_reclaim,
       number_of_files
FROM   v$recovery_file_dest;

SELECT file_type, percent_space_used, percent_space_reclaimable
FROM   v$flash_recovery_area_usage;
```

Verify: FRA < 80%, reclaimable growing means backups/archives can rotate.

## 5. TEMP

```sql
SELECT tablespace_name,
       ROUND(bytes_used/1024/1024/1024, 2) used_gb,
       ROUND(bytes_free/1024/1024/1024, 2) free_gb,
       ROUND(bytes_used*100/(bytes_used+bytes_free), 1) pct_used
FROM   v$temp_space_header;

-- Big users
SELECT s.sid, s.username, s.sql_id, ROUND(su.blocks*8192/1024/1024,2) mb
FROM   v$sort_usage su JOIN v$session s ON s.saddr = su.session_addr
ORDER  BY mb DESC
FETCH  FIRST 10 ROWS ONLY;
```

Verify: TEMP < 70%, big sorts identified.

## 6. Undo

```sql
SELECT ROUND(SUM(bytes)/1024/1024/1024,2) undo_gb
FROM   dba_data_files
WHERE  tablespace_name = (SELECT value FROM v$parameter WHERE name='undo_tablespace');

-- Retention success
SELECT   MAX(tuned_undoretention) tuned_seconds,
         MAX(maxquerylen) longest_query,
         SUM(ssolderrcnt) ora_1555_count,
         SUM(nospaceerrcnt) ora_30036_count
FROM     v$undostat
WHERE    begin_time > SYSDATE - 7;
```

Verify: no ORA-01555 or ORA-30036, tuned_retention comfortably above longest_query.

## 7. Segment Growth (Historical)

```sql
SELECT   owner_name, object_name, subobject_name,
         MAX(space_used_delta)/1024/1024 max_delta_mb,
         COUNT(*) snaps
FROM     dba_hist_seg_stat
JOIN     dba_hist_snapshot USING (snap_id)
WHERE    end_interval_time > SYSDATE - 30
GROUP BY owner_name, object_name, subobject_name
HAVING   MAX(space_used_delta) > 1024*1024*1024
ORDER BY max_delta_mb DESC
FETCH FIRST 30 ROWS ONLY;
```

Verify: growth aligns with business.

## 8. ASM Diskgroup Health

```sql
SELECT name, state, type,
       ROUND(total_mb/1024, 2) total_gb,
       ROUND(free_mb/1024, 2)  free_gb,
       ROUND(usable_file_mb/1024, 2) usable_gb,
       ROUND(100*(1-free_mb/total_mb), 1) pct_used
FROM   v$asm_diskgroup;
```

Verify: no DG > 85%, usable_file_mb positive (enough space to survive disk loss).

## 9. ASM Rebalance

```sql
SELECT * FROM v$asm_operation;
```

Verify: no ongoing rebalance (or if there is, expected timing).

## 10. ASM Disk Status

```sql
SELECT dg.name diskgroup, d.name disk, d.path,
       d.state, d.mount_status, d.mode_status, d.failgroup,
       d.repair_timer
FROM   v$asm_diskgroup dg JOIN v$asm_disk d USING (group_number)
WHERE  d.state <> 'NORMAL' OR d.mount_status <> 'CACHED' OR d.mode_status <> 'ONLINE'
ORDER  BY dg.name;
```

Verify: all disks NORMAL, CACHED, ONLINE.

## 11. Failure Group Balance

```sql
SELECT dg.name diskgroup, d.failgroup,
       COUNT(*) disks,
       ROUND(SUM(d.total_mb)/1024, 2) total_gb
FROM   v$asm_diskgroup dg JOIN v$asm_disk d USING (group_number)
GROUP  BY dg.name, d.failgroup
ORDER  BY dg.name, d.failgroup;
```

Verify: each DG has ≥ N failure groups per redundancy level (2 for NORMAL, 3 for HIGH).

## 12. IO Latency

```sql
SELECT df.tablespace_name, df.file_name,
       fs.phyrds, fs.phywrts,
       ROUND(fs.readtim/GREATEST(fs.phyrds,1)*10, 2) avg_read_ms,
       ROUND(fs.writetim/GREATEST(fs.phywrts,1)*10, 2) avg_write_ms
FROM   v$filestat fs JOIN dba_data_files df ON df.file_id = fs.file#
WHERE  fs.phyrds+fs.phywrts > 100
ORDER  BY fs.phyrds+fs.phywrts DESC
FETCH  FIRST 20 ROWS ONLY;
```

Verify: reads < 20 ms, writes < 15 ms (for OLTP tier).

## 13. FS-Level Health (OS)

```bash
# Filesystem free
df -h

# ASM disks alive
ls -la /dev/oracleasm/disks/  # or afd
asmcmd afd_lsdsk 2>/dev/null

# Multipath
multipath -ll

# Kernel messages last 24h
dmesg -T | grep -iE "sd |i/o error|multipath" | tail -30
```

## 14. Snapshot Backup Storage

For cloud (EBS, ASM on cloud):

- Snapshot IOPS provisioned matches workload.
- Snapshot storage cost trending.

## 15. Growth Forecast

```sql
-- Last 90-day trend, extrapolate 12 months
SELECT tablespace_name,
       ROUND(MIN(tablespace_usedsize)*8/1024/1024, 2) start_gb,
       ROUND(MAX(tablespace_usedsize)*8/1024/1024, 2) end_gb,
       ROUND((MAX(tablespace_usedsize)-MIN(tablespace_usedsize))*8/1024/1024/3, 2) monthly_growth_gb,
       ROUND((MAX(tablespace_usedsize)-MIN(tablespace_usedsize))*8/1024/1024*4, 2) projected_year_gb
FROM   dba_hist_tbspc_space_usage
WHERE  snap_id > (SELECT MIN(snap_id) FROM dba_hist_snapshot WHERE end_interval_time > SYSDATE - 90)
GROUP  BY tablespace_id, tablespace_name
HAVING (MAX(tablespace_usedsize)-MIN(tablespace_usedsize))*8/1024/1024 > 5
ORDER  BY 4 DESC;
```

## 16. Deliverable

- Capacity map today + 12 months.
- IO latency baseline.
- ASM redundancy adequacy.
- Order of storage/rebalance actions.

## Related

- [ASM](../19-asm/index.md).
- [Tablespace Full runbook](../27-runbooks/tablespace-full.md).
- [Storage chapter](../04-storage/index.md).
