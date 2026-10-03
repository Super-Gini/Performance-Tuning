# OPatch

## Overview

**OPatch** is Oracle's Java-based utility for applying and rolling back binary patches to an Oracle Home. It lives at `$ORACLE_HOME/OPatch/opatch`. Every patch — RU, RUR, one-off — ships as a directory with a `README.txt`, an `etc/config/inventory.xml`, and payload files. OPatch inspects the payload, compares to the OUI inventory, resolves conflicts, and copies files into place.

OPatch does **binary** work only. The catalog SQL side is handled separately by [Datapatch](datapatch.md).

## Getting the Right OPatch Version

Every RU requires a minimum OPatch version. The RU README lists it. Check yours:

```bash
$ORACLE_HOME/OPatch/opatch version
OPatch Version: 12.2.0.1.44
```

Update if needed — download from MOS Patch 6880880 for your platform, unzip **inside** `$ORACLE_HOME`:

```bash
cd $ORACLE_HOME
mv OPatch OPatch_old
unzip -o /tmp/p6880880_190000_Linux-x86-64.zip
```

## Inventory Prep — CRITICAL

Almost every OPatch failure is a **stale inventory**. Verify before you touch anything:

```bash
$ORACLE_HOME/OPatch/opatch lsinventory

# Deep check
$ORACLE_HOME/OPatch/opatch lsinventory -detail | less
```

Look for:

- **All homes registered** in `~/oraInventory/ContentsXML/inventory.xml`.
- **No stale entries** (removed homes still listed).
- Version at bottom matches what you expect.

Fix a broken inventory:

```bash
# If oraInventory is missing / needs re-init
$ORACLE_HOME/oui/bin/attachHome.sh -invPtrLoc /etc/oraInst.loc \
    ORACLE_HOME="$ORACLE_HOME" ORACLE_HOME_NAME="OraDB19Home1"
```

## Standard Patch Application — Single Instance

### 1. Download & Unpack

```bash
# On the DB host, as oracle
cd /u01/app/oracle/patches
unzip p35943157_190000_Linux-x86-64.zip
cd 35943157
cat README.txt   # ALWAYS read; steps vary per RU
```

### 2. Stop the Instance

```bash
sqlplus / as sysdba <<EOF
SHUTDOWN IMMEDIATE
EXIT
EOF

lsnrctl stop
```

Verify nothing is holding `$ORACLE_HOME`:

```bash
lsof +D $ORACLE_HOME 2>/dev/null | head
```

### 3. Conflict Check (Dry-Run)

```bash
cd /u01/app/oracle/patches/35943157
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -ph ./
$ORACLE_HOME/OPatch/opatch prereq CheckSystemSpace -ph ./
```

If a one-off patch will conflict with the RU, download the RU-adjusted replacement one-off from MOS and use `NAPPLY -SKIP` — or take the conflict now.

### 4. Apply

```bash
$ORACLE_HOME/OPatch/opatch apply
# NON-INTERACTIVELY:
$ORACLE_HOME/OPatch/opatch apply -silent
```

Log lands at `$ORACLE_HOME/cfgtoollogs/opatch/opatch<timestamp>.log`.

### 5. Verify Inventory

```bash
$ORACLE_HOME/OPatch/opatch lsinventory | grep -E '^Patch|Applied on'
```

Should now show the new RU on the last line.

### 6. Start + Datapatch

```bash
lsnrctl start
sqlplus / as sysdba <<EOF
STARTUP
EXIT
EOF

# Apply catalog SQL fixes
cd $ORACLE_HOME/OPatch
./datapatch -verbose
```

See [Datapatch](datapatch.md).

## Rollback

Symmetric to apply. Stop instance, then:

```bash
$ORACLE_HOME/OPatch/opatch rollback -id 35943157
# then Datapatch again to reverse SQL
$ORACLE_HOME/OPatch/datapatch -verbose
```

## `napply` — Multiple Patches at Once

```bash
# All patch dirs under ./patches/
$ORACLE_HOME/OPatch/opatch napply ./patches -skip_subset -skip_duplicate
```

Useful when applying an RU + several one-offs. `-skip_duplicate` and `-skip_subset` prevent re-adding fixes already in the RU.

## `auto` — RAC / GI Automation

For Grid Infrastructure and RAC databases, use [OPatchauto](opatchauto.md) instead — it orchestrates rolling patching across nodes.

## Prereq Command Reference

Run these BEFORE the real apply:

```bash
opatch prereq CheckConflictAgainstOHWithDetail -ph <patch_dir>
opatch prereq CheckPatchApplicableOnCurrentPlatform -ph <patch_dir>
opatch prereq CheckSystemSpace -ph <patch_dir>
opatch prereq CheckCompatibilityWithOHConfig -ph <patch_dir>
opatch prereq CheckActiveFilesAndExecutables -ph <patch_dir>
```

## Diagnostic Queries (After Datapatch)

```sql
-- What binary patches has this DB been through
SELECT patch_id, patch_uid, version, action, status, action_time
FROM   dba_registry_sqlpatch
ORDER  BY action_time DESC;

-- Any missing?
SELECT * FROM dba_registry_sqlpatch WHERE status <> 'SUCCESS';

-- OPatch view (12c+)
SELECT patch_id, description FROM opatch_xml_inv WHERE rownum <= 20;
```

Command line:

```bash
opatch lsinventory -bugs_fixed | less
opatch lspatches
```

## Common Issues

- **`OPatchSession cannot load inventory for the given Oracle Home`** — Bad `oraInst.loc` or wrong permissions on `oraInventory`. `chown oracle:oinstall -R oraInventory`.
- **`OUI-67073: UtilSession failed: Prerequisite check "CheckActiveFilesAndExecutables" failed`** — Instance / listener still running, or a process has `$ORACLE_HOME` open. Full shutdown + `fuser $ORACLE_HOME/lib/*`.
- **`OUI-67124: Conflict with patch`** — Existing one-off collides. Get RU-adjusted one-off from MOS or roll back the old one-off.
- **`OPATCHAUTO-72046`** — RAC-specific — see [OPatchauto](opatchauto.md).
- **Patch applies but Datapatch fails** — Usually a PDB is closed. `ALTER PLUGGABLE DATABASE ALL OPEN;` first.
- **After patch: ORA-04043 or ORA-00600 on system views** — Datapatch didn't complete. Rerun `datapatch -verbose`; check `PDB_PLUG_IN_VIOLATIONS`.

## Best Practices

1. **Always** update OPatch to the version required by the RU README first.
2. Test the entire flow in a lower environment before touching production.
3. Take an `opatch lsinventory` snapshot and a **full RMAN backup** immediately before applying.
4. Read the RU README end-to-end each time — steps change.
5. Run all `opatch prereq Check*` commands before `apply`.
6. Never skip Datapatch — the binary is only half the fix.
7. Verify with `dba_registry_sqlpatch` and `dba_registry` afterward.
8. Keep the previous RU on disk until the new one has soaked for a week — makes rollback fast.
9. For RAC, use OPatchauto — don't roll your own.
10. Document every patch applied in your CMDB / runbook.

## Interview Questions

1. **Q:** What is OPatch vs Datapatch?
   **A:** OPatch patches binaries in `$ORACLE_HOME`; Datapatch applies the corresponding catalog SQL to each database.

2. **Q:** Why must the DB be stopped before OPatch apply?
   **A:** OPatch replaces shared libraries and executables in `$ORACLE_HOME`; open file handles from a running instance would prevent this.

3. **Q:** How do you know which patches are on a home?
   **A:** `opatch lsinventory` or `SELECT * FROM opatch_xml_inv`.

4. **Q:** RU vs RUR vs one-off?
   **A:** RU = quarterly bug + security. RUR = regressions/security only on top of last RU. One-off = single-bug fix outside the quarterly cycle.

5. **Q:** How do you roll back a patch that broke production?
   **A:** Stop instance → `opatch rollback -id <id>` → start → `datapatch -verbose`.

## References

- Oracle Universal Installer & OPatch User's Guide 19c
- MOS Patch 6880880 — latest OPatch download
- MOS Doc ID 1929745.1 — OPatch and Datapatch FAQ
- MOS Doc ID 2118136.2 — Master 19c Release Notes
