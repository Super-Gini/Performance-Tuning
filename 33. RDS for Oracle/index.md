# RDS for Oracle — Deeper Reference

Sequel to the [AWS RDS Oracle Overview](../29-cloud/aws-rds/overview.md). This chapter goes deeper into RDS Oracle specifics: internal architecture, backups, monitoring, and the practical limitations that shape day-to-day operations.

## Contents

| Page                                    | Purpose                                   |
| --------------------------------------- | ----------------------------------------- |
| [RDS Architecture](rds-architecture.md) | Under-the-hood RDS Oracle setup           |
| [RDS Backups](rds-backups.md)           | PITR, snapshots, Data Pump                |
| [RDS Monitoring](rds-monitoring.md)     | Performance Insights, Enhanced Monitoring |
| [RDS Limitations](rds-limitations.md)   | Things you can't do; workarounds          |

## When RDS Is the Right Choice

- Standard Oracle workload, no exotic features.
- Small ops team without deep Oracle DBA skill.
- 24/7 workload but no need for RAC or full Data Guard control.
- Multi-region DR via cross-region snapshots is acceptable.

## When It Isn't

- Need RAC.
- Need real Data Guard (SYNC + broker + fine control).
- Need SYS-level access, shell, custom patches.
- Data > ~40 TB (RDS storage cap approaching).
- IOPS ceiling would be an issue (256k io2).

Consider [EC2 (BYOL)](../29-cloud/aws-ec2/overview.md) or [OCI DBCS](../29-cloud/oci-dbcs/overview.md) instead.

## Related

- [AWS RDS Overview (from Cloud chapter)](../29-cloud/aws-rds/overview.md).
- [AWS DMS](../38-migrations/aws-dms.md) — for migrating to/from RDS.
