# Runbook: High I/O

## Symptom

- User I/O wait class dominant.
- `db file sequential read` / `db file scattered read` average > 20 ms.
- Application slow, disk queue depth high.

## Triage

### OS side

```bash
iostat -x 5 3          # per-device utilization
sar -d 5 3             # disk stats
```

Key metrics:

- `%util` > 90 → saturated.
- `await` > 20 ms → slow.
- `avgqu-sz` > 4 → queued.

### DB side

```sql
-- Top wait events last 15 min via V$SYSMETRIC or ASH
SELECT event, COUNT(*) samples, wait_class
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 15/1440
   AND wait_class = 'User I/O'
GROUP  BY event, wait_class
ORDER  BY 2 DESC;

-- SQL causing I/O
SELECT   sql_id, COUNT(*) samples, wait_class
FROM     v$active_session_history
WHERE    sample_time > SYSDATE - 15/1440
   AND   wait_class = 'User I/O'
GROUP BY sql_id, wait_class
ORDER BY 2 DESC
FETCH FIRST 10 ROWS ONLY;

-- Per-datafile I/O
SELECT   f.name, s.phyrds, s.phywrts,
         ROUND(s.readtim/GREATEST(s.phyrds,1),3) avg_read_cs,
         ROUND(s.writetim/GREATEST(s.phywrts,1),3) avg_write_cs
FROM     v$datafile f JOIN v$filestat s ON s.file# = f.file#
ORDER BY s.phyrds+s.phywrts DESC
FETCH FIRST 15 ROWS ONLY;

-- IO stats by type (buffer/direct/temp)
SELECT filetype_name,
       small_read_reqs+large_read_reqs total_reads,
       small_write_reqs+large_write_reqs total_writes
FROM   v$iostat_file
ORDER  BY total_reads+total_writes DESC;
```

## Actions

### 1. Identify saturated devices

`iostat -x` shows which LUN is 100% util. Correlate to tablespace / datafile.

### 2. Reduce load

- Kill runaway sessions.
- Postpone batch jobs.
- Throttle via Resource Manager.

### 3. Balance load

- Move hot datafiles to different storage.
- Use ASM rebalance (`ALTER DISKGROUP DATA REBALANCE POWER 4`).
- Grow buffer cache to hide reads.

### 4. Address direct-path

Direct-path reads bypass cache — usually huge scans:

```sql
-- Which SQL does direct-path
SELECT   sql_id, plan_hash_value, direct_writes, physical_reads_direct
FROM     v$sql
WHERE    physical_reads_direct > 1000
ORDER BY physical_reads_direct DESC
FETCH FIRST 10 ROWS ONLY;
```

Fix: index, or move to a reporting time.

### 5. Address TEMP I/O

Sort/hash spilling to TEMP:

```sql
SELECT s.sid, s.username, s.sql_id, ROUND(su.blocks*8192/1024/1024,2) mb
FROM   v$sort_usage su JOIN v$session s ON s.saddr = su.session_addr
ORDER  BY mb DESC;
```

Fix: bigger PGA, better plan.

## Verification

```sql
SELECT metric_name, value FROM v$sysmetric
WHERE  metric_name IN ('User I/O Waits Percentage','Average Active Sessions')
   AND intsize_csec = 6000;
```

`iostat`: `%util` back to normal.

## Post-Mortem

- Was it a bad plan or genuine load?
- Storage saturated per LUN or globally?
- Buffer cache too small?
- Sudden growth in a table causing scans?

## Prevention

- Adequate buffer cache to hide OLTP reads.
- Adequate PGA for sorts.
- Reporting on standby (Active Data Guard).
- Alert on user I/O wait > 40%.
- Storage QoS or dedicated IOPS.

## Related

- [I/O Analysis](../12-performance-tuning/io-analysis.md).
- [User I/O wait events](../25-reference/wait-events/user-io.md).
- [PGA](../03-instance-architecture/memory/pga.md).
