# ASM — Interview Questions

**Q: What is Oracle ASM?**
A: **Automatic Storage Management** — Oracle's volume manager and filesystem for database files. Runs as a separate instance (`+ASM`) that manages diskgroups of raw block devices.

**Q: Why use ASM instead of a filesystem?**
A: Automatic striping across all disks, mirror-based redundancy at ASM level, online rebalance on add/remove, RAC-native shared storage, no filesystem overhead.

**Q: ASM redundancy levels?**
A: `EXTERNAL` (no mirror — rely on storage RAID), `NORMAL` (2-way), `HIGH` (3-way), `FLEX` (per-file redundancy), `EXTENDED` (stretched cluster).

**Q: What is a diskgroup?**
A: A collection of ASM disks presented as a single storage pool. Files (datafiles, redo, control, backups) are striped across all disks in the DG.

**Q: What are failure groups?**
A: Subsets of disks in a DG that share a failure boundary (rack, controller). ASM mirrors across failure groups so no single failure loses both copies.

**Q: What's ASM allocation unit (AU)?**
A: The base allocation size — default 1 MB (4 MB on Exadata). Files are made of multiple AUs distributed across disks in the DG.

**Q: What ASM background processes do you know?**
A: `RBAL` (coordinates rebalance), `ARBn` (rebalance slaves), `GMON` (diskgroup monitor), `MARK` (marks stale AUs), `PING` (peer detection in RAC).

**Q: How do you add a disk to a diskgroup?**
A: `ALTER DISKGROUP DATA ADD DISK '/dev/oracleasm/disks/NEWDISK' NAME DATA_0005;` — ASM auto-rebalances.

**Q: What is `POWER` in rebalance?**
A: Number of parallel `ARBn` slaves — 1 (slow) to 1024 (extreme). Higher = faster but more IO overhead.

**Q: What is ASM Filter Driver (AFD)?**
A: 12.1+ kernel-level filter that protects ASM disks from being mounted by other tools and provides consistent device identity. Modern replacement for ASMLIB.

**Q: How do you find which files use most space in a DG?**
A: `SELECT file_number, type, bytes/1024/1024 mb FROM v$asm_file WHERE group_number = &g ORDER BY bytes DESC;`

**Q: What happens if a disk fails in NORMAL redundancy?**
A: ASM continues serving from the mirror copy. If `DISK_REPAIR_TIME` (default 3.6h) elapses without the disk coming back, ASM drops it and rebalances remaining disks.

**Q: What is ACFS?**
A: **Automatic Cluster File System** — a general-purpose cluster filesystem on top of ASM. Used for non-database shared files (application binaries, GoldenGate trails).

**Q: Why is `+DATA` and `+RECO` a common layout?**
A: `+DATA` for datafiles + control files (frequently written), `+RECO` for FRA (backups + archives + flashback). Separation isolates IO patterns and simplifies snapshot backup.

**Q: How do you connect to the ASM instance?**
A: `sqlplus / as sysasm` (or `SYSDBA`, `SYSOPER`, `SYSASM` for admin ops). Environment variable `ORACLE_SID=+ASM` (or `+ASM1` in RAC).

**Q: Can you shrink a diskgroup?**
A: Yes — `ALTER DISKGROUP DATA DROP DISK 'DATA_0005';` triggers rebalance. If successful (enough free space on remaining disks), the disk is dropped.

**Q: RAC evictions — could a diskgroup issue cause one?**
A: Yes — if the voting disk is on ASM (typical) and the DG becomes unresponsive, CSSD can't heartbeat → eviction. That's why voting disks are on their own DG (`+CRS`) with HIGH redundancy.

## Related

- [ASM Architecture](../19-asm/asm-architecture.md).
- [ASM Filter Driver](../19-asm/asm-filter-driver.md).
- [ASM Rebalance](../19-asm/rebalance.md).
