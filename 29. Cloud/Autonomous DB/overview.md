# Oracle Autonomous Database — Overview

## Model

**Autonomous Database (ADB)** is Oracle's self-driving, self-securing, self-repairing database. You provide the schema and SQL; Oracle handles everything else — provisioning, tuning, patching, scaling, backups, security, recovery.

Two flavors:

- **Autonomous Data Warehouse (ADW)** — DW workloads.
- **Autonomous Transaction Processing (ATP)** — OLTP workloads.

Sub-flavors:

- **Shared Infrastructure** — Multi-tenant Oracle cloud.
- **Dedicated Infrastructure** — Your own Exadata rack.

## What You Get

- Fully-provisioned Oracle 19c or 23ai.
- SQL Developer Web / APEX / OML built in.
- Automatic tuning: SPB, indexes (Automatic Indexing on ATP).
- Automatic backups + PITR.
- Automatic patching in scheduled windows.
- TLS-required connectivity.
- Autoscaling — CPU up to 3× baseline on demand.
- Full standby (Autonomous Data Guard) with one click.

## What You Give Up

- **SYSDBA / OS access** — none.
- Init parameter control — most are locked.
- Manual RMAN — replaced by managed backups.
- Direct file access.
- Custom triggers on SYS objects.
- Some Oracle features (Real Application Testing, Diagnostic Pack API — some are Autonomous-only).

## Sizing

- Provision by **OCPU** (one Oracle CPU = ~2 vCPUs).
- Storage in TB.
- Autoscale multiplies OCPUs during load bursts (charged per second).

## Backup

Automatic every 60 seconds — PITR to any second within retention (default 60 days).

Manual backups on demand.

## HA

- **99.95% SLA** on ADB-Shared.
- **Autonomous Data Guard** — one-click standby in another region, ~15 s failover.

## Access

Via **wallet-based** TLS:

```bash
# Download wallet from OCI console
unzip wallet.zip -d /home/oracle/adw_wallet

# Set env
export TNS_ADMIN=/home/oracle/adw_wallet

# Connect
sqlplus admin/<pw>@adw_high
```

Service names:

- `<db>_high` — high concurrency.
- `<db>_medium` — balanced.
- `<db>_low` — many concurrent, low resource each.
- `<db>_tp` — OLTP.
- `<db>_tpurgent` — highest priority TP.

## Tools Included

- SQL Developer Web.
- APEX.
- ORDS (Oracle REST Data Services).
- Oracle Machine Learning (OML) notebooks.
- Graph Studio.

## When to Choose Autonomous

- Green-field applications.
- Small team, no DBA.
- Workload well-served by Oracle SQL.
- You want cost-per-usage without hardware provisioning.

## When Not To

- Regulatory environment requiring specific security controls.
- Legacy custom PL/SQL using undocumented features.
- Very large workloads requiring specific tuning.
- On-prem or non-OCI cloud is mandatory.

## Migration to Autonomous

- **Data Pump** — impdp with `network_link` (fastest).
- **AWS DMS + native ingest** — for AWS-to-Autonomous.
- **GoldenGate** — near-zero downtime.
- **ZDM** — Oracle's automated tool.

## Related

- [DBCS Overview](../oci-dbcs/overview.md) — full-DBA alternative in OCI.
- [Migration Methods](../../23-upgrade-migration/migration-methods.md).
- [Zero Downtime Migration](../../38-migrations/zero-downtime-migration.md).
