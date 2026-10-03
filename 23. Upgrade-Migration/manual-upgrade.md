# Manual Upgrade

## Overview

The **manual upgrade** path is running the upgrade SQL scripts directly with `catctl.pl` from the higher home. It's the ultimate control mode — every step visible, every parameter overridable — and the fallback for edge cases where DBUA or AutoUpgrade misbehave. It's also what the automation tools call internally.

Situations where manual is worth the effort:

- Failed AutoUpgrade / DBUA, need to complete step-by-step.
- Non-standard database (custom charsets, huge component set, tricky init parameters).
- Data Guard primary + physical standby setups needing precise coordination.
- Educational — understanding what happens under the hood.

## Prerequisites (Same as Any Upgrade)

- 19c home installed.
- Preupgrade tool run:
  ```bash
  $ORACLE_HOME_19/jdk/bin/java -jar \
     $ORACLE_HOME_19/rdbms/admin/preupgrade.jar \
     FILE TEXT DIR /tmp/preup19c
  ```
- Fixups reviewed.
- Full RMAN backup taken.
- Enough FRA for restore point.
- All PDBs open (if CDB).
- `oratab` updated with both homes.

## Step-by-Step

### 1. Copy Init/Password Files to 19c Home

```bash
export SRC_HOME=/u01/app/oracle/product/12.2.0/dbhome_1
export TGT_HOME=/u01/app/oracle/product/19.0.0/dbhome_1

cp -p $SRC_HOME/dbs/spfilePRD.ora  $TGT_HOME/dbs/
cp -p $SRC_HOME/dbs/orapwPRD       $TGT_HOME/dbs/
```

For init.ora (pfile) rather than spfile, do the same and adjust deprecated parameters BEFORE `startup upgrade`.

### 2. Apply Pre-Upgrade Fixups (Source Home)

Log in via source home:

```bash
export ORACLE_HOME=$SRC_HOME
export ORACLE_SID=PRD
export PATH=$ORACLE_HOME/bin:$PATH

sqlplus / as sysdba @/tmp/preup19c/preupgrade_fixups.sql
```

Restart to ensure fixes take:

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
```

### 3. Set Environment to 19c Home

```bash
export ORACLE_HOME=$TGT_HOME
export ORACLE_SID=PRD
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:$LD_LIBRARY_PATH
```

Update `/etc/oratab` to point PRD at the new home.

### 4. Take a Guaranteed Restore Point

```sql
sqlplus / as sysdba

STARTUP MOUNT;
ALTER DATABASE FLASHBACK ON;   -- if not already
CREATE RESTORE POINT PRE_UPGRADE_19C GUARANTEE FLASHBACK DATABASE;
ALTER DATABASE OPEN;
```

### 5. Bring Down Cleanly, Start Up in `UPGRADE` Mode

```sql
SHUTDOWN IMMEDIATE;

STARTUP UPGRADE;
```

If CDB:

```sql
ALTER PLUGGABLE DATABASE ALL OPEN UPGRADE;
```

Verify:

```sql
SELECT name, open_mode FROM v$pdbs;
-- All should be MIGRATE (a.k.a. UPGRADE mode)
```

### 6. Run `catctl.pl` — The Main Upgrade

```bash
cd $ORACLE_HOME/rdbms/admin
$ORACLE_HOME/perl/bin/perl catctl.pl -n 8 -l /u01/app/oracle/upgrade_logs catupgrd.sql
```

Flags:

| Flag       | Purpose                                                  |
| ---------- | -------------------------------------------------------- |
| `-n <p>`   | Parallel degree. Typically `min(cores, 8)`.              |
| `-l <dir>` | Log directory. Contents you'll pore over.                |
| `-c 'A,B'` | Run only these PDBs (in CDB).                            |
| `-C 'A,B'` | Exclude these PDBs.                                      |
| `-p <n>`   | Start at phase N (resume after failure).                 |
| `-P <n>`   | Stop at phase N (bounded run for testing).               |
| `-M`       | Skip `catupgrd_datapatch_upgrade.sql` prerequisite step. |
| `-N`       | Skip the timezone update phase.                          |

The upgrade runs in **~92 phases** (varies by source version). Watch the log dir:

```bash
tail -f /u01/app/oracle/upgrade_logs/catupgrd*.log
```

Typical timing on modern hardware:

- CDB with 3 PDBs, `n=8`: 45–90 minutes.
- Non-CDB, 1 TB: 30–60 minutes.
- Big custom component set (Java, Spatial, OLAP, Text): +30 minutes.

### 7. Post-Upgrade Actions

```bash
# Move back to normal mode
sqlplus / as sysdba
```

```sql
SHUTDOWN IMMEDIATE;
STARTUP;   -- normal mode

-- If CDB
ALTER PLUGGABLE DATABASE ALL OPEN;

-- Recompile
@$ORACLE_HOME/rdbms/admin/utlrp.sql

-- Post-upgrade fixups
@/tmp/preup19c/postupgrade_fixups.sql

-- Time zone
@$ORACLE_HOME/rdbms/admin/utltz_upg_check.sql
@$ORACLE_HOME/rdbms/admin/utltz_upg_apply.sql

-- Dictionary stats
EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
EXEC DBMS_STATS.GATHER_FIXED_OBJECTS_STATS;
```

### 8. Verify

```sql
COLUMN comp_name FORMAT A45
SELECT comp_id, comp_name, version, status
FROM   dba_registry
ORDER  BY comp_name;

SELECT owner, object_type, COUNT(*)
FROM   dba_objects
WHERE  status = 'INVALID'
GROUP  BY owner, object_type;

-- Registry history
SELECT action_time, action, script, version, status
FROM   dba_registry_history
ORDER  BY action_time DESC
FETCH  FIRST 10 ROWS ONLY;
```

Every component `VALID`. Zero invalid SYS/SYSTEM objects.

### 9. Apply Latest RU

Immediately after upgrade, apply the latest RU (see [Release Updates](../22-patching/release-updates.md)):

```bash
opatch apply /patches/<latest_ru>
datapatch -verbose
```

## Resume After Failure

If `catctl.pl` fails mid-phase, don't panic:

```bash
# Find the phase that failed in the log
grep -E "^Serial|^Parallel" /u01/app/oracle/upgrade_logs/catupgrd_summary.log | tail

# Restart from that phase
$ORACLE_HOME/perl/bin/perl catctl.pl -n 8 -p <phase#> \
    -l /u01/app/oracle/upgrade_logs catupgrd.sql
```

## Rollback (Restore Point)

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
FLASHBACK DATABASE TO RESTORE POINT PRE_UPGRADE_19C;
ALTER DATABASE OPEN RESETLOGS;

-- Set oratab back to source home
```

Then update `/etc/oratab` and use the source home going forward.

## RAC Considerations

- Set `cluster_database=FALSE` before starting the upgrade.
- Stop all but one instance.
- Run `catctl.pl` from one node.
- After successful upgrade, set `cluster_database=TRUE`, `startup` on remaining instances.

## Data Guard Considerations

- **Standby stays as-is** during the primary upgrade.
- Redo apply is stopped:
  ```sql
  ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
  ```
- Redo apply from a higher-version primary works into a lower-version standby only for redo-apply-compatible operations — for most upgrades you should **upgrade the standby first** (see MOS Doc ID 1265700.1), then switch over, then upgrade the (new) standby.

## Common Issues

- **Phase X failed with ORA-04068** — Usually a package invalidated mid-run. `-p X` to resume.
- **`ORA-01722` in `catcpb.sql`** — Corruption in a dictionary table. `DBMS_HM.RUN_CHECK` first.
- **`ORA-01555`** in a big catproc phase — Undo tablespace too small. Bump before rerunning.
- **`OJVM` component `INVALID`** after — OJVM RU not applied to 19c home; apply and rerun `catupgrd_ojvm.sql`.
- **`_ORACLE_SCRIPT=TRUE` required warning** — Some manual scripts need this session state; set it.

## Best Practices

1. Do this only when you have a specific reason — AutoUpgrade is safer.
2. Always take a GRP AND an RMAN backup.
3. Log parallel degree = CPU count / 2 (leave headroom).
4. Log everything with `-l` to a dedicated directory you can archive.
5. Never mix `-n` values on resume — reuse the same parallel degree.
6. Compare `dba_registry_history` before / after to confirm each phase ran.
7. Run `utlrp` after every failed / resumed run.
8. Apply the latest RU + Datapatch same day.
9. Baseline the SGA parameters — some default sizings change after upgrade.
10. Document every ORA- in the runbook so it's easier next time.

## Interview Questions

1. **Q:** What does `catctl.pl` do?
   **A:** It's the Perl driver that partitions the upgrade catalog scripts into ~92 phases and runs them serial or parallel per phase against the CDB root and each PDB.

2. **Q:** How do you resume a failed manual upgrade?
   **A:** Find the failing phase in the log, restart `catctl.pl` with `-p <phase#>`.

3. **Q:** Why can `startup upgrade` differ from `startup`?
   **A:** `UPGRADE` mode restricts logon to `SYS`, disables triggers, and skips some initialization so upgrade SQL can rewrite the dictionary safely.

4. **Q:** How do you make a manual upgrade rollback-able?
   **A:** Enable flashback, `CREATE RESTORE POINT ... GUARANTEE FLASHBACK DATABASE` before shutdown-to-upgrade.

5. **Q:** What's your very first action if `catctl.pl` fails at phase 65?
   **A:** Check the log file for the exact ORA-, read `dba_registry_history` for the last successful action, decide: fix and `-p 65` resume, or flashback to restore point.

## References

- Oracle Database Upgrade Guide 19c — Manual Upgrade
- MOS Doc ID 2118136.2 — 19c Master Note
- MOS Doc ID 1265700.1 — Data Guard Rolling Upgrade
- MOS Doc ID 884522.1 — catctl.pl reference
