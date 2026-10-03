# RMAN — Recovery Manager

**RMAN (Recovery Manager)** is the utility Oracle ships for backup and recovery. It knows the internal structure of datafiles, controlfiles, spfiles, and archive logs; it can back them up online, restore them safely, and drive point-in-time recovery — all while tracking what's on disk / tape and coordinating with the control file or a recovery catalog.

If you take away one rule from this section: **RMAN is the only supported backup mechanism for a production Oracle database.** OS-level file copies are a legacy option that survives for niche cases; RMAN is what you should use every day.

## Contents

| Page                                                  | Purpose                                             |
| ----------------------------------------------------- | --------------------------------------------------- |
| [RMAN Architecture](rman-architecture.md)             | Target, catalog, channels, backup sets, pieces      |
| [Backup Strategy](backup-strategy.md)                 | Full, incremental, block change tracking, retention |
| [Recovery Catalog](recovery-catalog.md)               | Central catalog database                            |
| [Control File Repository](control-file-repository.md) | Using the control file when there's no catalog      |
| [Recovery](recovery.md)                               | Full database recovery flow                         |
| [Restore](restore.md)                                 | Restoring datafiles, controlfiles, spfile           |
| [PITR](pitr.md)                                       | Point-in-Time Recovery — DBPITR                     |
| [TSPITR](tspitr.md)                                   | Tablespace Point-in-Time Recovery                   |
| [Duplicate Database](duplicate-database.md)           | Cloning production for dev / DR                     |
| [Block Media Recovery](block-media-recovery.md)       | `RECOVER BLOCK ...` for corruption                  |

## Related

- [Archive Logs](../04-storage/archive-logs.md) — recovery depends on these.
- [Flashback](../16-flashback/index.md) — alternative to RMAN for some scenarios.
- [Data Guard](../17-data-guard/index.md) — DG uses RMAN for standby creation.

## The RMAN Method

1. **Understand your RPO/RTO** — how much data can you lose, how long can you be down?
2. **Design retention** based on those numbers.
3. **Enable ARCHIVELOG mode**.
4. **Configure the FRA** (Fast Recovery Area).
5. **Set persistent RMAN configuration** (parallelism, retention, encryption).
6. **Automate** — daily incremental, weekly full, archive log backups every 2h.
7. **Test recovery** — quarterly at minimum, or your backups are Schrödinger's data.

Nothing above is optional.
