# Backup Strategy

## Overview

An RMAN backup strategy defines **what** you back up, **when**, **how often**, and **for how long** — driven by RPO (Recovery Point Objective) and RTO (Recovery Time Objective). A well-designed strategy makes every recovery scenario answerable: point-in-time to any moment in the retention window, block-level for corruption, full restore for disaster.

## The Two Numbers

- **RPO**: acceptable data loss (e.g., 15 minutes) — dictates archive log backup frequency.
- **RTO**: acceptable downtime for recovery (e.g., 4 hours) — dictates backup type + parallelism.

Everything below is derived from those.

## The Standard Strategy

For most production OLTP:

| Component                                        | Frequency       |
| ------------------------------------------------ | --------------- |
| Full (Level 0)                                   | Weekly (Sunday) |
| Incremental Level 1 (cumulative or differential) | Nightly         |
| Archive log backup                               | Every 2–4 hours |
| Control file autobackup                          | Automatic       |
| Retention                                        | 7–35 days       |
| Off-site copy                                    | Nightly         |

DW / low-DML systems may go longer between L0.

## Incremental Types

- **Level 0** (`INCREMENTAL LEVEL 0`) — equivalent to a full backup, but marked so subsequent L1s can reference it.
- **Level 1 Differential** — since last L0 or L1.
- **Level 1 Cumulative** — since last L0. Slower backup but faster recovery.

RMAN reads changed blocks only (with **Block Change Tracking** on) or scans everything (without BCT).

## Block Change Tracking (BCT)

```sql
ALTER DATABASE ENABLE BLOCK CHANGE TRACKING
  USING FILE '+DATA/PROD/change_tracking.dbf';
```

Maintains a small file (~1/30000 of DB size) listing changed blocks per SCN. Incrementals read only relevant blocks — often 10× faster than without BCT.

Requires enterprise edition. Standard practice on any DB > 100 GB.

## Compression

```
CONFIGURE COMPRESSION ALGORITHM 'MEDIUM';    -- default LOW/MEDIUM/HIGH
```

- `BASIC` — free with EE.
- `LOW` — Advanced Compression Option, LZO-like, fast.
- `MEDIUM` — default; balanced.
- `HIGH` — best ratio; higher CPU.

## Encryption

```
CONFIGURE ENCRYPTION FOR DATABASE ON;
CONFIGURE ENCRYPTION ALGORITHM 'AES256';
```

Three encryption modes:

- **Transparent** — Wallet-based, integrated with TDE.
- **Password** — Backup carries password; restore prompts.
- **Dual** — Wallet OR password.

Regulatory environments: always encrypt backups. Store keys separately from backups.

## Retention Policy

```
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 14 DAYS;
-- OR
CONFIGURE RETENTION POLICY TO REDUNDANCY 3;
```

- **RECOVERY WINDOW** — keep enough to recover to any time within N days. Preferred.
- **REDUNDANCY** — keep N copies of each backup.

## Standard Weekly Script

```rman
# weekly_full.rman
CONNECT TARGET /
CONNECT CATALOG rman_user/pwd@rmancat

RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c2 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c3 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c4 DEVICE TYPE DISK;

  BACKUP AS COMPRESSED BACKUPSET
    INCREMENTAL LEVEL 0
    DATABASE
    PLUS ARCHIVELOG
    DELETE INPUT;

  BACKUP CURRENT CONTROLFILE;
  BACKUP SPFILE;
}

REPORT OBSOLETE;
DELETE NOPROMPT OBSOLETE;
CROSSCHECK BACKUP;
DELETE NOPROMPT EXPIRED BACKUP;
```

## Standard Nightly Incremental

```rman
# nightly_incr.rman
CONNECT TARGET /
CONNECT CATALOG rman_user/pwd@rmancat

RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c2 DEVICE TYPE DISK;

  BACKUP AS COMPRESSED BACKUPSET
    INCREMENTAL LEVEL 1
    DATABASE
    PLUS ARCHIVELOG
    DELETE INPUT;
}

DELETE NOPROMPT OBSOLETE;
```

## Archive Log Backup Every 2 Hours

```rman
# arch_backup.rman
CONNECT TARGET /

RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
  BACKUP ARCHIVELOG ALL NOT BACKED UP 2 TIMES DELETE INPUT;
}
```

## Off-Site Copy

After disk backup completes, copy to off-site storage — tape, cloud object storage (S3, OCI Object Storage), or a secondary site's FRA. Use `BACKUP BACKUPSET` or filesystem copy.

Modern pattern: **Zero Data Loss Recovery Appliance (ZDLRA)** — Oracle-engineered backup target.

## Validate — the Only Way to Trust Backups

```rman
RESTORE DATABASE VALIDATE;
RESTORE ARCHIVELOG ALL VALIDATE;
BACKUP VALIDATE CHECK LOGICAL DATABASE;
```

Reads backup pieces and confirms they're readable + logically consistent. Does not affect the target DB.

## Test Recovery Quarterly

Restore to a scratch machine using nothing but backups and validate startup + application queries.

## Diagnostic Queries

```sql
-- Recent backup successes / failures
SELECT session_key, input_type, status,
       start_time, end_time,
       ROUND(input_bytes/1024/1024/1024, 2) AS input_gb,
       ROUND(output_bytes/1024/1024/1024, 2) AS output_gb,
       ROUND(compression_ratio, 2) AS compression_ratio,
       ROUND(elapsed_seconds/60, 1) AS minutes
FROM   v$rman_backup_job_details
WHERE  start_time > SYSDATE - 14
ORDER  BY start_time DESC;

-- Backup size trend
SELECT TO_CHAR(start_time, 'YYYY-MM-DD') AS day,
       input_type,
       ROUND(SUM(output_bytes)/1024/1024/1024, 1) AS gb
FROM   v$rman_backup_job_details
WHERE  status = 'COMPLETED'
   AND start_time > SYSDATE - 30
GROUP  BY TO_CHAR(start_time, 'YYYY-MM-DD'), input_type
ORDER  BY day DESC;

-- Recovery window
SELECT necessary_backup_time
FROM   v$rman_output
FETCH FIRST 1 ROWS ONLY;

-- BCT status
SELECT status, filename, bytes/1024/1024 AS mb FROM v$block_change_tracking;
```

## Common Issues

- **`RMAN-06183: datafile or datafile copy larger than MAXPIECESIZE`** — Configure larger `MAXPIECESIZE` or use `SECTION SIZE`.
- **`RMAN-03009: failure of backup command`** — Look at underlying error in RMAN output; usually storage / permissions.
- **FRA full** — Delete obsolete backups, resize FRA, or ship off-site more aggressively.
- **Slow backups** — Bump channels; enable BCT; check disk I/O.

## Best Practices

1. **BCT enabled** — free performance for incrementals.
2. `BACKUP AS COMPRESSED BACKUPSET` — save space.
3. Weekly L0 + nightly L1.
4. Archive log backup every 2–4h.
5. Retention ≥ 2× DR window.
6. `DELETE OBSOLETE` after every backup.
7. **CROSSCHECK weekly** to catch missing pieces.
8. Off-site copies mandatory.
9. Quarterly restore drill.
10. Alert on any FAILED status.

## Interview Questions

1. **Q:** L0 vs L1?
   **A:** L0 = full snapshot baseline. L1 = incremental since last L0 (or L1 differential).

2. **Q:** Block Change Tracking?
   **A:** Optional file recording changed blocks per SCN — makes L1 much faster.

3. **Q:** Retention policy?
   **A:** `RECOVERY WINDOW OF N DAYS` = keep enough to recover to any time in the last N days.

4. **Q:** How do you verify backups?
   **A:** `RESTORE DATABASE VALIDATE`, `BACKUP VALIDATE CHECK LOGICAL`.

5. **Q:** Encryption modes?
   **A:** Transparent (wallet), Password, Dual.

6. **Q:** Cumulative vs differential L1?
   **A:** Cumulative: since last L0 (bigger backup, faster restore). Differential: since last L0 or L1 (smaller backup).

## References

- Oracle Database Backup and Recovery User's Guide 19c
- MOS Doc ID 388422.1 — RMAN Best Practices
- MOS Doc ID 1268927.1 — BCT Sizing
