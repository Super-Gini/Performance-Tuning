# Runbook: ASM Disk Failure

## Symptom

- Diskgroup member reports OFFLINE / MISSING.
- Alert log: `WARNING: ASM Disk ... offline`.
- Reads/writes to that diskgroup slow or failing.

## Triage

ASM side:

```sql
-- As sysasm on the +ASM instance
CONNECT / AS SYSASM

SELECT   dg.name AS diskgroup, dg.state, dg.type,
         ROUND(dg.total_mb/1024,2) total_gb,
         ROUND(dg.free_mb/1024,2) free_gb,
         ROUND(dg.usable_file_mb/1024,2) usable_gb
FROM     v$asm_diskgroup dg;

SELECT   dg.name diskgroup, d.name disk, d.path,
         d.state, d.mount_status, d.mode_status,
         d.failgroup, ROUND(d.total_mb/1024,2) gb, ROUND(d.free_mb/1024,2) free
FROM     v$asm_diskgroup dg JOIN v$asm_disk d
              ON d.group_number = dg.group_number
WHERE    dg.name = '&target_dg'
ORDER BY d.disk_number;

-- Rebalance
SELECT * FROM v$asm_operation;
```

OS side:

```bash
# Linux devices
ls -la /dev/oracleasm/disks/
ls -la /dev/asm*

# AFD
asmcmd afd_lsdsk

# Kernel messages
dmesg | tail -100 | grep -i sd
```

## Actions

### 1. Determine impact

- `EXTERNAL` redundancy: no ASM mirror — depends on storage array HA. Instance may crash if storage LUN gone.
- `NORMAL` / `HIGH`: ASM mirror will kick in; database keeps running.

### 2. Get the disk back online (if transient)

Often it's a stuck path — try to bring back:

```sql
ALTER DISKGROUP DATA ONLINE DISK 'DATA_0003';
```

Or all in a failure group:

```sql
ALTER DISKGROUP DATA ONLINE ALL;
```

If ASM successfully re-syncs (Fast Mirror Resync), the DG state returns to normal.

Check resync:

```sql
SELECT dg.name, d.name, d.state, d.repair_timer
FROM   v$asm_diskgroup dg JOIN v$asm_disk d
              ON d.group_number = dg.group_number
WHERE  d.state <> 'NORMAL';
```

### 3. If disk is permanently gone

Drop it and let ASM rebalance:

```sql
ALTER DISKGROUP DATA DROP DISK 'DATA_0003';
```

Monitor rebalance:

```sql
SELECT * FROM v$asm_operation;
```

Rebalance can take hours. Increase power if urgent (respect storage capacity):

```sql
ALTER DISKGROUP DATA REBALANCE POWER 8;
```

### 4. Add replacement disks

```sql
ALTER DISKGROUP DATA ADD DISK '/dev/oracleasm/disks/NEWDISK1' NAME DATA_0010;

-- Rebalance
ALTER DISKGROUP DATA REBALANCE POWER 4;
```

### 5. If the whole diskgroup is dismounted

```sql
ALTER DISKGROUP DATA MOUNT;
```

Investigate why it dismounted — could be corruption.

## Verification

```sql
SELECT name, state, type FROM v$asm_diskgroup;
SELECT COUNT(*) FROM v$asm_disk WHERE state <> 'NORMAL';
SELECT * FROM v$asm_operation;

-- DB side — datafiles OK
SELECT status, COUNT(*) FROM v$datafile GROUP BY status;
```

## Post-Mortem

- Root cause: hardware, path, multipath timeout?
- Redundancy adequate?
- ASM `DISK_REPAIR_TIME` set appropriately?
- Alerting caught it in time?

## Prevention

- NORMAL or HIGH redundancy on production DGs.
- Multiple failure groups on different HW.
- Multipath (MPIO / dm-multipath) configured properly.
- Monitor `V$ASM_DISK.STATE` and DG `USABLE_FILE_MB`.
- `DISK_REPAIR_TIME` = 12+ hours so transient issues don't force full drop.

## Related

- [ASM Architecture](../19-asm/asm-architecture.md).
- [ASM Rebalance](../19-asm/rebalance.md).
- [Failure Groups](../19-asm/failure-groups.md).
