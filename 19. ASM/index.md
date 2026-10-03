# Automatic Storage Management (ASM)

**Oracle ASM** is Oracle's volume manager and filesystem for database files. It stripes files across a **diskgroup** of raw devices, provides **redundancy** (mirroring at ASM level), rebalances online when disks are added or removed, and integrates directly with Oracle for backup, cloning, and RAC shared storage.

ASM ships as part of **Grid Infrastructure** — you don't buy it separately. It runs as its own instance (`+ASM`) on each node.

## Contents

| Page                                      | Purpose                                  |
| ----------------------------------------- | ---------------------------------------- |
| [ASM Architecture](asm-architecture.md)   | ASM instance, diskgroups, files, ACFS    |
| [Diskgroups](diskgroups.md)               | Creating, expanding, dropping diskgroups |
| [Failure Groups](failure-groups.md)       | Redundancy and failure isolation         |
| [Allocation Units](allocation-units.md)   | AU size and file layout                  |
| [ASMCMD](asmcmd.md)                       | Command-line utility                     |
| [ASM Filter Driver](asm-filter-driver.md) | Kernel filter for device isolation       |
| [Rebalance](rebalance.md)                 | Online rebalance operations              |
| [Monitoring](monitoring.md)               | ASM alert log, `V$ASM_*` views           |

## Related

- [RAC](../18-rac/index.md) — ASM provides shared storage.
- [Datafiles](../04-storage/datafiles.md) — where they live in ASM.
- [Bigfile Tablespaces](../04-storage/bigfile-tablespaces.md) — natural fit for ASM.
