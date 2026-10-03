# Voting Disk

## Overview

The **Voting Disk** (also called the **voting file**) is the tiebreaker for cluster membership. When a network partition splits the cluster into groups, each group writes its perceived membership to the voting disk. CSSD uses the recorded votes to decide which partition survives — the majority.

Voting disks live on **shared storage** — modern deployments use ASM, older ones used raw devices or block devices.

## Size and Count

- Each voting disk is small (~300 MB).
- Must be an **odd number** for majority voting: 1, 3, or 5.
- HIGH-redundancy ASM diskgroup automatically creates 5.
- NORMAL-redundancy: 3.
- EXTERNAL-redundancy: 1 (rely on storage-array redundancy).

## Where They Live

```bash
# Query
crsctl query css votedisk

# Sample output
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   1a2b3c4d5e6f7g8h9i0j1k2l3m4n5o6p (+CRS/orcl/vote/vote1.vote) [CRS]
 2. ONLINE   ...                              [CRS]
 3. ONLINE   ...                              [CRS]
```

## Quorum Failure Group

For even-node clusters (2, 4 nodes), a third **quorum failure group** provides the tiebreaker. Configure at ASM level:

```sql
CREATE DISKGROUP crs NORMAL REDUNDANCY
  FAILGROUP fg1 DISK '/dev/asm/crs01'
  FAILGROUP fg2 DISK '/dev/asm/crs02'
  QUORUM FAILGROUP fg3 DISK '/dev/asm/crs03'
  ATTRIBUTE 'compatible.asm' = '19.0.0';
```

Quorum failure group is used only for voting — not for data.

## Adding / Removing

Legacy syntax (raw/block devices):

```bash
crsctl add css votedisk /dev/asmvote2
crsctl delete css votedisk /dev/asmvote2
```

When voting disks are in ASM, they follow ASM diskgroup redundancy — you don't add/remove individually. Instead, adjust diskgroup redundancy.

Replace voting disks:

```bash
crsctl replace votedisk +DATA
```

Moves voting disks from current diskgroup to `+DATA`.

## Backup

**Not needed** — voting disks are stateless. In a disaster, recreate from an ASM restore.

## Failure Scenarios

- **All voting disks unreachable** — cluster hangs; all nodes eventually evict.
- **Majority of voting disks unreachable** — cluster hangs.
- **Minority of voting disks unreachable** — cluster continues; CSSD warns.

Redundancy at the ASM level provides resilience.

## Diagnostic Queries

```bash
# Voting disks
crsctl query css votedisk

# Voting disk usage
crsctl query css votedisk -a

# CSS state including voting disks
crsctl check css
crsctl check cluster -all

# Recent CSS activity
tail -100 $ORACLE_BASE/diag/crs/$(hostname -s)/crs/trace/ocssd.trc
```

## Common Issues

- **`CRS-1613: unable to write to voting file`** — Storage issue. Check ASM diskgroup and physical disks.
- **`CRS-1636: CSSD is not able to see enough voting disks`** — Loss of majority — cluster will evict.
- **`CRS-4406: Oracle CSSD cannot access voting disk`** — Path missing.
- **Adding voting disk fails** — Diskgroup redundancy mismatch; adjust.

## Best Practices

1. **Odd count** (1, 3, or 5).
2. **HIGH-redundancy ASM diskgroup** for OCR + voting.
3. Voting disks and OCR can share the same diskgroup, or split — either is acceptable.
4. **Third failure group** for even-node clusters.
5. Monitor `crsctl query css votedisk` regularly.
6. Alert on any `OFFLINE` voting disk.
7. Storage-level redundancy (RAID, ASM mirror) still recommended.
8. Practice storage failure in a lab — understand the eviction behavior.

## Interview Questions

1. **Q:** What does a voting disk do?
   **A:** Tiebreaker for CSS membership in network partitions. Larger group by votes survives.

2. **Q:** How many?
   **A:** Odd number: 1, 3, or 5. Modern: 5 in HIGH-redundancy ASM.

3. **Q:** Quorum failure group?
   **A:** Voting-only diskgroup member preventing ties in even-node clusters.

4. **Q:** Backup?
   **A:** Not needed — recreated from ASM.

5. **Q:** How to move to a different diskgroup?
   **A:** `crsctl replace votedisk +NEW_DG`.

## References

- Oracle Grid Infrastructure Administration 19c — Voting Files
- MOS Doc ID 428681.1 — Voting Disks
- MOS Doc ID 1050908.1 — CSS Troubleshooting
