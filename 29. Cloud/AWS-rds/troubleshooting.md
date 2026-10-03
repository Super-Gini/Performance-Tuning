# AWS RDS Oracle — Troubleshooting

## Overview

RDS-specific issues you'll hit that aren't in the standard on-prem playbook.

## Symptom: Instance in `storage-full` state

**Cause**: Storage hit capacity.

**Fix**:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --allocated-storage 500 \
    --apply-immediately
```

Storage extension takes 15–60 min but non-blocking after the initial modify.

**Prevent**: Enable storage autoscaling.

## Symptom: `insufficient-capacity` at create/scale time

**Cause**: AWS out of capacity for that instance type in that AZ.

**Fix**: Try another AZ, another instance class, or wait.

## Symptom: RDS won't start after reboot

**Cause**: Usually a parameter change with `ApplyMethod=immediate` that made the DB unstartable (bad init parameter).

**Fix**:

1. Revert the parameter group:
   ```bash
   aws rds modify-db-instance --db-instance-identifier prd \
       --db-parameter-group-name default.oracle-ee-19 \
       --apply-immediately
   ```
2. Reboot from console.
3. Fix the parameter, then re-apply the custom group.

## Symptom: ORA-01555 spike

**Cause**: Storage autoscaling isn't tuning undo retention automatically.

**Fix**: Bump via parameter group:

```
undo_retention = 10800
undo_tablespace = <auto or explicit>
```

Reboot to apply.

## Symptom: Multi-AZ failover took > 5 min

**Cause**: Multi-AZ commit acknowledgement had large replay queue.

**Fix**: If frequent, investigate long-running batch that generated redo faster than replica could apply. Consider read replica for the batch workload.

## Symptom: `ORA-00060` deadlocks — no trace files

**Cause**: Trace files rotated before you fetched.

**Fix**: List and download quickly:

```bash
aws rds describe-db-log-files --db-instance-identifier prd \
    --filename-contains 'deadlock' \
    --file-last-written $(date -d '-1 hour' +%s)000

aws rds download-db-log-file-portion \
    --db-instance-identifier prd --log-file-name trace/...trc \
    --output text
```

Enable frequent CloudWatch Logs export:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --cloudwatch-logs-export-configuration '{"EnableLogTypes":["alert","audit","listener","trace"]}'
```

## Symptom: Can't run `ALTER SYSTEM KILL SESSION` — ORA-01031

**Cause**: You don't have `ALTER SYSTEM` — RDS restricts this.

**Fix**:

```sql
EXEC rdsadmin.rdsadmin_util.kill(150, 32458);
```

`p_method` parameter takes `IMMEDIATE` or `PROCESS`.

## Symptom: Can't `ALTER USER SYS IDENTIFIED BY ...`

**Cause**: RDS blocks direct SYS modification.

**Fix**: Rotate the master password via API:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --master-user-password NewPw2026 --apply-immediately
```

Master user is your account with quasi-SYSDBA rights via `rdsadmin`.

## Symptom: Data Pump import fails with "cannot access DATA_PUMP_DIR"

**Cause**: RDS-managed directory. You can't create new directories.

**Fix**: Use built-in `DATA_PUMP_DIR`:

```sql
-- Check
SELECT directory_name, directory_path FROM dba_directories WHERE directory_name = 'DATA_PUMP_DIR';

-- Upload from S3 first
BEGIN
  rdsadmin.rdsadmin_s3_tasks.download_from_s3(
    p_bucket_name    => 'my-oracle-dumps',
    p_directory_name => 'DATA_PUMP_DIR');
END;
/

-- Then impdp
```

## Symptom: Automated backup failed silently

**Cause**: RDS shows in Events; you may miss it if you don't have EventBridge integration.

**Fix**: Set up EventBridge rule for RDS backup events. Alert to Slack / PagerDuty.

## Symptom: Read replica lag high

**Cause**: Async replication; primary is generating redo faster than replica applies.

**Fix**:

- Read replica instance class same or larger than primary.
- Read replica's storage same tier.
- Investigate primary's redo rate — batch job problem?
- If sustained, promote a bigger replica.

## Symptom: RDS won't accept `ALTER SYSTEM SET ... SCOPE=BOTH`

**Cause**: Not all parameters are dynamic; RDS routes through parameter group.

**Fix**: Modify parameter group with `ApplyMethod=pending-reboot` or `immediate` if the param is dynamic-per-Oracle.

## Symptom: `rdsadmin` package errors

**Cause**: Called from PDB in a Multitenant setup — some `rdsadmin` calls must run in CDB root.

**Fix**: `ALTER SESSION SET CONTAINER = CDB$ROOT` before the call.

## General Debugging Tools

- **CloudWatch Logs** — alert log, listener log, trace files.
- **Enhanced Monitoring** — OS metrics.
- **Performance Insights** — DB load.
- **CloudTrail** — API-level audit for who modified the instance.
- **`rdsadmin.rdsadmin_util`** — utility procedures.

## Related

- [Overview](overview.md).
- [Backups](backups.md).
- [Performance](performance.md).
- [Alert Log](../../24-monitoring/alert-log.md).
