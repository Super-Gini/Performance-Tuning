# Failure Groups

## Overview

A **failure group** is a subset of disks in a diskgroup that share a common failure mode — same controller, same SAN array, same rack. ASM's redundancy places mirror copies across different failure groups, so a single failure (rack losing power, controller dying) doesn't lose the data.

If you don't specify failure groups, each disk is its own failure group. That's often fine — but for site-aware / rack-aware protection, explicit groups matter.

## Redundancy and Failure Group Minimums

| Redundancy      | Min FGs        | Mirror copies per file         |
| --------------- | -------------- | ------------------------------ |
| EXTERNAL        | 1              | 0 (delegates to storage array) |
| NORMAL          | 2              | 2                              |
| HIGH            | 3              | 3                              |
| FLEX (12.2+)    | 3              | Varies per file                |
| EXTENDED (18c+) | Multiple sites | Site-aware                     |

## Creating with Failure Groups

```sql
CREATE DISKGROUP data NORMAL REDUNDANCY
  FAILGROUP fg_rack1
    DISK '/dev/asm/rack1_disk1', '/dev/asm/rack1_disk2'
  FAILGROUP fg_rack2
    DISK '/dev/asm/rack2_disk1', '/dev/asm/rack2_disk2'
  ATTRIBUTE 'compatible.asm' = '19.0.0';
```

Each mirror copy lives on a different failure group. Losing all of `fg_rack1` still leaves data accessible via `fg_rack2`.

## Quorum Failure Group

For voting disks in even-node clusters, a **QUORUM FAILGROUP** provides a tiebreaker without holding user data:

```sql
CREATE DISKGROUP crs NORMAL REDUNDANCY
  FAILGROUP fg1 DISK '/dev/asm/crs1'
  FAILGROUP fg2 DISK '/dev/asm/crs2'
  QUORUM FAILGROUP fg3 DISK '/dev/asm/crs3'
  ATTRIBUTE 'compatible.asm' = '19.0.0';
```

The quorum group is voting-only; typically an NFS-mounted file on a third site.

## Site-Aware (Extended Cluster)

For stretched clusters across two data centers, use EXTENDED redundancy (18c+) with **site tagging**:

```sql
ALTER DISKGROUP data SET ATTRIBUTE 'compatible.asm' = '18.0.0';
ALTER DISKGROUP data SET ATTRIBUTE 'compatible.rdbms' = '18.0.0';

ALTER DISKGROUP data ADD SITE site_a;
ALTER DISKGROUP data ADD SITE site_b;

ALTER DISKGROUP data ADD FAILGROUP fg_a1 DISK '/dev/asm/a1' SITE site_a;
-- ...
```

ASM ensures at least one mirror per site.

## Diagnostic Queries

```sql
-- Failure groups per diskgroup
SELECT dg.name AS diskgroup, d.failgroup, d.failgroup_type,
       COUNT(*) AS disks
FROM   v$asm_diskgroup dg JOIN v$asm_disk d ON d.group_number = dg.group_number
GROUP  BY dg.name, d.failgroup, d.failgroup_type
ORDER  BY dg.name, d.failgroup;

-- Any failure group offline?
SELECT dg.name, d.failgroup, d.name AS disk, d.state, d.mode_status
FROM   v$asm_diskgroup dg JOIN v$asm_disk d ON d.group_number = dg.group_number
WHERE  d.state <> 'NORMAL' OR d.mode_status <> 'ONLINE'
ORDER  BY dg.name, d.failgroup;
```

## Common Issues

- **Missing failure group tolerance** — Loss of any 2 disks in same FG blocks the DG (NORMAL). Design FGs by physical failure domain.
- **Uneven failure group sizes** — Under NORMAL redundancy, wasted space in the larger group. Balance disk counts and sizes.
- **`ORA-15068`** — Redundancy insufficient (e.g., 3 disks in NORMAL — one failure group has only one disk).

## Best Practices

1. **Failure groups = physical failure domains** (rack, controller, JBOD).
2. Balance number of disks and disk sizes across failure groups.
3. NORMAL: minimum 2 FGs, prefer 3 for margin.
4. HIGH: minimum 3 FGs; prefer 4 or 5.
5. **Quorum FG** for extended clusters or when even FGs risk ties.
6. Use **site tagging** (EXTENDED redundancy) for stretched clusters.
7. Alert if any disk in any FG offline.
8. Do not use ASM redundancy on top of storage-array RAID unless you have specific reason — waste of space.
9. Document FG mapping to physical infrastructure.

## Interview Questions

1. **Q:** What is a failure group?
   **A:** Subset of disks in a diskgroup sharing a failure domain. ASM places mirrors on different failure groups.

2. **Q:** NORMAL redundancy minimum FGs?
   **A:** 2. HIGH: 3.

3. **Q:** Quorum failure group?
   **A:** Voting-only group for tiebreaker in extended / even-node configs.

4. **Q:** Extended clusters?
   **A:** ASM EXTENDED redundancy (18c+) with site-aware failure groups.

5. **Q:** If you don't specify FGs?
   **A:** Each disk is its own failure group.

## References

- Oracle ASM Administrator's Guide 19c — Failure Groups
- MOS Doc ID 1523046.1 — ASM Best Practices
- MOS Doc ID 2118589.1 — Extended Cluster
