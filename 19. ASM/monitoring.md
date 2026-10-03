# ASM Monitoring

## Overview

ASM monitoring means answering: is the ASM instance up? Are diskgroups mounted and healthy? Is any disk offline? Is space consumed evenly? Is a rebalance in progress? Are I/O latencies acceptable?

## Key Views

Query as SYSASM in `+ASM` instance:

| View              | Purpose                      |
| ----------------- | ---------------------------- |
| `V$ASM_DISKGROUP` | Diskgroup state, sizes       |
| `V$ASM_DISK`      | Per-disk state, sizes, path  |
| `V$ASM_FILE`      | Files stored in ASM          |
| `V$ASM_CLIENT`    | Databases connected to ASM   |
| `V$ASM_OPERATION` | In-flight rebalance          |
| `V$ASM_ATTRIBUTE` | Diskgroup attributes         |
| `V$ASM_ALIAS`     | Filename aliases             |
| `V$ASM_TEMPLATE`  | File templates per diskgroup |

## Health Check Query

```sql
-- On +ASM
SELECT dg.name AS diskgroup, dg.state, dg.type,
       dg.total_mb/1024 AS total_gb,
       dg.free_mb/1024 AS free_gb,
       dg.usable_file_mb/1024 AS usable_gb,
       ROUND((1 - dg.free_mb/dg.total_mb) * 100, 1) AS pct_used,
       dg.offline_disks
FROM   v$asm_diskgroup dg
ORDER  BY dg.name;
```

Alert on:

- `state <> MOUNTED`.
- `usable_file_mb < 10% of total` (or set threshold per DG).
- `offline_disks > 0`.

## Disk-Level Check

```sql
SELECT dg.name AS dg, d.name, d.failgroup, d.mount_status,
       d.mode_status, d.state, d.header_status
FROM   v$asm_diskgroup dg JOIN v$asm_disk d ON d.group_number = dg.group_number
WHERE  d.mode_status <> 'ONLINE'
   OR  d.state <> 'NORMAL'
   OR  d.header_status NOT IN ('MEMBER','FORMER');
```

Any row = investigate.

## I/O Metrics

```sql
-- Per-disk I/O
SELECT dg.name AS dg, d.name AS disk,
       d.reads, d.writes,
       ROUND(d.read_time/DECODE(d.reads,0,1,d.reads)*1000, 2) AS avg_read_ms,
       ROUND(d.write_time/DECODE(d.writes,0,1,d.writes)*1000, 2) AS avg_write_ms
FROM   v$asm_diskgroup dg JOIN v$asm_disk d ON d.group_number = dg.group_number
ORDER  BY dg.name, d.name;

-- Aggregate per DG
SELECT dg.name, SUM(d.reads) AS reads, SUM(d.writes) AS writes,
       ROUND(SUM(d.read_time)/DECODE(SUM(d.reads),0,1,SUM(d.reads))*1000, 2) AS avg_read_ms
FROM   v$asm_diskgroup dg JOIN v$asm_disk d ON d.group_number = dg.group_number
GROUP  BY dg.name;
```

Alert on `avg_read_ms > 20` — storage issue.

## Rebalance Monitoring

```sql
SELECT * FROM v$asm_operation;
-- OPERATION, STATE, POWER, SOFAR, EST_WORK, EST_MINUTES
```

Alert if rebalance running > 4 hours unexpectedly.

## Client Connections

```sql
SELECT c.instance_name, c.db_name, c.status,
       dg.name AS diskgroup
FROM   v$asm_client c JOIN v$asm_diskgroup dg ON dg.group_number = c.group_number;
```

## Alert Log

ASM alert log at:

```
$ORACLE_BASE/diag/asm/+asm/+ASMn/alert/log.xml
$ORACLE_BASE/diag/asm/+asm/+ASMn/trace/alert_+ASMn.log
```

Grep for `ERROR` and `WARNING`. Monitor for:

- Disk offline messages.
- Rebalance start/end.
- Diskgroup mount/dismount.
- Voting disk write failures.

## OS-Level

```bash
# ASM background processes
ps -ef | grep -E 'asm_rbal|asm_arb|asm_gmon'

# ASM instance status
crsctl stat res -t | grep asm

# ASM alert log recent
tail -n 200 $ORACLE_BASE/diag/asm/+asm/+ASM1/trace/alert_+ASM1.log
```

## Common Issues

- **Diskgroup dismounted** — Alert log shows why. Remount or investigate.
- **Disk offline** — Check underlying storage; may auto-online after `disk_repair_time`.
- **High avg read/write ms** — Storage array issue or SAN saturation.
- **Rebalance stuck** — Adjust power or wait; check `EST_WORK`.
- **Usable space negative** — Redundancy overhead + failure group asymmetry; add balanced disks.

## Best Practices

1. Query `V$ASM_DISKGROUP` in cron every 5 min.
2. Alert on:
   - Any disk not `NORMAL/ONLINE`.
   - `usable_file_mb / total_mb < 15%`.
   - Rebalance duration exceeding threshold.
   - Alert log ERROR / WARNING.
3. Grafana / OEM dashboard for I/O trends.
4. Cluster Health Advisor for predictive.
5. Manual `md_backup` monthly.
6. Test disk offline / online in lab.
7. Retain ASM alert log ≥ 90 days.

## Interview Questions

1. **Q:** Where do you check diskgroup state?
   **A:** `V$ASM_DISKGROUP`.

2. **Q:** What's `usable_file_mb`?
   **A:** Real free space accounting for redundancy overhead.

3. **Q:** Alert thresholds?
   **A:** Usable < 15%, any offline disk, rebalance overrun, alert log ERROR.

4. **Q:** ASM alert log location?
   **A:** `$ORACLE_BASE/diag/asm/+asm/+ASMn/trace/alert_+ASMn.log`.

5. **Q:** How to check I/O latency?
   **A:** `V$ASM_DISK.read_time / d.reads * 1000` — avg ms.

## References

- Oracle ASM Administrator's Guide 19c
- MOS Doc ID 1523046.1 — ASM Best Practices
- MOS Doc ID 265769.1 — GI Overview
