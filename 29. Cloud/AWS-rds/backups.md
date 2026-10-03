# AWS RDS Oracle — Backups

## Model

RDS backups are **snapshot-based**, not RMAN. AWS takes a full block-level snapshot on schedule, then streams transaction logs so you can point-in-time recover between snapshots.

## Automated Backups

- Enabled by setting `BackupRetentionPeriod` > 0 (up to 35).
- Daily snapshot inside your configured window.
- Transaction logs streamed continuously.
- **PITR** — you can restore to any second within retention.
- **Storage** on AWS-managed S3, no extra cost within retention.

Enable:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --backup-retention-period 30 \
    --preferred-backup-window 02:00-03:00 \
    --apply-immediately
```

## Manual Snapshots

Take on demand:

```bash
aws rds create-db-snapshot \
    --db-snapshot-identifier prd-pre-app-release \
    --db-instance-identifier prd
```

Retained until you delete. Can be shared cross-account, copied cross-region.

## Point-in-Time Restore

Restore to a specific timestamp within retention:

```bash
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier prd \
    --target-db-instance-identifier prd-recovered \
    --restore-time 2026-08-06T12:34:00Z \
    --db-subnet-group-name my-subnet-group
```

Creates a **new** instance — you don't overwrite the original.

## Restore from Snapshot

```bash
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier prd-restored \
    --db-snapshot-identifier prd-pre-app-release \
    --db-subnet-group-name my-subnet-group
```

## Cross-Region Snapshot Copy

For DR / compliance:

```bash
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier arn:aws:rds:us-east-1:...:snapshot:prd-manual \
    --target-db-snapshot-identifier prd-manual-copy \
    --kms-key-id <kms_key_in_target_region> \
    --region us-west-2
```

## Data Pump Backups (extra)

For portable/logical backups you control:

```sql
-- Export to RDS DATA_PUMP_DIR
DECLARE
  h1 NUMBER;
BEGIN
  h1 := DBMS_DATAPUMP.OPEN('EXPORT','SCHEMA',NULL,'app_export');
  DBMS_DATAPUMP.ADD_FILE(h1,'app.dmp','DATA_PUMP_DIR');
  DBMS_DATAPUMP.METADATA_FILTER(h1,'SCHEMA_EXPR','IN (''APP'')');
  DBMS_DATAPUMP.START_JOB(h1);
END;
/
```

Then move to S3:

```sql
BEGIN
  rdsadmin.rdsadmin_s3_tasks.upload_to_s3(
    p_bucket_name    => 'my-oracle-dumps',
    p_prefix         => 'app.dmp',
    p_s3_prefix      => 'exports/2026-08-06/',
    p_directory_name => 'DATA_PUMP_DIR');
END;
/
```

## What You Can't Do

- Full-database RMAN backup for compliance/off-cloud archival — the automated snapshot is inside AWS.
- Restore an RDS snapshot to EC2 — not portable.
- Restore an EC2 backup to RDS — must use Data Pump / DMS.

## Costs

- Automated backup storage up to your DB size is free.
- Beyond that, per-GB-month S3-backed cost.
- Manual snapshots and cross-region copies always billed.

## Retention Planning

| Regulation   | Recommended retention |
| ------------ | --------------------- |
| Internal ops | 7–14 days             |
| SOX          | 7 years (manual snap) |
| HIPAA        | 6 years (manual snap) |
| PCI          | 1 year rolling        |

Combine short PITR window (~14 days) with periodic manual snapshots (monthly for a year).

## Related

- [RDS Overview](overview.md).
- [Data Pump](../../21-data-pump/index.md).
- [Backup Strategy on-prem](../../15-rman/backup-strategy.md).
