# Rebalance

## Overview

**Rebalance** is ASM's process of redistributing extents across disks in a diskgroup so all disks share the load evenly. Triggered automatically when disks are added or removed, and manually via `ALTER DISKGROUP ... REBALANCE`.

Rebalance is **online**: databases can read and write during the operation. But it consumes I/O bandwidth — schedule power (parallelism) appropriately.

## Automatic Trigger

- `ALTER DISKGROUP ... ADD DISK` — rebalance to include new disk.
- `ALTER DISKGROUP ... DROP DISK` — rebalance to evacuate departing disk.
- `ALTER DISKGROUP ... RESIZE DISK` — rebalance to reflect new size.

## Manual Trigger

```sql
ALTER DISKGROUP data REBALANCE POWER 8;
ALTER DISKGROUP data REBALANCE POWER 8 WAIT;
ALTER DISKGROUP data REBALANCE POWER 0;  -- pause
```

`WAIT` = block until complete; `NOWAIT` (default) returns immediately.

## Power

`POWER` = number of `ARBn` slave processes performing rebalance. Range 0–1024.

- **1–4** — low impact; slow.
- **5–10** — moderate; standard maintenance windows.
- **11+** — heavy; may impact production I/O.

Set default:

```sql
ALTER SYSTEM SET asm_power_limit = 4 SCOPE=BOTH;
```

Per-operation overrides:

```sql
ALTER DISKGROUP data ADD DISK '/dev/asm/newdisk' REBALANCE POWER 8;
```

## Estimation

Before starting a big rebalance, estimate impact:

```sql
EXPLAIN WORK FOR ALTER DISKGROUP data REBALANCE POWER 8;
SELECT est_minutes FROM v$asm_estimate;
```

## Monitor

```sql
SELECT * FROM v$asm_operation;
-- GROUP_NUMBER, OPERATION, STATE, POWER, SOFAR, EST_WORK, EST_RATE, EST_MINUTES, ERROR_CODE
```

`SOFAR / EST_WORK` = progress.

## Interrupt / Pause

```sql
ALTER DISKGROUP data REBALANCE POWER 0;
```

Halts current rebalance. Can resume by setting non-zero power.

## `DROP DISK` and Rebalance

```sql
ALTER DISKGROUP data DROP DISK data_0002;
```

ASM immediately starts rebalancing to migrate data off the departing disk. Once complete, the disk is physically removed from the diskgroup.

For emergency removal:

```sql
ALTER DISKGROUP data DROP DISK data_0002 FORCE;
```

Data on that disk is **not migrated** — the diskgroup relies on mirror copies for recovery. Only safe under NORMAL/HIGH redundancy.

## Fast Rebalance (12c+)

For adding many disks at once, use `ADD DISK` with multiple disk specs:

```sql
ALTER DISKGROUP data ADD DISK
  '/dev/asm/newdisk01',
  '/dev/asm/newdisk02',
  '/dev/asm/newdisk03',
  '/dev/asm/newdisk04'
  REBALANCE POWER 16;
```

One rebalance operation covers all new disks — faster than one-at-a-time.

## Diagnostic Queries

```sql
-- Current operation
SELECT operation, state, power, sofar, est_work, est_rate,
       ROUND(sofar/DECODE(est_work,0,1,est_work)*100, 1) AS pct,
       est_minutes
FROM   v$asm_operation
WHERE  state <> 'DONE';

-- Rebalance history
SELECT * FROM v$asm_operation_history
ORDER  BY start_time DESC
FETCH FIRST 10 ROWS ONLY;

-- Diskgroup state
SELECT name, state, offline_disks, voting_files
FROM   v$asm_diskgroup;

-- Disks state
SELECT name, state, mode_status, header_status, mount_status
FROM   v$asm_disk
WHERE  state <> 'NORMAL' OR mode_status <> 'ONLINE';
```

## Common Issues

- **Rebalance stuck** — Check I/O bottleneck; may just be slow due to volume.
- **`ORA-15039: could not perform rebalance operation`** — Check `V$ASM_OPERATION.ERROR_CODE`.
- **Rebalance affecting production I/O** — Lower `POWER` or schedule off-hours.
- **Rebalance never completes** — Recent DDL adds more work; monitor `EST_WORK`.

## Best Practices

1. **Schedule rebalances** during off-hours for high power.
2. `ASM_POWER_LIMIT = 4` as default.
3. Use `EXPLAIN WORK` before large rebalances.
4. **Add multiple disks in one command** — fewer rebalance passes.
5. Monitor `V$ASM_OPERATION`.
6. Set alert on rebalance duration > expected.
7. Test power impact in lab before production.
8. Avoid overlapping structural changes.
9. For emergency disk removal under redundant DGs: `DROP DISK FORCE`.

## Interview Questions

1. **Q:** What is ASM rebalance?
   **A:** Redistribute extents across disks to balance load. Triggered by disk add/remove or manual.

2. **Q:** POWER?
   **A:** Number of ARBn slaves — controls parallelism (0–1024). Higher = faster + more I/O impact.

3. **Q:** How to monitor?
   **A:** `V$ASM_OPERATION` — sofar, est_work, est_minutes.

4. **Q:** Online?
   **A:** Yes — databases can read/write during rebalance.

5. **Q:** DROP DISK FORCE?
   **A:** Immediate removal without migration. Relies on mirror copies. NORMAL/HIGH only.

6. **Q:** Add multiple disks at once?
   **A:** `ADD DISK d1, d2, d3` — one rebalance covers all.

## References

- Oracle ASM Administrator's Guide 19c — Rebalance
- MOS Doc ID 1523046.1 — ASM Best Practices
- MOS Doc ID 429293.1 — Rebalance Tuning
