# RDS Oracle Monitoring

## Layers

| Layer                    | Metric type        | Granularity |
| ------------------------ | ------------------ | ----------- |
| CloudWatch (RDS metrics) | Coarse OS + DB     | 60 s        |
| Enhanced Monitoring      | Per-process OS     | 1–60 s      |
| Performance Insights     | DB load (ASH-like) | 1 s         |
| Oracle native views      | Everything else    | Real-time   |

## CloudWatch Metrics (Built-In)

Available without extra config:

- `CPUUtilization`
- `DatabaseConnections`
- `FreeStorageSpace`, `FreeableMemory`
- `ReadIOPS`, `WriteIOPS`, `ReadLatency`, `WriteLatency`
- `NetworkReceiveThroughput`, `NetworkTransmitThroughput`
- `ReplicaLag` (read replicas)

Set alarms:

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name prd-cpu-high \
    --metric-name CPUUtilization \
    --namespace AWS/RDS \
    --statistic Average \
    --period 300 \
    --evaluation-periods 3 \
    --threshold 85 \
    --dimensions Name=DBInstanceIdentifier,Value=prd \
    --comparison-operator GreaterThanThreshold \
    --alarm-actions arn:aws:sns:...:my-alerts
```

## Enhanced Monitoring

Container agent inside the RDS host publishes per-process, per-second OS metrics to CloudWatch Logs.

Enable:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --monitoring-interval 60 \
    --monitoring-role-arn arn:aws:iam::...:role/rds-monitoring
```

Available metrics:

- Per-process CPU / memory / IO
- Filesystem utilization
- Network per-interface
- Kernel metrics

Access via console **or** CloudWatch Logs `/aws/rds/instance/<db>/enhanced_monitoring`.

## Performance Insights

DB load view. Enable at create time (or later):

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --enable-performance-insights \
    --performance-insights-retention-period 7 \
    --apply-immediately
```

Access via RDS console → Performance Insights.

Shows:

- **Average Active Sessions (AAS)** over time.
- Breakdown by wait class.
- Top SQL by DB load.
- Top hosts / users / applications.

Retention: 7 days (free), 24 months (paid).

Query PI programmatically:

```bash
aws pi get-resource-metrics \
    --service-type RDS \
    --identifier <db-resource-id> \
    --start-time 2026-08-06T10:00:00Z \
    --end-time 2026-08-06T11:00:00Z \
    --period-in-seconds 60 \
    --metric-queries '[{"Metric":"db.load.avg"}]'
```

## Native Oracle Views

All the standard views work inside RDS:

```sql
-- Sessions
SELECT sid, username, event, wait_class, sql_id
FROM   v$session WHERE type='USER' AND status='ACTIVE';

-- Wait classes
SELECT wait_class, SUM(time_waited)/100/60 mins
FROM   v$system_event
WHERE  wait_class <> 'Idle'
GROUP  BY wait_class ORDER BY 2 DESC;

-- Top SQL
SELECT sql_id, executions, elapsed_time, cpu_time
FROM   v$sqlarea
ORDER  BY elapsed_time DESC
FETCH  FIRST 20 ROWS ONLY;
```

## AWR / ASH

AWR runs normally on RDS. Generate reports:

```sql
-- Via rdsadmin
BEGIN
  rdsadmin.rdsadmin_diagnostic_util.dump_awr_report(
    p_bsnapshot => &begin_snap,
    p_esnapshot => &end_snap,
    p_report_type => 'html');
END;
/
```

Or the standard way:

```sql
SPOOL /tmp/awr.html
SELECT DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
       (SELECT dbid FROM v$database),
       (SELECT instance_number FROM v$instance),
       &begin_snap_id, &end_snap_id)
FROM   dual;
SPOOL OFF
```

## Log Files

Alert log, listener log, trace files stored in `bdump/adump/trace/`. Access:

```bash
aws rds describe-db-log-files --db-instance-identifier prd

aws rds download-db-log-file-portion \
    --db-instance-identifier prd \
    --log-file-name trace/alert_PRD.log.YYYY-MM-DD \
    --output text > alert.log
```

Or export to CloudWatch Logs continuously:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --cloudwatch-logs-export-configuration '{"EnableLogTypes":["alert","audit","listener","trace"]}'
```

Alert log + CloudWatch Insights = great for detecting `ORA-00600` fleet-wide.

## Recommended Baseline Monitoring

- **CloudWatch alarms** on CPU, storage, IOPS latency, connection count.
- **Enhanced Monitoring** enabled (60 s at minimum).
- **Performance Insights** enabled with 7-day retention.
- **CloudWatch Logs export** for alert log — feed to SIEM.
- **Native AWR** for capacity planning.

## Related

- [RDS Architecture](rds-architecture.md).
- [Performance (Cloud chapter)](../29-cloud/aws-rds/performance.md).
- [Troubleshooting (Cloud chapter)](../29-cloud/aws-rds/troubleshooting.md).
- [AWR](../12-performance-tuning/awr.md), [ASH](../12-performance-tuning/ash.md).
