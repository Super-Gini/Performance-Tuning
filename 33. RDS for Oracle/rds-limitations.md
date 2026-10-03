# RDS Oracle Limitations

## What You Can't Do

### Operating System

- **No SSH** to the DB host.
- **No shell access** — not even readonly.
- **No file system access** — read/write via `rdsadmin` procedures only.
- **No modification of OS parameters** — kernel params, `ulimit`, HugePages.
- **No custom OS packages** — you can't install anything on the host.

### Oracle Roles / Users

- **No SYSDBA / SYSOPER access** — the master user is not SYS.
- **Can't ALTER USER SYS** directly.
- **Can't modify SYS-owned objects**.
- **Can't grant DBA to arbitrary users** (some restrictions).

### Features Missing

- **Real Application Clusters (RAC)** — not available.
- **Data Guard** managed by you — RDS Multi-AZ is not Data Guard.
- **Oracle Golden Gate** — as a managed service, no; can install client-side.
- **Automatic Storage Management (ASM)** — invisible; RDS uses EBS.
- **DBFS** — no.
- **DATABASE VAULT** — no.
- **Fine-grained auditing** — limited.
- **Total Recall / Flashback Data Archive** — supported since 2020.
- **Real Application Testing** — supported since 2019.

### Features Restricted

- **RMAN** — very limited via `rdsadmin.rdsadmin_rman_util`. Can back up controlfile / archivelogs on demand; full backups are RDS-managed.
- **Native NEEDED features via `rdsadmin`**:
  - Kill session (`rdsadmin_util.kill`).
  - Flush shared pool (`rdsadmin_util.flush_shared_pool`).
  - Rebuild indexes online (standard SQL works).
  - Grants on SYS objects (`rdsadmin_util.grant_sys_object`).
  - Character set change (`rdsadmin_util.alter_default_charset`).
  - Add data file to a tablespace (`rdsadmin_util.add_datafile`).

### Init Parameters Restricted

Not exposed via parameter groups:

- Memory sizing (`sga_target`, `memory_target`) — set by instance class.
- Some redo log parameters.
- Cluster / RAC parameters.
- Underscore parameters (mostly).

Some are dynamic (immediate), some require reboot.

### Backup Restrictions

- **Can't run arbitrary RMAN** (limited via `rdsadmin_rman_util`).
- **Can't backup to your own S3 directly with RMAN** (use Data Pump + S3 upload).
- **Snapshots are not portable** — can't restore to EC2 or on-prem.

### Storage Restrictions

- Max storage: 64 TB (io2/gp3).
- Max IOPS: 256k (io2 Block Express variants).
- Can't shrink storage — only grow.
- Storage type changes possible but slow (backfill).

### Network

- Instance in AWS-managed VPC; you specify subnets.
- Public accessibility off by default (recommended).
- Custom TNS_ADMIN / listener.ora changes: not possible.
- Cross-account VPC peering: yes.
- Direct Connect / VPN: yes.

### Auditing

- Standard Oracle auditing works.
- Unified Auditing supported.
- FGA supported but with restrictions.
- Best to also enable CloudWatch Logs export.

## Common Workarounds

| Missing feature       | Workaround                              |
| --------------------- | --------------------------------------- |
| SYSDBA                | Master user + `rdsadmin` package        |
| Native RMAN backup    | Data Pump + S3 upload                   |
| Shell OS metrics      | Enhanced Monitoring in CloudWatch       |
| Custom patches        | Not possible — pick a supported version |
| Data Guard            | Read replica (uses DG under covers)     |
| RAC                   | Migrate to OCI or EC2                   |
| ASM                   | Trust EBS + gp3/io2                     |
| Trace file access     | `aws rds download-db-log-file-portion`  |
| Custom init parameter | Parameter group (if exposed)            |

## When Limitations Force You Off RDS

Move to EC2 (BYOL) if:

- Need RAC.
- Need SYS-level access.
- Need to install third-party tools alongside DB.
- Need Data Guard native (with broker, real-time apply, SYNC control).
- Data > 40 TB.
- IOPS ceiling would bite.

Migrate via:

- **AWS DMS** for near-zero downtime.
- **Data Pump network mode** for cutover.
- **Snapshot + import** for cold migrations.

See [Migration Methods](../23-upgrade-migration/migration-methods.md).

## Related

- [RDS Architecture](rds-architecture.md).
- [Overview (Cloud chapter)](../29-cloud/aws-rds/overview.md).
- [EC2 as alternative](../29-cloud/aws-ec2/overview.md).
