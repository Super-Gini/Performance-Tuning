# RAC Health Check

Cluster-specific audit — beyond the Database Health Check items.

## 1. Cluster Topology

```bash
crsctl query css votedisk
ocrcheck
crsctl status resource -t
crsctl check cluster -all
```

Verify:

- 3+ voting disks in different failure groups.
- OCR healthy on 2+ locations.
- All cluster nodes reachable and ONLINE.

## 2. Instances

```sql
SELECT inst_id, instance_name, host_name, status, startup_time,
       ROUND(SYSDATE-startup_time,2) days_up
FROM   gv$instance
ORDER  BY inst_id;

SELECT inst_id, instance_number, thread# FROM gv$instance;
```

Verify: all instances OPEN, thread# matches instance#, uptime consistent (no recent unexplained restarts).

## 3. Redo Threads

```sql
SELECT thread#, groups, sequence#, status FROM gv$thread;

SELECT thread#, group#, bytes/1024/1024 mb, members, archived, status
FROM   gv$log
ORDER  BY thread#, group#;
```

Verify: each instance has its own thread with 3+ log groups, multiplexed.

## 4. Services

```sql
SELECT inst_id, name, network_name, creation_date
FROM   gv$services
WHERE  name NOT LIKE 'SYS$%'
ORDER  BY inst_id, name;

SELECT * FROM dba_services;
```

```bash
srvctl config service -db PRD
srvctl status service -db PRD
```

Verify:

- Services intentionally placed (preferred/available).
- Load balanced clients (JDBC UCP) know the services.
- No services running "everywhere" by mistake.

## 5. Interconnect

```sql
SELECT inst_id, name, ip_address, is_public, source
FROM   gv$cluster_interconnects
ORDER  BY inst_id;

SELECT inst_id, ROUND(sum_bytes_sent/1024/1024/1024, 2) gb_sent,
       ROUND(sum_bytes_received/1024/1024/1024, 2) gb_received
FROM   gv$dlm_traffic_controller
GROUP  BY inst_id, sum_bytes_sent, sum_bytes_received
ORDER  BY inst_id;
```

OS side (on each node):

```bash
ip link show <iface_private>
ethtool <iface_private>
netstat -s | grep -iE 'error|drop|retrans'
```

Verify: private interconnect ≥ 10 Gbps, no errors/drops, HAIP or bonded.

## 6. GCS / GES Stats

```sql
-- Global Cache blocks / interconnect
SELECT   inst_id, name, value
FROM     gv$sysstat
WHERE    name IN ('gc cr blocks received','gc current blocks received',
                  'gc cr blocks served','gc current blocks served',
                  'gc cr block receive time','gc current block receive time')
ORDER BY name, inst_id;

-- gc wait times
SELECT   inst_id, event, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(average_wait,3) cs
FROM     gv$system_event
WHERE    event LIKE 'gc%' AND wait_class <> 'Idle'
ORDER BY inst_id, time_waited DESC;
```

Verify: average `gc *` wait < 5 ms; no `gc buffer busy release` spike.

## 7. Enqueue and Latch Contention

```sql
SELECT   inst_id, event, total_waits,
         ROUND(time_waited/100/60,1) mins
FROM     gv$system_event
WHERE    wait_class IN ('Cluster','Concurrency','Application')
   AND   time_waited > 0
ORDER BY inst_id, time_waited DESC
FETCH FIRST 30 ROWS ONLY;
```

Verify: no runaway concurrency waits per node.

## 8. Recent Reconfigurations

```sql
SELECT   inst_id, reconfig#, reconfig_hrs, event, reason,
         cpu_time, memory_used
FROM     gv$rac_reconfiguration
ORDER BY inst_id, reconfig# DESC
FETCH FIRST 20 ROWS ONLY;
```

Verify: no unplanned reconfigs (evictions).

## 9. Eviction Risk

Historical CSSD miss/kill events:

```bash
grep -iE "miss|kill|evict" $GRID_HOME/log/*/cssd/ocssd.log | tail -30
```

Load spikes correlating to CSSD delays:

```bash
sar -q -f /var/log/sa/sa06 | awk '$3 > 20 {print}'
```

Verify: no missed heartbeats in last 30 days.

## 10. High-Priority Processes

```sql
SHOW PARAMETER _high_priority_processes
```

Verify: includes `LMS*|LGWR|VKTM|CKPT|LMD|LMON` at minimum.

## 11. CRS Timeouts

```bash
crsctl get css misscount
crsctl get css disktimeout
crsctl get css reboottime
```

Defaults: misscount 30, disktimeout 200, reboottime 3. Adjust only with reason.

## 12. Cluster Alerts

```sql
SELECT   originating_timestamp, host_id, message_text
FROM     v$diag_alert_ext
WHERE    originating_timestamp > SYSDATE - 30
   AND   (message_text LIKE '%evict%' OR message_text LIKE '%IPC%'
          OR message_text LIKE '%reconfig%'
          OR message_text LIKE '%LMS%')
ORDER BY originating_timestamp DESC
FETCH FIRST 30 ROWS ONLY;
```

## 13. ASM DGs Health

Cross-reference with [Storage Health Check](storage-health-check.md).

```sql
SELECT name, state, type, ROUND(free_mb/1024,2) free_gb,
       ROUND(usable_file_mb/1024,2) usable_gb
FROM   v$asm_diskgroup;

SELECT * FROM v$asm_operation;
```

## 14. Deliverable Structure

Follow [Database Health Check](database-health-check.md) template; add cluster-specific items.

## Related

- [RAC](../18-rac/index.md).
- [Clusterware](../18-rac/clusterware.md).
- [Cache Fusion](../18-rac/cache-fusion.md).
- [RAC Node Eviction runbook](../27-runbooks/rac-node-eviction.md).
