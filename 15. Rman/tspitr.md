# TSPITR — Tablespace Point-in-Time Recovery

## Overview

**Tablespace Point-in-Time Recovery** (TSPITR) restores a specific tablespace to a past SCN/time while the rest of the database remains at the current SCN. Used when a bad batch corrupted one schema but you don't want to roll the entire database back.

TSPITR is complex — RMAN creates a temporary auxiliary instance, restores the tablespace + system-critical files, recovers to target time, exports metadata, imports to primary. Consider [Flashback Table](../16-flashback/flashback-table.md) or Data Pump as simpler alternatives first.

## Prerequisites

- Tablespace is **self-contained** — no dependencies on objects in other tablespaces (chase FKs, indexes, materialized views).
- Backup covering target time exists.
- Archive logs from backup SCN through target SCN.
- Enough disk for auxiliary instance.

Check self-containment:

```sql
BEGIN
  DBMS_TTS.TRANSPORT_SET_CHECK(
    ts_list => 'USERS,USERS_INDEX',
    incl_constraints => TRUE);
END;
/

SELECT * FROM transport_set_violations;
```

If violations exist, you must include all dependent tablespaces or refactor.

## Fully Automated TSPITR

RMAN handles auxiliary instance transparently:

```rman
RMAN> RECOVER TABLESPACE users, users_index
      UNTIL TIME "TO_DATE('2026-08-06 14:30:00','YYYY-MM-DD HH24:MI:SS')"
      AUXILIARY DESTINATION '/aux/tspitr';
```

RMAN:

1. Creates auxiliary instance under `/aux/tspitr`.
2. Restores SYSTEM, SYSAUX, UNDO, target tablespace(s) to auxiliary.
3. Recovers auxiliary to target time.
4. Exports metadata for the target tablespace.
5. Drops the target tablespace from the primary.
6. Plugs the recovered tablespace files back into the primary.
7. Imports the metadata.
8. Drops the auxiliary instance.

## Fully Automated with Named Instance

```rman
RECOVER TABLESPACE users
  UNTIL TIME "SYSDATE - 1"
  AUXILIARY DESTINATION '/aux/tspitr';
```

## After TSPITR

The tablespace lives at the target time. Rest of the database continues from current time. Users may see:

- Objects in the tablespace are at their pre-incident state.
- Objects that referenced now-changed data may error out — verify.
- Statistics for restored objects may be stale — regather.

Take a fresh **L0 backup** including the recovered tablespace right after TSPITR.

## Manual TSPITR (Advanced)

Rarely needed; RMAN's automated flow covers most cases. Refer to the _Backup and Recovery User's Guide_ for the ~30-step manual workflow.

## Common Issues

- **Tablespace not self-contained** — Add dependent tablespaces to the recovery set or use Data Pump instead.
- **`ORA-19849: error while reading backup piece`** — Backup missing; check retention.
- **Insufficient disk in auxiliary destination** — Enlarge or use a different filesystem.
- **`ORA-38856: cannot mark instance UNNAMED (SYSTEM PDBs)` in Multitenant** — TSPITR of PDB tablespaces has extra requirements.
- **PDB TSPITR** — supported in 19c but requires local undo and proper prep.

## Alternatives to Consider First

- **Flashback Table** — if change is within undo retention, one-command undo.
- **Flashback Query** — extract prior version via `AS OF TIMESTAMP`.
- **Data Pump import** — from a logical backup or standby export.
- **Duplicate database** — full clone at target time; extract needed data.

TSPITR is heaviest — reserve for cases where the alternatives can't handle the volume or timeline.

## Diagnostic Queries

```sql
-- Check tablespace self-containment before TSPITR
BEGIN
  DBMS_TTS.TRANSPORT_SET_CHECK('USERS', TRUE);
END;
/
SELECT * FROM transport_set_violations;

-- Prior tablespace state (via flashback dictionary)
SELECT contents, extent_management FROM dba_tablespaces WHERE tablespace_name = 'USERS';

-- Archive log range
SELECT MIN(first_time), MAX(first_time) FROM v$archived_log;
```

## Best Practices

1. **Consider alternatives first** — Flashback Table often simpler.
2. Verify **self-containment** early.
3. Auxiliary destination on **separate storage** — avoids I/O contention.
4. Ensure **archive logs** on-hand for the whole target period.
5. Practice in a lab — TSPITR has many moving parts.
6. Take L0 backup post-TSPITR.
7. Regather statistics on recovered objects.
8. Alert application team — object references may error after tablespace goes back in time.

## Interview Questions

1. **Q:** What is TSPITR?
   **A:** Recovery of a specific tablespace to a past SCN while rest of DB stays current.

2. **Q:** Prerequisites?
   **A:** Self-containment, backup at target time, archive logs, disk for auxiliary.

3. **Q:** Alternatives?
   **A:** Flashback Table, Flashback Query, Data Pump import, duplicate database.

4. **Q:** How does automated TSPITR work?
   **A:** RMAN spins up an auxiliary instance, restores + recovers there, exports metadata, drops old tablespace, plugs recovered files into primary.

5. **Q:** When would you avoid TSPITR?
   **A:** When Flashback (Table/Query/DB) covers the case; when tablespaces are highly interdependent.

6. **Q:** After TSPITR?
   **A:** Verify data consistency, regather stats, take a fresh L0 backup.

## References

- Oracle Database Backup and Recovery User's Guide 19c — TSPITR
- MOS Doc ID 1521286.1 — TSPITR
- MOS Doc ID 1935365.1 — Multitenant PDB TSPITR
