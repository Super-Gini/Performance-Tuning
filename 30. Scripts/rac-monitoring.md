# RAC Monitoring Scripts

## Instance Status Cluster-Wide

```sql
SELECT inst_id, instance_name, host_name, status, database_status,
       ROUND(SYSDATE - startup_time, 2) days_up
FROM   gv$instance
ORDER  BY inst_id;
```

## Sessions per Instance

```sql
SELECT inst_id, COUNT(*) total,
       SUM(DECODE(status,'ACTIVE',1,0)) active
FROM   gv$session
WHERE  type = 'USER'
GROUP  BY inst_id
ORDER  BY inst_id;
```

## GCS (Global Cache) Stats

```sql
-- Buffer transfers between instances
SELECT   inst_id, name, value
FROM     gv$sysstat
WHERE    name IN ('gc cr blocks received','gc current blocks received',
                  'gc cr blocks served','gc current blocks served',
                  'gc cr block receive time','gc current block receive time')
ORDER BY name, inst_id;

-- Average GCS wait
SELECT   inst_id, event, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(average_wait, 2) avg_cs
FROM     gv$system_event
WHERE    event LIKE 'gc%' AND wait_class <> 'Idle'
   AND   time_waited > 0
ORDER BY inst_id, time_waited DESC;
```

## GES (Global Enqueue) Stats

```sql
SELECT inst_id, resource_name, current_utilization, max_utilization
FROM   gv$resource_limit
WHERE  resource_name IN ('gcs_resources','ges_resources','ges_enqueues')
ORDER  BY inst_id, resource_name;
```

## Interconnect Traffic

```sql
SELECT   inst_id, name, ip_address, is_public, source
FROM     gv$cluster_interconnects
ORDER BY inst_id;

SELECT   inst_id, ROUND(sum_bytes_sent/1024/1024/1024, 2) gb_sent,
         ROUND(sum_bytes_received/1024/1024/1024, 2) gb_received
FROM     gv$dlm_traffic_controller
GROUP BY inst_id, sum_bytes_sent, sum_bytes_received
ORDER BY inst_id;
```

## Service Placement

```sql
SELECT inst_id, name network_name, TRIM(pdb) pdb,
       creation_date
FROM   gv$services
WHERE  name NOT LIKE 'SYS$%'
ORDER  BY inst_id, name;
```

From CRS side:

```bash
srvctl config service -db PRD
srvctl status service -db PRD
```

## Cluster Resource State

```bash
crsctl status resource -t
crsctl check cluster -all

# Voting disks
crsctl query css votedisk

# OCR
ocrcheck
```

## Recent Reconfiguration Events

```sql
SELECT   inst_id, reconfig#, reconfig_hrs, event, reason,
         cpu_time, memory_used
FROM     gv$rac_reconfiguration
ORDER BY inst_id, reconfig# DESC
FETCH FIRST 20 ROWS ONLY;
```

## Cluster Alerts (Recent)

```sql
SELECT   originating_timestamp, host_id, message_text
FROM     v$diag_alert_ext
WHERE    originating_timestamp > SYSDATE - 1
   AND   (message_text LIKE '%evict%' OR message_text LIKE '%IPC%'
          OR message_text LIKE '%reconfig%')
ORDER BY originating_timestamp DESC;
```

## Cache Fusion Bottleneck (Which Block?)

```sql
SELECT   sql_id, event, p1 file#, p2 block#, p3 blocks,
         COUNT(*) samples
FROM     gv$active_session_history
WHERE    sample_time > SYSDATE - 15/1440
   AND   event LIKE 'gc%'
GROUP BY sql_id, event, p1, p2, p3
ORDER BY samples DESC
FETCH FIRST 20 ROWS ONLY;
```

## Related

- [RAC](../18-rac/index.md).
- [Cache Fusion](../18-rac/cache-fusion.md).
- [RAC Node Eviction runbook](../27-runbooks/rac-node-eviction.md).
