# ASM Filter Driver (AFD)

## Overview

**ASM Filter Driver (AFD)** is a Linux kernel module (12.1+) that provides two functions:

1. **Persistent device naming** — labels raw devices so ASM sees stable names regardless of `/dev/sd*` reordering after reboot.
2. **I/O filter** — rejects non-Oracle writes to ASM disks at the kernel level, preventing accidental corruption from OS tools (dd, fdisk, mkfs).

AFD replaces the legacy **ASMLIB**. Recommended for all new Linux GI installations.

## Setup

Requires kernel modules and configuration in place before diskgroup creation.

### Prerequisites

- Grid Infrastructure installed.
- Kernel-devel headers matching running kernel.

### Enable AFD (as root)

```bash
# Set environment for grid
export ORACLE_HOME=$GRID_HOME

# Provision devices
$GRID_HOME/bin/asmcmd afd_configure

# Label disks
$GRID_HOME/bin/asmcmd afd_label DATA1 /dev/sdb
$GRID_HOME/bin/asmcmd afd_label DATA2 /dev/sdc
$GRID_HOME/bin/asmcmd afd_label RECO1 /dev/sdd
```

Devices now visible under `/dev/oracleafd/disks/` and referenced as `AFD:DATA1` in ASM.

### Verify

```bash
$GRID_HOME/bin/asmcmd afd_state
$GRID_HOME/bin/asmcmd afd_lslbl
$GRID_HOME/bin/asmcmd afd_scan
```

## Create a Diskgroup with AFD

```sql
SQL> CONNECT / AS SYSASM

SQL> CREATE DISKGROUP data EXTERNAL REDUNDANCY
       DISK 'AFD:DATA1', 'AFD:DATA2'
       ATTRIBUTE 'compatible.asm' = '19.0.0';
```

## Migrating from ASMLIB to AFD

```bash
# Stop CRS
sudo $GRID_HOME/bin/crsctl stop crs

# For each ASMLIB disk, unlabel from ASMLIB, then label with AFD
sudo /etc/init.d/oracleasm stop
# ... for each disk
sudo $GRID_HOME/bin/asmcmd afd_label DATA1 /dev/sdb

# Start CRS
sudo $GRID_HOME/bin/crsctl start crs
```

## AFD vs ASMLIB vs udev

| Method | Persistent naming | I/O filter | Modern |
| ------ | :---------------: | :--------: | :----: |
| AFD    |        ✅         |     ✅     |   ✅   |
| ASMLIB |        ✅         |     ❌     | Legacy |
| udev   |        ✅         |     ❌     |   OK   |

AFD is preferred where supported.

## Diagnostic Queries

```bash
# AFD state
$GRID_HOME/bin/asmcmd afd_state

# List labeled disks
$GRID_HOME/bin/asmcmd afd_lslbl

# Scan for new labeled disks
$GRID_HOME/bin/asmcmd afd_scan
```

```sql
-- ASM disk with AFD path
SELECT name, path, library, mount_status, state
FROM   v$asm_disk
WHERE  path LIKE 'AFD:%';
```

## Common Issues

- **`AFD-620: AFD is not supported on this system`** — Kernel not supported; check MOS matrix.
- **Disk not visible** — `afd_scan` to refresh; check `afd_state`.
- **Kernel update broke AFD** — Reinstall AFD module for the new kernel.
- **AFD not autostart** — verify `oracleafd` systemd service.

## Best Practices

1. **AFD on all new Linux GI** (12.1+).
2. Include AFD kernel module reinstall in OS patching procedure.
3. Label disks with **descriptive names** — DATA1, RECO1, CRS1.
4. Backup label metadata: `afd_lslbl > labels.txt`.
5. Do not mix AFD and ASMLIB in the same diskgroup.
6. Test after kernel updates.
7. Alert if AFD module fails to load.

## Interview Questions

1. **Q:** What is AFD?
   **A:** ASM Filter Driver — kernel module for persistent device naming and I/O filtering to prevent non-Oracle writes.

2. **Q:** AFD vs ASMLIB?
   **A:** AFD adds an I/O filter. ASMLIB (legacy) only provides persistent naming.

3. **Q:** How to label?
   **A:** `asmcmd afd_label <label> <device>`.

4. **Q:** After kernel update?
   **A:** Reinstall AFD module for the new kernel.

5. **Q:** Use in diskgroup DDL?
   **A:** `DISK 'AFD:DATA1'`.

## References

- Oracle ASM Administrator's Guide 19c — AFD
- MOS Doc ID 2098186.1 — AFD Overview
- MOS Doc ID 2130742.1 — AFD Best Practices
