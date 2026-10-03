# OCI Database Cloud Service (DBCS) — Overview

## Model

Oracle's own managed database service on Oracle Cloud Infrastructure. Full DBA-visible Oracle experience with Oracle-run infrastructure. Available as **VM DB Systems** (single instance or 2-node RAC) or **Bare Metal DB Systems**.

## Editions

- **Standard** — SE2.
- **Enterprise Edition (EE)** — includes ADG, TDE, etc.
- **Enterprise Edition High Performance** — EE + Advanced Security, DB Vault, Advanced Comp, RAC.
- **Enterprise Edition Extreme Performance** — Highest tier, RAC + Multitenant + full pack.

## Shapes

| Shape                   | Type       | vCPUs / RAM        | Notes                        |
| ----------------------- | ---------- | ------------------ | ---------------------------- |
| `VM.Standard*.Flex`     | Virtual    | 1–64 / 16 GB–1 TB  | Most flexible                |
| `BM.DenseIO2.52`        | Bare metal | 52 / 768 GB / NVMe | High IO for OLTP             |
| `Exadata Cloud Service` | Bare metal | Real Exadata cells | Best perf for large workload |

## What OCI Manages

- Underlying compute + storage.
- Node OS patches.
- DB software installs (you pick versions).
- Backup service (with policy).
- Fast Application Notification.

## What You Manage

- Init parameters (full Oracle SPFILE).
- Schemas, users, DDL.
- Data Guard (via OCI console).
- RMAN backups (or use OCI-managed).
- Monitoring integration.

## Getting Started

```bash
# Create via OCI CLI
oci db system launch \
    --availability-domain <ad> \
    --compartment-id <ocid> \
    --shape "VM.Standard2.4" \
    --db-version "19.0.0.0" \
    --hostname "prd-db01" \
    --ssh-authorized-keys-file ~/.ssh/oci.pub \
    --subnet-id <subnet_ocid> \
    --db-name "PRD" \
    --admin-password "<pwd>"
```

## Backup

Two options:

- **Managed automatic** — daily to Object Storage, PITR retention 7–60 days.
- **RMAN** — you run it via SSH-to-DB.

Managed backup config:

```bash
oci db backup create --database-id <db_ocid> --display-name "manual-prep-release"
```

## Data Guard

OCI has Data Guard as a first-class feature — one-click enable, uses another VM DB System in a different AD/region as standby.

## RAC

Available on 2-node VM DB Systems and higher. Fully managed shared storage, VNIC config, private interconnect.

## Networking

- **VCN** — Virtual Cloud Network.
- **Private subnets** for DB.
- **Service gateway** for Object Storage backup.
- **FastConnect** for on-prem hybrid.

## Advantages Over AWS RDS Oracle

- SYSDBA access.
- Full RMAN.
- Full Data Guard.
- RAC.
- Full trace file access.
- License-Included available in all editions.
- Latest RUs available faster than AWS.

## Disadvantages Over AWS RDS

- Fewer regions than AWS.
- Fewer integrations with non-Oracle apps.
- Learning curve if your team is AWS-centric.

## When to Choose

- Need RAC in cloud → OCI DBCS or ExaCS.
- Need Data Guard with switchover control → OCI or EC2.
- OCI-Zero-Downtime Migration is easier than to AWS.
- Enterprise Oracle features (Vault, TDE, etc.) needed at scale.

## Related

- [Autonomous DB](../autonomous-db/overview.md) — the fully-Oracle-managed alternative.
- [Exadata Cloud](../exadata-cloud/overview.md) — for Exadata-scale.
- [Zero Downtime Migration](../../38-migrations/zero-downtime-migration.md).
