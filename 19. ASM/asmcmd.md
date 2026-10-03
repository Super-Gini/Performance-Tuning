# ASMCMD

## Overview

**`asmcmd`** is a command-line utility for interacting with ASM — file navigation, disk operations, backup metadata, snapshot management. Think of it as `bash` for ASM: `ls`, `cd`, `mkdir`, `cp`, `rm`, plus ASM-specific commands.

Runs as `grid` (owner of GI). Interactive shell or one-off command.

## Basic Usage

```bash
# Interactive
$ asmcmd
ASMCMD> ls
DATA/
RECO/

ASMCMD> cd +DATA/ORCL
ASMCMD> ls
CONTROLFILE/
DATAFILE/
ONLINELOG/
PARAMETERFILE/
TEMPFILE/

# One-off
$ asmcmd ls +DATA/ORCL/DATAFILE/
```

## Common Commands

| Command                   | Purpose                    |
| ------------------------- | -------------------------- |
| `ls`, `cd`, `pwd`         | Filesystem navigation      |
| `mkdir`, `rm`, `mv`, `cp` | File operations            |
| `du`, `df`                | Space usage                |
| `lsdg`                    | Diskgroup list             |
| `lsdsk`                   | Disk list                  |
| `lsct`                    | Client databases connected |
| `lsof`                    | Open files                 |
| `remap`                   | Rewrite corrupt sectors    |
| `md_backup`, `md_restore` | Metadata backup / restore  |
| `volcreate`, `voldelete`  | ACFS volume management     |
| `chgrp`, `chown`, `chmod` | File permissions           |

## Examples

### List diskgroups with usage

```
ASMCMD> lsdg
State  Type    Rebal  Sector  Block  AU  Total_MB   Free_MB  Req_mir_free  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  NORMAL  N       512   4096  1M   1024000    500000        102400          198800              0             N  DATA/
MOUNTED  NORMAL  N       512   4096  1M    512000    350000         51200          149400              0             N  RECO/
```

### List disks

```
ASMCMD> lsdsk -k
Total_MB  Free_MB  OS_MB     Name       Failgroup    Site_Name  Site_GUID       Failgroup_Type  Library                    Label   UDID  Product  Redund   Path
102400   50000    102400    DATA_0000  FG1          ...        ...             REGULAR         AFD Library - Generic ...   DATA1   ...    ...      UNKNOWN  AFD:DATA1
```

### Copy a file out of ASM

```
ASMCMD> cp +DATA/ORCL/DATAFILE/users.257.1234567890 /tmp/users_copy.dbf
```

### Backup diskgroup metadata

```
ASMCMD> md_backup /backup/asm_metadata.bkp
```

Restore later on rebuilt diskgroup:

```
ASMCMD> md_restore /backup/asm_metadata.bkp
```

### Client databases

```
ASMCMD> lsct
DB_Name  Status         Software_Version  Compatible_version  Instance_Name  Disk_Group
orcl     CONNECTED             19.0.0.0.0          19.0.0.0.0           orcl1  DATA
orcl     CONNECTED             19.0.0.0.0          19.0.0.0.0           orcl1  RECO
```

### Space check

```
ASMCMD> df -h
```

## Non-Interactive Scripting

```bash
asmcmd ls +DATA/ORCL/DATAFILE/ | while read f; do echo "File: $f"; done
```

Combine with SQL:

```bash
asmcmd lsdg | grep DATA | awk '{print $8, $9}'
```

## Common Issues

- **`ASMCMD-8102: no connection to ASM`** — GI down or wrong environment. Set `ORACLE_SID=+ASM1` and `ORACLE_HOME=$GRID_HOME`.
- **Permission denied** — Not connected as SYSASM equivalent (grid user with GRID_OWNER role).
- **`cp` fails on large file** — Ensure destination has space.

## Best Practices

1. Learn `lsdg`, `lsdsk`, `lsct`, `ls`, `du` — daily use.
2. `md_backup` during change windows.
3. Use for scripted health checks in monitoring.
4. Run as `grid` user.
5. Prefer `srvctl` for diskgroup lifecycle; `asmcmd` for content operations.
6. Regularly test md_backup / md_restore in lab.

## Interview Questions

1. **Q:** What is `asmcmd`?
   **A:** Command-line utility for ASM — interactive shell for file, disk, and metadata management.

2. **Q:** `lsdg` output shows?
   **A:** Diskgroup name, state, redundancy, sizes, offline disks.

3. **Q:** How to copy a file from ASM to filesystem?
   **A:** `asmcmd cp +DATA/path /local/path`.

4. **Q:** Metadata backup?
   **A:** `md_backup <file>`. Useful before major disk operations.

## References

- Oracle ASM Administrator's Guide 19c — asmcmd
- MOS Doc ID 265769.1 — GI Overview
