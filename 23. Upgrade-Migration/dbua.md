# DBUA — Database Upgrade Assistant

## Overview

**DBUA** is the interactive/silent GUI wizard for upgrading an Oracle database in place — the legacy predecessor to AutoUpgrade. It ships with every Oracle Home: `$ORACLE_HOME_19/bin/dbua`. It orchestrates the same catupgrd flow AutoUpgrade uses but with an X-Windows GUI walkthrough (or a silent `responseFile` mode).

Since 19c, **Oracle's own recommendation is AutoUpgrade**. DBUA is still fully supported but is being pushed into maintenance mode. Use DBUA when:

- You need a GUI walkthrough for training / audit trail.
- The database is very small and simple; the overhead of writing an AutoUpgrade JSON isn't worth it.
- Your org's runbooks still specify DBUA.

## Prerequisites

Same as AutoUpgrade — see [Upgrade & Migration](index.md):

- 19c home already installed.
- Enough FRA space for the guaranteed restore point.
- Pre-upgrade tool run and issues resolved.
- Archivelog mode (recommended).

Set the environment BEFORE launching:

```bash
# 19c home ORATAB path
export ORACLE_HOME=/u01/app/oracle/product/19.0.0/dbhome_1
export PATH=$ORACLE_HOME/bin:$PATH

# Old SID
export ORACLE_SID=PRD
```

## Interactive Mode

```bash
$ORACLE_HOME_19/bin/dbua
```

Wizard steps:

1. **Select database** — pick from the list `dbua` finds via `/etc/oratab`.
2. **Prerequisite Checks** — runs preupgrade internally; flags issues.
3. **Upgrade Options**:
   - Recompile invalid objects at end.
   - Upgrade timezone data.
   - Gather statistics before/after.
   - Update dictionary stats.
   - Move datafiles / redo logs (optional).
4. **Recovery Options** — Guaranteed Restore Point + optional RMAN backup.
5. **Management Options** — OEM registration.
6. **Summary** — click Finish.

Progress screen shows each phase with elapsed time. On completion you get an HTML report at `$ORACLE_BASE/cfgtoollogs/dbua/`.

## Silent Mode (Response File)

More production-friendly than the GUI:

```bash
$ORACLE_HOME_19/bin/dbua -silent \
    -sid PRD \
    -oracleHome /u01/app/oracle/product/12.2.0/dbhome_1 \
    -upgradeTimezone true \
    -recompileInvalidObjects true \
    -createGRP true \
    -useGRP true \
    -performFixUp true \
    -logDir /u01/app/oracle/upgrade_logs
```

Or feed a response file:

```bash
$ORACLE_HOME_19/bin/dbua -silent -responseFile /tmp/dbua.rsp
```

`dbua.rsp` skeleton:

```
oracle.assistants.dbua.PROPERTIES.SID=PRD
oracle.assistants.dbua.PROPERTIES.oracleHome=/u01/app/oracle/product/12.2.0/dbhome_1
oracle.assistants.dbua.PROPERTIES.upgradeTimezone=true
oracle.assistants.dbua.PROPERTIES.recompileInvalidObjects=true
oracle.assistants.dbua.PROPERTIES.createGRP=true
oracle.assistants.dbua.PROPERTIES.useGRP=true
oracle.assistants.dbua.PROPERTIES.performFixUp=true
oracle.assistants.dbua.PROPERTIES.gatherStatsBeforeUpgrade=true
oracle.assistants.dbua.PROPERTIES.emExpressPort=5500
```

## What DBUA Actually Does

1. Copies (or re-uses) the pfile/spfile, adjusts deprecated parameters.
2. Adds an entry to `/etc/oratab` for the 19c home.
3. Runs preupgrade fixups in the source home.
4. Shuts down the DB in the source home.
5. Starts up in the 19c home in `UPGRADE` mode.
6. Runs `$ORACLE_HOME_19/rdbms/admin/catctl.pl` — the driver for `catupgrd.sql`.
7. Runs `utlrp.sql` for invalids.
8. Optionally runs `utltz_upg_apply.sql`.
9. Removes the source home's `oratab` entry (or updates it to the new home).

The heavy lifting is `catctl.pl -n N catupgrd.sql` — same as manual upgrade.

## Command-Line Reference

```bash
dbua -silent -help
```

Key options:

| Option                       | Purpose                           |
| ---------------------------- | --------------------------------- | ------------------------------ |
| `-sid <name>`                | Source DB.                        |
| `-oracleHome <path>`         | Source home.                      |
| `-newSid <name>`             | Optional: rename during upgrade.  |
| `-parallelUpgrade <n>`       | Parallel degree for catctl.pl.    |
| `-upgradeTimezone <true      | false>`                           | Run tz upgrade.                |
| `-createGRP <true            | false>`                           | Take Guaranteed Restore Point. |
| `-useGRP <true               | false>`                           | Enable rollback capability.    |
| `-performFixUp <true         | false>`                           | Run preupgrade fixups.         |
| `-recompileInvalidObjects`   | Run utlrp.                        |
| `-emConfiguration`           | OEM configuration.                |
| `-postUpgradeScripts <file>` | Custom SQL to run after upgrade.  |
| `-preUpgradeScripts <file>`  | Custom SQL to run before upgrade. |
| `-logDir`                    | Directory for logs.               |

## Post-Upgrade Verification

```sql
COLUMN comp_name FORMAT A45
SELECT comp_id, comp_name, version, status
FROM   dba_registry
ORDER  BY comp_name;

SELECT owner, object_type, COUNT(*)
FROM   dba_objects
WHERE  status = 'INVALID'
GROUP  BY owner, object_type;

-- Check registry log
SELECT action, action_time, id, script,
       time TIME, status
FROM   dba_registry_history
ORDER  BY action_time DESC;
```

Any COMPONENT that isn't `VALID` after upgrade needs investigation.

## Rollback

If you enabled `-createGRP true`:

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO RESTORE POINT <RESTORE_POINT_NAME>;
ALTER DATABASE OPEN RESETLOGS;

-- Switch oratab back to old home
```

DBUA displays the restore point name in its final report — save it.

## Common Issues

- **`ORA-39725: single-instance database cannot be upgraded to Oracle Database 19c`** — Enterprise/Standard edition mismatch. Cannot upgrade SE1 → EE in place.
- **`ORA-01722: invalid number`** in `catupgrd` — Pre-existing corruption in `sys` tables. Run `dbverify` and `DBMS_HM.RUN_CHECK`.
- **DBUA aborts with `DBT-8006`** — Free space check failed. Grow the tablespaces DBUA warned about.
- **Component `JAVAVM` fails** — OJVM patch mismatch. Ensure OJVM RU applied to 19c home.
- **`catctl.pl` OOM** — `LOG_BUFFER` too small, or `_PGA_MAX_SIZE` too small for parallel. Ramp both.
- **Silent mode gives no output** — DBUA silent is quieter than most silent installers. Watch the log dir.

## Best Practices

1. Prefer AutoUpgrade for new automations; use DBUA when the runbook already exists.
2. Always run in silent mode for repeatability.
3. `-createGRP true -useGRP true` — always. Cheap safety.
4. Verify `dba_registry` after — all components `VALID`.
5. Verify `preupgrade.log` file was clean of `SEVERE` items before running.
6. Keep the response file in version control.
7. Take a fresh RMAN backup before DBUA — GRP is not a backup substitute.
8. Test the exact response file on a lower env first.
9. Apply the latest RU immediately after upgrade.
10. Retire the source home only after 30 days of stable operation.

## Interview Questions

1. **Q:** How is DBUA different from AutoUpgrade?
   **A:** DBUA is GUI/silent-response-file; AutoUpgrade is JSON-driven and can handle multiple databases in parallel with more automation around fixups and reports.

2. **Q:** What is a Guaranteed Restore Point?
   **A:** A named restore point that requires flashback logs be retained until dropped — guarantees you can flashback to it regardless of `db_flashback_retention_target`.

3. **Q:** What is `catctl.pl` and how does DBUA use it?
   **A:** The Perl driver for the upgrade SQL. DBUA calls it under the hood: `catctl.pl -n <parallel> catupgrd.sql`.

4. **Q:** What are typical `catupgrd.sql` phases?
   **A:** Data dictionary changes, PL/SQL package rebuilds, component upgrades (XDB, Text, OLAP), timezone updates, statistics gathering.

5. **Q:** Post-upgrade, an OLAP component is `INVALID`. What do you do?
   **A:** Check `dba_registry_history` for the failed step, review `$ORACLE_BASE/cfgtoollogs/dbua/upgrade<time>/catupgrd*.log`, run the component-specific reload script (e.g., `catproc.sql`, `catolap.sql`).

## References

- Oracle Database Upgrade Guide 19c — DBUA chapter
- MOS Doc ID 2298383.1 — DBUA Master Note
- MOS Doc ID 1600006.1 — DBUA troubleshooting
