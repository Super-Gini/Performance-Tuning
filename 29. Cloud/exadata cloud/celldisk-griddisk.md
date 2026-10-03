# Exadata Cell Disks & Grid Disks

## The Stack

Physical disks in a cell → **Cell Disks** → **Grid Disks** → ASM Diskgroups.

```
Physical NVMe (12 per cell)
    ↓ (managed as)
Cell Disks (one per physical disk)
    ↓ (partitioned into)
Grid Disks (typically 2-3 per Cell Disk)
    ↓ (presented over IB to compute)
ASM Diskgroups (+DATAC1, +RECOC1, etc.)
    ↓
Oracle Database
```

## Cell Disks

One per physical disk (or per PMEM/flash device). Created at cell provisioning:

```bash
CellCLI> LIST CELLDISK
CD_00_cellhost1  CD  normal
CD_01_cellhost1  CD  normal
...
FD_00_cellhost1  FD  normal   # flash disk
FD_01_cellhost1  FD  normal
```

`CD_` = disk / hard cell disk, `FD_` = flash cell disk.

## Grid Disks

Slices of Cell Disks presented to ASM. Multiple Grid Disks per Cell Disk let you separate DGs:

```bash
CellCLI> LIST GRIDDISK
DATAC1_CD_00_cellhost1  active
DATAC1_CD_01_cellhost1  active
RECOC1_CD_00_cellhost1  active
RECOC1_CD_01_cellhost1  active
...
```

Naming convention: `<DGName>_<CellDiskName>`.

## Diskgroups on Exadata

Standard setup:

- `+DATAC1` — datafiles.
- `+RECOC1` — FRA, archives, backups.
- `+SPARSE` — for snapshot databases (optional).

Redundancy on Exadata Cloud is **HIGH** by default (3-way ASM mirror), which requires ≥ 3 cells.

## Space Math

Base X10M has 3 cells × ~64 TB each = 192 TB raw.

With HIGH redundancy:

- Usable ≈ 192 / 3 = 64 TB.
- Split: 80% DATAC1, 20% RECOC1.

## Grid Disk Ops

Extending storage — larger Grid Disks:

```bash
# Offline the disk, resize, online
CellCLI> ALTER GRIDDISK DATAC1_CD_00_cellhost1 INACTIVE
CellCLI> ALTER GRIDDISK DATAC1_CD_00_cellhost1 SIZE=60T
CellCLI> ALTER GRIDDISK DATAC1_CD_00_cellhost1 ACTIVE
```

ASM auto-rebalances.

Adding a cell — Oracle handles most of this. Provision cell → integrate into IB fabric → create Grid Disks → ASM DGs auto-extend.

## Sparse Grid Disks

Optional — used for **Snapshot databases** (near-zero-copy clones):

```
CellCLI> CREATE GRIDDISK SPARSE_CD_00_cellhost1 SPARSE, SIZE=100G ...
```

Then `+SPARSE` diskgroup. Enable Snapshot databases:

```sql
CREATE PLUGGABLE DATABASE dev_pdb FROM prd_pdb SNAPSHOT COPY;
```

Uses copy-on-write for near-instant clones.

## Monitoring

```sql
-- ASM side
SELECT   dg.name diskgroup, COUNT(*) disks
FROM     v$asm_diskgroup dg JOIN v$asm_disk d
                ON d.group_number = dg.group_number
GROUP BY dg.name;

-- Cell side
!dcli -c c1,c2,c3 "cellcli -e list griddisk attributes name,status,size where status <> 'active'"
```

## Related

- [Overview](overview.md).
- [Storage Cells](storage-cells.md).
- [ASM Architecture](../../19-asm/asm-architecture.md).
