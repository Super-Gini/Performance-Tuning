# Datapatch

## Overview

**Datapatch** applies the **SQL/PLSQL side** of a patch to each database — the data dictionary changes, PL/SQL bodies, and catalog updates that go with the binary changes OPatch put on disk. It's a wrapper around `sqlplus` that reads the patch registry, figures out which SQL fragments need to run, and applies them in the right order to the CDB root and each PDB.

Rule: **every OPatch operation is followed by a Datapatch operation**. Skip Datapatch and you get ORA-04068, ORA-04063, and mysterious ORA-00600s later.

## Where It Lives

```
$ORACLE_HOME/OPatch/datapatch
```

Runs against the current `$ORACLE_SID`. In a CDB, it patches root + all open PDBs in one invocation.

## Prerequisites

1. Binary patch already applied via `opatch apply`.
2. Database open (`STARTUP` in non-CDB; `STARTUP` + `ALTER PLUGGABLE DATABASE ALL OPEN` in CDB).
3. `SYSDBA` credentials (Datapatch runs as `/ as sysdba`).
4. No blocking objects — long-running DDL, invalid `SYSTEM` objects.

## Standard Run

```bash
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=PRD1
export PATH=$ORACLE_HOME/OPatch:$PATH

# All PDBs must be open
sqlplus / as sysdba <<EOF
ALTER PLUGGABLE DATABASE ALL OPEN;
EXIT
EOF

datapatch -verbose
```

Sample output:

```
SQL Patching tool version 19.19.0.0.0 Production on ...
Copyright (c) 2012, 2023, Oracle.  All rights reserved.

Connecting to database...OK
Gathering patch information...OK
Determining current state...
Bootstrapping registry and package to current versions...done

Current state of interim SQL patches:
  ...

Adding patches to installation queue and performing prereq checks...
Installation queue:
  For the following PDBs: CDB$ROOT PDB1 PDB2
    Nothing to roll back
    The following patches will be applied:
      35943157 (Database Release Update : 19.19.0.0.230418 (35943157))

Installing patches...
Patch installation complete.  Total patches installed: 3

Validating logfiles...done
```

## What Datapatch Actually Does

1. Reads `$ORACLE_HOME/sqlpatch/<patch_id>/*` — the SQL scripts shipped with the patch.
2. Compares to `DBA_REGISTRY_SQLPATCH` — what's already applied.
3. For each pending patch:
   - Opens each PDB (if closed).
   - Runs the patch's SQL/PLSQL scripts.
   - Updates `DBA_REGISTRY_SQLPATCH` with status.
4. Writes per-patch, per-container log to `$ORACLE_BASE/cfgtoollogs/sqlpatch/`.

## Verifying After Run

```sql
-- Patches applied per container
SELECT   con_id, patch_id, patch_uid, action, status, description
FROM     cdb_registry_sqlpatch
WHERE    action_time > SYSDATE - 1
ORDER BY con_id, action_time;

-- Any failures
SELECT * FROM cdb_registry_sqlpatch
WHERE  status <> 'SUCCESS';

-- Invalids
SELECT owner, object_type, COUNT(*)
FROM   cdb_objects
WHERE  status = 'INVALID'
GROUP  BY owner, object_type;
```

Recompile invalids:

```sql
EXEC UTL_RECOMP.RECOMP_PARALLEL(8);
```

Or use the shipped script:

```bash
sqlplus / as sysdba @$ORACLE_HOME/rdbms/admin/utlrp.sql
```

## Rolling Back

Only makes sense after `opatch rollback` on the binary side.

```bash
datapatch -verbose
```

Datapatch detects the binary is rolled back and reverses the SQL changes it made previously.

For a **manual force-rollback of a specific patch**:

```bash
datapatch -verbose -force -pdbs 'PDB1,PDB2' -bundle_series RU
```

Rarely needed; usually just re-run `datapatch` after `opatch rollback`.

## CDB Considerations

Datapatch handles CDBs elegantly — but you must **open all PDBs**:

```sql
ALTER PLUGGABLE DATABASE ALL OPEN;
```

If a PDB was closed at Datapatch time, when you later open it you'll see:

```sql
COLUMN violation FORMAT A50
SELECT name, cause, type, message
FROM   pdb_plug_in_violations;
```

Fix:

```sql
ALTER SESSION SET CONTAINER = PDB1;
-- Re-run datapatch scoped to that PDB
```

From the OS:

```bash
datapatch -verbose -pdbs 'PDB1'
```

## Refreshable PDBs, Application PDBs

- **Refreshable PDB**: Datapatch runs on the source; refreshes propagate.
- **Application root / seed / PDB**: Datapatch handles the hierarchy — run once from CDB root.
- **Standby (Data Guard) PDBs**: Redo-apply catches the SQL patches from the primary. Do NOT run Datapatch on the standby.

## Options

| Option                 | Purpose                                         |
| ---------------------- | ----------------------------------------------- |
| `-verbose`             | Print detailed output.                          |
| `-pdbs 'A,B'`          | Restrict to specific PDBs.                      |
| `-skip_upgrade_check`  | Bypass version compat check (dangerous).        |
| `-force`               | Re-run for a patch even if `SUCCESS`.           |
| `-prereq`              | Just print what would be applied.               |
| `-bundle_series RU`    | Restrict to a series.                           |
| `-db_name PRD`         | Use for standby databases where SID may differ. |
| `-apply`               | Explicit apply (default).                       |
| `-rollback <patch_id>` | Explicit rollback of one patch.                 |

## Logs

```
$ORACLE_BASE/cfgtoollogs/sqlpatch/
├── sqlpatch_<pid>_<timestamp>/
│   ├── sqlpatch_summary.log      <- start here
│   ├── sqlpatch_debug.log        <- deep detail
│   └── <patch_id>_apply_<CDB>_CDBROOT_<timestamp>.log
└── ...
```

## Common Issues

- **`Prereq check failed, exiting without installing any patches`** — Usually a stale invalid object in SYS or an in-progress DDL. Query `DBA_DDL_LOCKS`.
- **`ORA-01109: database not open`** — PDB was closed. `ALTER PLUGGABLE DATABASE ALL OPEN`.
- **`ORA-65086: cannot open/close the pluggable database`** — PDB in restricted mode. `ALTER PLUGGABLE DATABASE <name> OPEN`.
- **`No patches to install`** but the DB clearly needs patching — Binary patch not present (`opatch lsinventory` shows it missing) or wrong `$ORACLE_HOME`.
- **`Java heap OutOfMemoryError`** — Very large patch queue. `export DATAPATCH_JVM_ARGS="-Xmx2g -Xms1g"` and rerun.
- **After PDB unplug/plug: PDB_PLUG_IN_VIOLATIONS says patches missing** — Run `datapatch -pdbs 'NEW_PDB'` scoped to that PDB.
- **Datapatch hangs** — Blocked on library cache lock from another session. Kill the blocker.

## Best Practices

1. Always run Datapatch **immediately** after OPatch apply/rollback.
2. Open all PDBs before running.
3. Use `-verbose` — the extra output helps diagnose issues.
4. Verify with `CDB_REGISTRY_SQLPATCH.status = 'SUCCESS'` for every container.
5. Recompile invalids with `UTL_RECOMP.RECOMP_PARALLEL` after.
6. On RAC, run Datapatch on **one node only** — it patches the shared dictionary.
7. On Data Guard, run Datapatch on the **primary only**; redo apply carries changes to standby.
8. Keep the `cfgtoollogs/sqlpatch` directory — some patches emit warnings you'll want to grep later.
9. Set `DATAPATCH_JVM_ARGS` for large patch queues.
10. On a newly plugged-in PDB, always run `datapatch -pdbs '<NEW>'` — this is the #1 forgotten step.

## Interview Questions

1. **Q:** Why do we need Datapatch when OPatch already patched the binaries?
   **A:** Many patches modify PL/SQL bodies, dictionary columns, or seed data — that lives inside the database, not the OS. Only Datapatch can put it there.

2. **Q:** What happens if you skip Datapatch?
   **A:** ORA-04068, ORA-04063, and inconsistent behavior between the binary and the dictionary. Views and packages compile against structures that don't match.

3. **Q:** How is Datapatch different in CDB vs non-CDB?
   **A:** In a CDB it patches root + all open PDBs in one invocation and updates `CDB_REGISTRY_SQLPATCH`. Non-CDB is one target only, `DBA_REGISTRY_SQLPATCH`.

4. **Q:** When you plug in a PDB from an older CDB, why does it show violations?
   **A:** The PDB's dictionary was at a lower patch level than the new CDB. Fix with `datapatch -pdbs 'NEW'`.

5. **Q:** On Data Guard, do you run Datapatch on both primary and standby?
   **A:** No — primary only. The standby applies the same SQL via redo.

## References

- Oracle Database Administrator's Guide 19c — Datapatch
- MOS Doc ID 1585822.1 — Datapatch Master Note
- MOS Doc ID 2680521.1 — Datapatch CDB / PDB
- MOS Doc ID 1929745.1 — OPatch and Datapatch FAQ
