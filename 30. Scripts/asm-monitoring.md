# ASM Monitoring Scripts

Run as SYSASM on the +ASM instance.

## Diskgroup Overview

```sql
COLUMN name FORMAT A15
COLUMN state FORMAT A10
COLUMN type FORMAT A8

SELECT name, state, type,
       ROUND(total_mb/1024, 2) total_gb,
       ROUND(free_mb/1024, 2)  free_gb,
       ROUND(usable_file_mb/1024, 2) usable_gb,
       ROUND(100*(1-free_mb/total_mb), 1) pct_used
FROM   v$asm_diskgroup
ORDER  BY name;
```

## Disks per Diskgroup

```sql
SELECT dg.name diskgroup, d.name disk, d.path,
       d.state, d.mount_status, d.mode_status,
       d.failgroup,
       ROUND(d.total_mb/1024, 2) total_gb,
       ROUND(d.free_mb/1024, 2)  free_gb,
       d.disk_number
FROM   v$asm_diskgroup dg
JOIN   v$asm_disk d ON d.group_number = dg.group_number
ORDER  BY dg.name, d.disk_number;
```

## Non-Normal Disks (Investigation)

```sql
SELECT dg.name diskgroup, d.name disk, d.path,
       d.state, d.mount_status, d.mode_status, d.failgroup,
       d.repair_timer
FROM   v$asm_diskgroup dg
JOIN   v$asm_disk d ON d.group_number = dg.group_number
WHERE  d.state <> 'NORMAL' OR d.mount_status <> 'CACHED' OR d.mode_status <> 'ONLINE'
ORDER  BY dg.name, d.disk_number;
```

## Rebalance Operations

```sql
SELECT group_number, operation, state, power, actual, sofar, est_work,
       est_rate, est_minutes
FROM   v$asm_operation
ORDER  BY group_number;
```

Convert group_number to name:

```sql
SELECT dg.name, o.operation, o.state, o.power, o.est_minutes
FROM   v$asm_operation o JOIN v$asm_diskgroup dg USING (group_number);
```

## Client Databases

```sql
SELECT dg.name diskgroup, c.instance_name, c.db_name, c.status,
       c.compatible_version
FROM   v$asm_client c JOIN v$asm_diskgroup dg USING (group_number)
ORDER  BY dg.name, c.db_name;
```

## Biggest ASM Files

```sql
SELECT dg.name diskgroup, f.file_number,
       f.type, ROUND(f.bytes/1024/1024/1024, 2) gb,
       f.striped, f.redundancy
FROM   v$asm_file f JOIN v$asm_diskgroup dg USING (group_number)
ORDER  BY f.bytes DESC
FETCH  FIRST 20 ROWS ONLY;
```

## Failure Group Balance

```sql
SELECT dg.name diskgroup, d.failgroup,
       COUNT(*) disks,
       ROUND(SUM(d.total_mb)/1024, 2) total_gb,
       ROUND(SUM(d.free_mb)/1024, 2)  free_gb
FROM   v$asm_diskgroup dg
JOIN   v$asm_disk d ON d.group_number = dg.group_number
GROUP  BY dg.name, d.failgroup
ORDER  BY dg.name, d.failgroup;
```

## ASM Attributes (per Diskgroup)

```sql
SELECT dg.name, a.name, a.value
FROM   v$asm_diskgroup dg
JOIN   v$asm_attribute a ON a.group_number = dg.group_number
WHERE  a.name IN ('au_size','compatible.rdbms','compatible.asm','disk_repair_time',
                  'sector_size','logical_sector_size')
ORDER  BY dg.name, a.name;
```

## OS-Side Health (from compute node)

```bash
# ASM cluster status
crsctl status resource -t | head -30

# Disks visible to OS
ls -l /dev/oracleasm/disks/ 2>/dev/null
asmcmd afd_lsdsk 2>/dev/null

# ASM alert log
tail -100 $ORACLE_BASE/diag/asm/+asm/+ASM1/trace/alert_+ASM1.log
```

## Related

- [ASM Architecture](../19-asm/asm-architecture.md).
- [ASM Rebalance](../19-asm/rebalance.md).
- [ASM Disk Failure runbook](../27-runbooks/asm-disk-failure.md).
