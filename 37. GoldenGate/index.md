# Oracle GoldenGate

**Oracle GoldenGate (OGG)** is Oracle's logical, log-based **change data capture (CDC) and replication** platform. It reads source database redo, transforms captured changes into a compact "trail file" format, ships it over TCP, and applies to the target database via SQL. Supports Oracle-to-Oracle, Oracle↔Postgres, Oracle↔Kafka, and many combinations.

## When to Use GoldenGate

- **Near-zero-downtime migration** (Oracle → RDS Oracle, Oracle → OCI, Oracle 12.2 → 19c).
- **Heterogeneous replication** (Oracle → PostgreSQL for analytics).
- **Active-active configuration** (multi-master, conflict-resolved).
- **Data warehouse offload** (OLTP → Kafka → warehouse).
- **Bi-directional cross-region for local reads**.

## Not for

- **Physical DR** — use Data Guard.
- **HA** — use RAC / Data Guard.
- **Single-tier point solution** — Data Pump for one-shot bulk.

## Contents

| Page                                  | Purpose                           |
| ------------------------------------- | --------------------------------- |
| [Architecture](architecture.md)       | Extract → trail → pump → Replicat |
| [Extract](extract.md)                 | Capture side                      |
| [Replicat](replicat.md)               | Apply side                        |
| [Troubleshooting](troubleshooting.md) | Common GG issues                  |

## Versions

- **Classic Extract** — legacy log mining from redo/archives.
- **Integrated Extract** — 11.2.0.3+, uses `DBMS_LOGMNR` inside the source DB. Recommended.
- **Integrated Replicat** — parallel apply orchestrated inside target DB.
- **Microservices Architecture (MA)** — 12.3+, web-managed. Newer sites use this.

## Licensing

Full-featured **GoldenGate for Oracle** is a separate license. **GoldenGate for Big Data** for Kafka etc. is a separate license too.

## Related

- [Data Guard](../17-data-guard/index.md) — physical alternative for DR.
- [Data Pump](../21-data-pump/index.md) — bulk companion.
- [Zero Downtime Migration](../38-migrations/zero-downtime-migration.md).
- [Migration Methods](../23-upgrade-migration/migration-methods.md).
