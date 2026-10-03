# AWS RDS for Oracle — Overview

## Model

Managed Oracle Database service on AWS. You get a running Oracle instance without shell access. AWS manages the host OS, patching (with your approval), backups, and HA.

## What You Get

- Provisioned instance (`db.r6i.4xlarge` etc.).
- Storage — EBS-backed, `gp3` / `io1` / `io2`.
- Automated backups (up to 35 days point-in-time recovery).
- Multi-AZ high availability (synchronous replication to standby).
- Read replicas (async).
- Automated minor version patching (opt-in).
- CloudWatch integration.
- Parameter groups (subset of init parameters).

## What You Don't Get

- Shell / SSH — none.
- Direct file access — no `ALTER DATABASE ... DATAFILE`.
- SYSDBA / SYSOPER outside AWS-provided masters.
- Data Guard managed by you (RDS provides its own HA).
- RMAN (mostly — limited via `rdsadmin` PL/SQL).
- Full trace file access (some via CloudWatch logs).
- Real Application Clusters.
- Advanced Compression Options / TDE — via License-Included versions only.

## Editions Available

- SE2 (BYOL or License-Included).
- EE (BYOL only in most regions).

## Instance Classes

Prefixed with `db.`:

- `db.r6i.*` — memory-optimized.
- `db.m6i.*` — general purpose.
- `db.r5b.*` — high-EBS-throughput.
- `db.x1e.*` — massive memory (SAP-scale).

## Storage

| Type  | Notes                                 |
| ----- | ------------------------------------- |
| `gp3` | Baseline, adjustable IOPS/throughput. |
| `io1` | Legacy provisioned IOPS.              |
| `io2` | Current provisioned IOPS.             |

Max storage: 64 TB. Max IOPS: 256,000 (io2).

## Access Pattern

You interact via:

- **Oracle client** (sqlplus, sqldeveloper) via the RDS endpoint.
- **`rdsadmin` package** for admin ops (kill session, purge audit trail, add tablespace).
- **AWS API/CLI** for infra ops (modify instance, snapshot, etc.).

Example `rdsadmin` calls:

```sql
-- Kill session (no ALTER SYSTEM privilege)
EXEC rdsadmin.rdsadmin_util.kill(150, 32458);

-- Add data file
EXEC rdsadmin.rdsadmin_util.add_datafile(
       tablespace_name => 'USERS',
       file_name       => 'users02.dbf',
       size_bytes      => 10737418240);   -- 10 GB

-- Grant SYSDBA-like operations (won't literally grant SYSDBA)
EXEC rdsadmin.rdsadmin_util.grant_sys_object(
       p_obj_name  => 'V_$SESSION',
       p_grantee   => 'APP',
       p_privilege => 'SELECT');
```

Full `rdsadmin` reference: MOS + AWS docs.

## HA — Multi-AZ

Enable at create time. AWS provisions a **synchronous replica** in another AZ. On failure, DNS name flips to the standby — typical failover 60–120 s. No manual intervention.

**Not the same as Oracle Data Guard**. AWS uses a proprietary block-level replication under the covers.

## Read Replicas

Cross-region / cross-AZ read-only copies. Asynchronous. Limited to a few per master.

## Backups

- **Automated backups** — daily snapshot + transaction logs → point-in-time recovery up to 35 days.
- **Manual snapshots** — you take on demand, retained until you delete.
- **Storage of backups** — S3-backed, transparent.

See [Backups](backups.md).

## Migration to RDS

- **AWS DMS** — logical, near-zero downtime.
- **Data Pump** — impdp from S3 via `rdsadmin`.
- **Restore from snapshot** — if source is also RDS.
- **Custom TDE keystore** — supported now via Managed AD or Encrypted BYOK.

See [AWS DMS](../../38-migrations/aws-dms.md).

## When RDS Isn't Enough

- Data Guard native replication (RDS's HA isn't Data Guard).
- RAC.
- Very-large DB (multi-tens of TB) where io2 caps become limiting.
- Custom patches / one-offs — RDS applies only AWS-approved patches.

Fall back to EC2 or ExaCS.

## Sub-pages

| Page                                  | Purpose                       |
| ------------------------------------- | ----------------------------- |
| [Backups](backups.md)                 | Snapshot + PITR               |
| [Performance](performance.md)         | Tuning within RDS constraints |
| [Troubleshooting](troubleshooting.md) | Common RDS problems           |

## Related

- [RDS for Oracle](../../33-rds-for-oracle/index.md) — deeper coverage.
- [AWS DMS](../../38-migrations/aws-dms.md).
- [EC2 Overview](../aws-ec2/overview.md) — the unmanaged alternative.
