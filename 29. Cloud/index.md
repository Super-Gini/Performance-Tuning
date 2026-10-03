# Cloud

Running Oracle Database in cloud environments — the offerings, tradeoffs, and operational realities. Four broad hosting models:

| Model                                        | Control     | Cost    | Oracle Support       |
| -------------------------------------------- | ----------- | ------- | -------------------- |
| **BYOL on IaaS** (AWS EC2, Azure VM, GCP CE) | Full DBA    | Lowest  | Full                 |
| **Managed RDBMS** (AWS RDS Oracle)           | Limited DBA | Medium  | Reduced              |
| **Oracle DBCS / ExaCS / Autonomous**         | Turn-key    | Higher  | Full, Oracle-managed |
| **Autonomous**                               | None        | Highest | Fully Oracle-managed |

## Contents

- **AWS EC2 (BYOL)** — [Overview](aws-ec2/overview.md), [Backups](aws-ec2/backup-strategy.md), [Filesystem Layout](aws-ec2/filesystem-layout.md)
- **AWS RDS Oracle** — [Overview](aws-rds/overview.md), [Backups](aws-rds/backups.md), [Performance](aws-rds/performance.md), [Troubleshooting](aws-rds/troubleshooting.md)
- **OCI DBCS** — [Overview](oci-dbcs/overview.md)
- **Autonomous DB** — [Overview](autonomous-db/overview.md)
- **Exadata Cloud** — [Overview](exadata-cloud/overview.md), [Storage Cells](exadata-cloud/storage-cells.md), [Celldisk/Griddisk](exadata-cloud/celldisk-griddisk.md), [Smart Scan](exadata-cloud/smart-scan.md), [Monitoring](exadata-cloud/monitoring.md)

## The Big Choices

- **BYOL vs License-Included**: BYOL uses your existing Oracle licenses; License-Included pays by hour. Rule of thumb: BYOL if you already own EE; License-Included if starting fresh at moderate scale.
- **RDS vs EC2**: RDS = less to manage but no shell, limited RMAN, no OS access. EC2 = full DBA control.
- **OCI-native**: Best Oracle integration (Data Guard native, ExaCS with real Exadata storage cells) but organizational preference for AWS/Azure often prevents it.

## Related

- [RDS for Oracle](../33-rds-for-oracle/index.md).
- [Migrations](../38-migrations/index.md).
- [Zero Downtime Migration](../38-migrations/zero-downtime-migration.md).
