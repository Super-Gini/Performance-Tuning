# AWS RDS Oracle — Performance

## Overview

Tuning RDS is like tuning on-prem Oracle **minus** OS-level knobs. You get:

- Init parameters via **parameter groups** (subset of Oracle parameters, some AWS-locked).
- Instance class scaling (vertical).
- Storage class + IOPS scaling.
- **Performance Insights** (CloudWatch-integrated ASH-like view).
- **Enhanced Monitoring** (per-process, per-second OS metrics — not `ps`, but pulled via container agent).

## Sizing

- CPU: `db.r6i.*` (memory-optimized).
- RAM: rule of thumb is 4–8× your working set.
- Storage: `io2` for latency-critical, `gp3` for cost.

Scale up:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --db-instance-class db.r6i.8xlarge \
    --apply-immediately
```

Downtime: ~5 minutes for the instance flip.

## Parameter Groups

Default groups are locked; you must create a custom group and reboot to apply.

```bash
aws rds create-db-parameter-group \
    --db-parameter-group-name prd-19c-custom \
    --db-parameter-group-family oracle-ee-19 \
    --description "Prod tuned"

aws rds modify-db-parameter-group \
    --db-parameter-group-name prd-19c-custom \
    --parameters "ParameterName=optimizer_use_sql_plan_baselines,ParameterValue=TRUE,ApplyMethod=immediate"

aws rds modify-db-instance --db-instance-identifier prd \
    --db-parameter-group-name prd-19c-custom \
    --apply-immediately
```

Not all init parameters are exposed. AWS blocks some (memory sizing controlled by instance class).

## Performance Insights

Enabled at DB create. Access via console or API. Shows:

- DB Load (average active sessions).
- Wait class breakdown.
- Top SQL by DB Load.
- Top hosts / users.

Retention: 7 days (free), 24 months (paid).

## Enhanced Monitoring

Container agent inside the RDS host publishes OS metrics to CloudWatch Logs at up to 1-second granularity.

Enable:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --monitoring-interval 60 \
    --monitoring-role-arn arn:aws:iam::...:role/rds-monitoring
```

## AWR/ASH

Available inside the DB — same views as on-prem. Run reports via SQL\*Plus:

```sql
SELECT * FROM TABLE(DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
    (SELECT dbid FROM v$database),
    (SELECT instance_number FROM v$instance),
    &begin_snap_id, &end_snap_id));
```

Or use `rdsadmin`:

```sql
EXEC rdsadmin.rdsadmin_diagnostic_util.dump_awr_report(&start_snap, &end_snap);
```

## Tracing

Trace files aren't directly accessible but AWS lets you download via console:

```bash
aws rds describe-db-log-files --db-instance-identifier prd

aws rds download-db-log-file-portion \
    --db-instance-identifier prd \
    --log-file-name trace/PRD_ora_12345.trc \
    --output text > local_trace.txt
```

## Common Tuning Patterns

- **SGA sizing**: Adjust the `memory_target` parameter via the parameter group.
- **PGA**: `pga_aggregate_target` via parameter group.
- **Optimizer stability**: `optimizer_features_enable`, `optimizer_adaptive_statistics=FALSE`.
- **Result cache**: `result_cache_mode='MANUAL'`, `result_cache_max_size`.
- **Cursor sharing**: `cursor_sharing='EXACT'` (default).

## Storage Autoscaling

Enable so you don't get ORA-01653 in the middle of the night:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --max-allocated-storage 4096 \
    --apply-immediately
```

## Limitations to Watch

- Max 40 databases per instance (SE), 300 (EE).
- Max concurrent connections: instance class dependent.
- IOPS max 256k (io2).
- Multi-AZ IOPS = provisioned, not doubled (single primary serving).
- Read replica lag depends on AZ / region.

## Related

- [Troubleshooting](troubleshooting.md).
- [Overview](overview.md).
- [AWR](../../12-performance-tuning/awr.md), [ASH](../../12-performance-tuning/ash.md).
