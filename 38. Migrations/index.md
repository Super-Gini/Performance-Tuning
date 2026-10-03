# Migrations

Focused reference for the migration methods DBAs run in real projects. Companion to [Upgrade & Migration → Migration Methods](../23-upgrade-migration/migration-methods.md) which gives the high-level chooser.

## Contents

| Page                                                      | Purpose                         |
| --------------------------------------------------------- | ------------------------------- |
| [Transportable Tablespaces](transportable-tablespaces.md) | TTS + XTTS                      |
| [Cross-Platform Migration](cross-platform-migration.md)   | Endian conversion, RMAN CONVERT |
| [Zero Downtime Migration](zero-downtime-migration.md)     | Oracle ZDM tool                 |
| [AWS DMS](aws-dms.md)                                     | AWS Database Migration Service  |

## Rule of Thumb

- **< 500 GB, same platform** → Data Pump.
- **500 GB – 5 TB, same endian** → TTS.
- **> 5 TB, any endian** → XTTS incremental.
- **To OCI** → ZDM.
- **To AWS RDS** → DMS + Data Pump.
- **Zero downtime any target** → GoldenGate.

## Related

- [Migration Methods (chooser)](../23-upgrade-migration/migration-methods.md).
- [Data Pump](../21-data-pump/index.md).
- [GoldenGate](../37-goldengate/index.md).
- [DMS Migration Issues (case study)](../34-real-world-case-studies/dms-migration-issues.md).
