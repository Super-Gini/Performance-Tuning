# Diskgroups

## Overview

A **diskgroup** is ASM's abstraction over a set of raw block devices. Files (datafiles, control files, redo logs, backups, archive logs) reference a diskgroup by name (`+DATA/...`). ASM handles striping, mirroring, and rebalancing transparently.

Most production RAC / GI deployments have at least: `+DATA` (datafiles + control files + online redo), `+RECO` (FRA, archives, flashback), plus optional `+CRS` (OCR + voting disks — often combined with DATA) and `+REDO` for redo isolation.

## Create a Diskgroup

```sql
-- Connect to ASM
SQL> CONNECT / AS SYSASM

SQL> CREATE DISKGROUP data NORMAL REDUNDANCY
       FAILGROUP fg1 DISK '/dev/asm/data01', '/dev/asm/data02'
       FAILGROUP fg2 DISK '/dev/asm/data03', '/dev/asm/data04'
       ATTRIBUTE 'compatible.asm' = '19.0.0.0',
                 'compatible.rdbms' = '19.0.0.0',
                 'au_size' = '4M';
```

Key attributes:

- **`compatible.asm`** — feature version floor.
- **`compatible.rdbms`** — minimum DB version that can use.
- **`au_size`** — allocation unit size (1M / 4M / 16M / 64M).
- **`content.check`** — background content verification (18c+).

## Add / Drop Disks

```sql
-- Add
ALTER DISKGROUP data ADD DISK '/dev/asm/data05' NAME data_0005;
ALTER DISKGROUP data ADD DISK '/dev/asm/data06' NAME data_0006 REBALANCE POWER 8;

-- Drop
ALTER DISKGROUP data DROP DISK data_0002;

-- Force drop
ALTER DISKGROUP data DROP DISK data_0002 FORCE;
```

Adding disks triggers a **rebalance** — ASM redistributes data across all disks, including new. See [Rebalance](rebalance.md).

## Mount / Dismount

```sql
ALTER DISKGROUP data MOUNT;
ALTER DISKGROUP data DISMOUNT;

-- ALL
ALTER DISKGROUP ALL MOUNT;
```

Auto-mounted at ASM instance startup based on `ASM_DISKGROUPS`.

## Drop Diskgroup

```sql
-- Must be dismounted first
ALTER DISKGROUP data DISMOUNT;

-- Then
DROP DISKGROUP data INCLUDING CONTENTS;
```

!!! danger
Deletes all contents. Ensure backups.

## Space Reporting

```sql
SELECT name, state, type, total_mb/1024 AS total_gb,
       free_mb/1024 AS free_gb, usable_file_mb/1024 AS usable_gb,
       ROUND((1 - free_mb/total_mb) * 100, 1) AS pct_used
FROM   v$asm_diskgroup
ORDER  BY name;
```

- **`total_mb`** — raw capacity across all disks.
- **`free_mb`** — physically free.
- **`usable_file_mb`** — free space accounting for redundancy overhead (**this is the number to watch**).

For NORMAL redundancy: usable ≈ (free - largest failgroup size) / 2. HIGH: divide by 3.

## Add More Redundancy

Cannot change from EXTERNAL to NORMAL/HIGH in place. Must:

1. Create new diskgroup with desired redundancy.
2. Move files over.
3. Drop old diskgroup.

## ASM Compatibility

Set via `compatible.asm` and `compatible.rdbms`:

```sql
ALTER DISKGROUP data SET ATTRIBUTE 'compatible.asm' = '19.0.0.0';
ALTER DISKGROUP data SET ATTRIBUTE 'compatible.rdbms' = '19.0.0.0';
```

Higher `compatible` enables newer features; but locks out older clients. One-way.

## Diagnostic Queries

```sql
-- Diskgroup composition
SELECT dg.name, COUNT(*) AS disks,
       ROUND(SUM(d.total_mb)/1024, 1) AS total_gb,
       ROUND(SUM(d.free_mb)/1024, 1) AS free_gb
FROM   v$asm_diskgroup dg JOIN v$asm_disk d ON d.group_number = dg.group_number
GROUP  BY dg.name;

-- Attributes
SELECT dg.name, a.name, a.value
FROM   v$asm_attribute a JOIN v$asm_diskgroup dg ON dg.group_number = a.group_number
WHERE  a.name IN ('compatible.asm','compatible.rdbms','au_size','disk_repair_time')
ORDER  BY dg.name, a.name;

-- Client databases
SELECT c.instance_name, dg.name AS diskgroup, c.status
FROM   v$asm_client c JOIN v$asm_diskgroup dg ON dg.group_number = c.group_number
ORDER  BY c.instance_name;

-- Files
SELECT f.type, COUNT(*) AS n, ROUND(SUM(f.bytes)/1024/1024/1024, 1) AS gb
FROM   v$asm_file f JOIN v$asm_diskgroup dg ON dg.group_number = f.group_number
WHERE  dg.name = 'DATA'
GROUP  BY f.type
ORDER  BY gb DESC;
```

## Common Issues

- **`ORA-15040`** — diskgroup incomplete; missing disks.
- **`ORA-15041`** — diskgroup space exhausted; add or clean.
- **`ORA-15130`** — offline disk; check hardware.
- **`ORA-15042`** — mounted disks < required for redundancy.
- **Rebalance slow** — Adjust `ASM_POWER_LIMIT`.

## Best Practices

1. **Same-size disks** in a diskgroup.
2. **Two failure groups minimum** for NORMAL, three for HIGH.
3. Set `compatible` at creation to your version.
4. Standard names: `+DATA`, `+RECO`.
5. Monitor `USABLE_FILE_MB` — the real free space.
6. Alert at 85% used, 95% used.
7. Add disks in even sets matching failure group structure.
8. Rebalance during off-hours if power > 4.
9. Keep `disk_repair_time` reasonable (default 3.6h).
10. Never mix old and new disk generations without a rebalance plan.

## Interview Questions

1. **Q:** What is a diskgroup?
   **A:** Collection of raw disks that ASM stripes files across.

2. **Q:** Redundancy modes?
   **A:** EXTERNAL (none), NORMAL (2-way), HIGH (3-way), FLEX, EXTENDED.

3. **Q:** `usable_file_mb`?
   **A:** Real free space accounting for redundancy overhead — the number to monitor.

4. **Q:** Add a disk?
   **A:** `ALTER DISKGROUP data ADD DISK '/dev/asm/newdisk';` triggers rebalance.

5. **Q:** Drop diskgroup?
   **A:** Dismount, then `DROP DISKGROUP ... INCLUDING CONTENTS;`.

6. **Q:** Convert EXTERNAL to NORMAL?
   **A:** Not in place — create new, move files, drop old.

## References

- Oracle ASM Administrator's Guide 19c
- MOS Doc ID 265769.1 — ASM Overview
- MOS Doc ID 1523046.1 — ASM Best Practices
