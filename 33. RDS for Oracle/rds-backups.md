# RDS Oracle Backups

## Layers of Protection

1. **Automated snapshots** — daily, retention 0–35 days.
2. **Continuous transaction logs** — PITR within retention.
3. **Manual snapshots** — you take on demand, no auto-expire.
4. **Data Pump exports** — logical, portable.
5. **Read replicas** — HA/DR, not a backup.

## Enable Automated Backups

At create time:

```bash
aws rds create-db-instance \
    ... \
    --backup-retention-period 30 \
    --preferred-backup-window "02:00-03:00"
```

Change later:

```bash
aws rds modify-db-instance --db-instance-identifier prd \
    --backup-retention-period 14 \
    --apply-immediately
```

## Point-in-Time Recovery

Restores to a **new instance** at any second within retention:

```bash
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier prd \
    --target-db-instance-identifier prd-pitr-restored \
    --restore-time 2026-08-06T12:30:45Z \
    --db-subnet-group-name my-subnet-group \
    --db-instance-class db.r6i.4xlarge
```

Timing: proportional to database size + amount of log to replay. Typically 15–60 min for a 500 GB DB.

## Manual Snapshots

Take before app releases, schema changes, etc.:

```bash
aws rds create-db-snapshot \
    --db-snapshot-identifier prd-pre-release-2026-08-06 \
    --db-instance-identifier prd
```

Manual snapshots are **not auto-deleted** — you must clean up.

Restore:

```bash
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier prd-restored \
    --db-snapshot-identifier prd-pre-release-2026-08-06 \
    --db-subnet-group-name my-subnet-group
```

## Cross-Region Snapshot Copy

For DR compliance:

```bash
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier arn:aws:rds:us-east-1:...:snapshot:prd-manual \
    --target-db-snapshot-identifier prd-manual-copy \
    --kms-key-id arn:aws:kms:us-west-2:...:key/... \
    --region us-west-2
```

Copies traverse public network encrypted; may take hours for large snapshots.

## Data Pump Exports

For logical portability (schema-only or table subsets):

```sql
-- Master user runs impdp/expdp via DBMS_DATAPUMP
DECLARE
  h1 NUMBER;
BEGIN
  h1 := DBMS_DATAPUMP.OPEN('EXPORT','SCHEMA',NULL,'app_export');
  DBMS_DATAPUMP.ADD_FILE(h1,'app.dmp','DATA_PUMP_DIR');
  DBMS_DATAPUMP.ADD_FILE(h1,'app.log','DATA_PUMP_DIR', DBMS_DATAPUMP.KU$_FILE_TYPE_LOG_FILE);
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
    p_s3_prefix      => 'exports/prd/2026-08-06/',
    p_directory_name => 'DATA_PUMP_DIR');
END;
/
```

Track the S3 upload task:

```sql
SELECT text FROM TABLE(rdsadmin.rds_file_util.read_text_file('BDUMP','dbtask-<task_id>.log'));
```

Clean up:

```sql
BEGIN
  UTL_FILE.FREMOVE('DATA_PUMP_DIR', 'app.dmp');
  UTL_FILE.FREMOVE('DATA_PUMP_DIR', 'app.log');
END;
/
```

## Retention Strategy

| Objective               | Recommendation                                        |
| ----------------------- | ----------------------------------------------------- |
| Development / testing   | 7-day PITR                                            |
| Production internal ops | 14–30 day PITR                                        |
| Compliance (SOX 7-year) | 14-day PITR + monthly manual snapshot, exported to S3 |
| Regulatory (HIPAA, PCI) | Cross-region copy + retention per regulation          |

## Restore Testing

Once a quarter minimum:

1. Restore latest snapshot to test instance.
2. Verify data integrity (row counts, checksums on critical tables).
3. Time the operation → your RTO.
4. Compare object counts vs source.

```bash
# Automated restore-test
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier prd-restore-test \
    --db-snapshot-identifier <latest-snap> \
    --db-instance-class db.r6i.large \
    --publicly-accessible false

# Verify
sqlplus admin@endpoint <<EOF
SELECT COUNT(*) FROM app.orders;
SELECT MAX(order_date) FROM app.orders;
EOF

# Cleanup
aws rds delete-db-instance --db-instance-identifier prd-restore-test --skip-final-snapshot
```

## Cost

- Automated backup up to your DB size: free.
- Beyond: S3-backed storage cost per GB-month.
- Manual snapshots: per GB-month always billed.
- Cross-region: additional data transfer + storage.

## Related

- [RDS Architecture](rds-architecture.md).
- [RDS Backups (Cloud chapter)](../29-cloud/aws-rds/backups.md).
- [Data Pump](../21-data-pump/index.md).
