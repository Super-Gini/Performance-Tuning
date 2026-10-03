# AWS EC2 — Filesystem Layout

## Standard Layout

Following OFA + Oracle best practice:

```
/                    -> root EBS (gp3, 30-50 GB)
/u01/app/oracle      -> ORACLE_BASE (gp3, 200 GB) - binaries, ADR
/u02/oradata         -> datafiles (io2, sized to data)
/u03/fra             -> Fast Recovery Area (gp3 or io2)
/u04/redo1           -> redo log group 1 (io2, small volume)
/u05/redo2           -> redo log group 2 (io2, small volume, different device)
```

Split across **separate EBS volumes** so IO isn't contended.

## EBS Volume Types

| Type                | Use for                     | Notes                                    |
| ------------------- | --------------------------- | ---------------------------------------- |
| `gp3`               | Binaries, FRA, non-critical | Cheap, up to 16k IOPS, 1 GB/s throughput |
| `io2`               | Datafiles, redo             | Consistent latency, up to 64k IOPS       |
| `io2 Block Express` | Very hot data               | Up to 256k IOPS, 4 GB/s                  |
| `st1`               | Streaming / DW warm data    | Throughput-optimized HDD                 |

**Recommendation**:

- Datafiles: `io2` sized for peak IOPS (peaks at 2–3× steady state).
- Redo: **separate** `io2` volume, dedicated. Redo latency is `log file sync` — must be low.
- FRA: `gp3` normally; escalate to `io2` if backups saturate.
- ADR / binaries: `gp3`, 200 GB is generous.

## Filesystem Choice

- **XFS** — recommended by Oracle for large filesystems.
- **ext4** — also supported. Mount with `noatime,nodiratime`.
- **ZFS** — supported on Oracle Linux 7+, more complex.
- **NFS** (from FSx for NetApp ONTAP, EFS) — supported per MOS Doc ID 359515.1, use only if you need shared storage (RAC).

Mount options:

```
/etc/fstab

/dev/xvdf   /u02/oradata   xfs   defaults,noatime,nodiratime,nofail 0 2
/dev/xvdg   /u03/fra       xfs   defaults,noatime,nofail 0 2
/dev/xvdh   /u04/redo1     xfs   defaults,noatime,nofail 0 2
/dev/xvdi   /u05/redo2     xfs   defaults,noatime,nofail 0 2
```

## LVM

Use LVM if you want to grow volumes across multiple EBS volumes:

```bash
pvcreate /dev/xvdf /dev/xvdg /dev/xvdh /dev/xvdi
vgcreate vg_oradata /dev/xvdf /dev/xvdg
lvcreate -L 500G -n lv_oradata vg_oradata
mkfs.xfs /dev/vg_oradata/lv_oradata
mount /dev/vg_oradata/lv_oradata /u02/oradata
```

**Warning**: Striping across EBS volumes multiplies IOPS but also blast radius. Consider ASM instead — Oracle handles striping intelligently.

## ASM on EC2

If you want ASM (recommended for larger deployments):

```bash
# Multiple EBS io2 volumes, one per DG
# Mount via device names or udev rules

# ASM Filter Driver setup
asmcmd afd_configure -init
asmcmd afd_label DISK1 /dev/xvdf
asmcmd afd_label DISK2 /dev/xvdg
```

Diskgroups:

- `+DATA` — datafiles.
- `+RECO` — FRA + archives.
- `+REDO` — dedicated (optional, if not using OS-level dedicated volumes).

External redundancy at ASM level (EBS handles durability). NORMAL redundancy doubles space but adds a mirror layer.

## HugePages

Set for the SGA:

```bash
# /etc/sysctl.conf
vm.nr_hugepages = 12800   # for SGA_TARGET=25G, page=2M => 25*1024/2 = 12800

# Actually apply
sysctl -p

# /etc/security/limits.conf
oracle soft memlock 26214400
oracle hard memlock 26214400
```

And in Oracle:

```sql
ALTER SYSTEM SET use_large_pages = 'ONLY' SCOPE=SPFILE;
```

## Snapshots

EBS snapshots capture the volume state — but **not** in a crash-consistent way for Oracle unless you `ALTER SYSTEM SUSPEND` first or use `BEGIN BACKUP`. For proper backup: use RMAN.

## Related

- [AWS EC2 Overview](overview.md).
- [Backup Strategy](backup-strategy.md).
- [ASM Architecture](../../19-asm/asm-architecture.md).
