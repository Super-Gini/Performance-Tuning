# OCR — Oracle Cluster Registry

## Overview

The **OCR (Oracle Cluster Registry)** is the persistent store for RAC resource metadata: databases, services, listeners, VIPs, network config, ACFS mounts, and application resources. CRSD reads and writes OCR; every `srvctl` command that modifies configuration ultimately updates OCR.

OCR lives on **shared storage** — usually inside an ASM diskgroup (typically `+CRS`, `+DATA`, or `+OCR`). Older GI installations used raw devices or block devices; ASM has been the standard since 11g.

## Files

- **OCR** — primary registry file(s).
- **OLR (Oracle Local Registry)** — per-node local file (`$GRID_HOME/cdata/<host>.olr`) that stores node-local pre-cluster info; boots the node until OCR is accessible.

## Redundancy

- **Multiple OCR mirrors** — up to 5. Configure with `ocrconfig -add` / `ocrconfig -replace`.
- On ASM: NORMAL or HIGH redundancy diskgroup + a copy in a separate diskgroup.

## Standard Operations

```bash
# Check current OCR
ocrcheck

# Verify integrity
ocrcheck -local

# Locations
cat /etc/oracle/ocr.loc     # Linux
cat /var/opt/oracle/ocr.loc # Solaris

# List mirrors
ocrcheck | head

# Backup OCR manually
sudo ocrconfig -manualbackup

# List automatic backups
sudo ocrconfig -showbackup

# Restore
sudo ocrconfig -restore /path/to/backup.ocr
```

## Automatic Backups

CRSD automatically backs up OCR every 4 hours to `$ORACLE_BASE/crsdata/<host>/crsdata`. Retained: last 3 successful backups. Daily backup at midnight; weekly on Sunday.

## Adding / Removing OCR Mirrors

```bash
# Add a mirror
sudo ocrconfig -add +DATA_MIRROR/orcl/ocr.dat

# Replace an existing mirror
sudo ocrconfig -replace +OLD_OCR/orcl/ocr.dat +NEW_OCR/orcl/ocr.dat

# Delete a mirror
sudo ocrconfig -delete +OLD_MIRROR/orcl/ocr.dat
```

Requires GI restart or reconfigure — some operations online, some not.

## Restoring OCR

If OCR is corrupted:

1. Stop CRS on all nodes: `crsctl stop crs`.
2. Identify a good backup: `ocrconfig -showbackup`.
3. Start CRS in exclusive mode on one node: `crsctl start crs -excl`.
4. Restore: `ocrconfig -restore /path/to/backup`.
5. Stop CRS.
6. Start CRS on all nodes: `crsctl start crs` on each.
7. Verify: `ocrcheck`.

Detailed procedure varies by GI version — always consult MOS Doc ID 1062983.1.

## Diagnostic Queries

```bash
# OCR status
ocrcheck

# Locations
ocrcheck | grep -i device

# Automatic backups
sudo ocrconfig -showbackup

# Manual backup
sudo ocrconfig -manualbackup

# Integrity check on local
ocrcheck -local

# OCR content export (readable)
sudo ocrdump /tmp/ocr.dump
head /tmp/ocr.dump
```

## Common Issues

- **`PROT-16: Internal Error`** — OCR corruption. Restore from backup.
- **`PROT-27: Cannot find location`** — OCR file inaccessible (diskgroup dismounted, permissions).
- **Backup missing** — Automatic backup path unreachable. Set `-backuploc`.
- **OCR full** — Enlarge OCR file (raw device / diskgroup).

## Best Practices

1. **Multiple OCR mirrors** — at least 2, ideally 3, on independent storage domains.
2. **Store in HIGH-redundancy ASM diskgroup.**
3. **Automate manual backups** during change windows and before RUs.
4. Retain OCR backups off-cluster.
5. `ocrcheck` in monitoring — alert on integrity failures.
6. Store OCR backup path on shared storage (not local to a single node).
7. Practice OCR restore in a lab periodically.
8. Never manually edit OCR content.
9. Keep `ocr.loc` under strict permissions.
10. Test after any OCR mirror change.

## Interview Questions

1. **Q:** What is OCR?
   **A:** Persistent registry of cluster resources (DBs, services, listeners, VIPs). Read/written by CRSD.

2. **Q:** Where is it stored?
   **A:** Shared storage — typically ASM diskgroup. Location in `/etc/oracle/ocr.loc`.

3. **Q:** OLR?
   **A:** Oracle Local Registry — per-node local file for pre-cluster bootstrap info.

4. **Q:** How often is OCR backed up automatically?
   **A:** Every 4 hours + daily + weekly. Retention: last few backups.

5. **Q:** Manual backup?
   **A:** `sudo ocrconfig -manualbackup`.

6. **Q:** Restore?
   **A:** Stop CRS, start one node exclusive, `ocrconfig -restore`, restart all.

## References

- Oracle Grid Infrastructure Administration 19c — OCR
- MOS Doc ID 1062983.1 — OCR Restore
- MOS Doc ID 428681.1 — OCR Overview
