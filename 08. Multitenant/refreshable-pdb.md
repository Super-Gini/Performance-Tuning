# Refreshable PDB

## Overview

A **refreshable PDB** is a hot-cloned PDB that periodically syncs from its source, capturing incremental changes via redo. It's Oracle's answer to "keep a QA/DR copy nearly current" without full re-cloning. Two modes: **MANUAL** refresh (DBA triggers) or **AUTOMATIC** every N minutes.

Refreshable PDBs are **read-only** until manually converted — protects against accidental writes that would break refresh.

## Architecture

```mermaid
flowchart LR
    Source[Source PDB<br/>open R/W] -->|redo| DBLink[DB Link]
    DBLink --> Refresh[Refresh Op]
    Refresh --> Target[Refreshable PDB<br/>MOUNTED or READ ONLY]
    Refresh -.periodic or manual.-> Target
```

## Internal Working

### Prerequisites

- **Local UNDO** on source and target CDBs.
- Source PDB in ARCHIVELOG (CDB in ARCHIVELOG).
- DB link from target CDB to source CDB with `CREATE PLUGGABLE DATABASE` privilege.

### Create Refreshable PDB

```sql
-- On target CDB
CREATE DATABASE LINK srccdb CONNECT TO c##dba IDENTIFIED BY pwd USING 'SRCCDB';

CREATE PLUGGABLE DATABASE hrpdb_qa FROM hrpdb@srccdb
  FILE_NAME_CONVERT = ('/src/hrpdb', '/tgt/hrpdb_qa')
  REFRESH MODE EVERY 60 MINUTES;

ALTER PLUGGABLE DATABASE hrpdb_qa OPEN READ ONLY;
```

`REFRESH MODE EVERY 60 MINUTES` = every hour. `REFRESH MODE MANUAL` = only on demand.

### Refresh

```sql
-- Manual refresh
ALTER PLUGGABLE DATABASE hrpdb_qa REFRESH;
```

Under the hood: source ships incremental redo via the DB link, target applies to catch up.

### Convert to Regular PDB

When you're ready to make the refreshable PDB a normal writable PDB:

```sql
ALTER PLUGGABLE DATABASE hrpdb_qa CLOSE;
ALTER PLUGGABLE DATABASE hrpdb_qa REFRESH MODE NONE;
ALTER PLUGGABLE DATABASE hrpdb_qa OPEN;
```

Once converted, cannot go back to refreshable.

### Use Cases

- **DR-lite** — cheap secondary copy without Data Guard.
- **Reporting** — read-only reporting near-current data.
- **Test environments** — periodically-refreshed QA.

## Components

| Component        | Purpose           |
| ---------------- | ----------------- |
| Source PDB       | Origin            |
| DB link          | Redo transport    |
| Refresh op       | Incremental apply |
| Refresh schedule | MINUTES or MANUAL |

## Important Parameters

None specific. Prerequisite: local UNDO.

## Important Views

| View                                        | Purpose        |
| ------------------------------------------- | -------------- |
| `CDB_PDBS.REFRESH_MODE`, `REFRESH_INTERVAL` | Refresh state  |
| `CDB_PDB_HISTORY`                           | Refresh events |

## Diagnostic Queries

```sql
-- Refresh state
SELECT con_id, name, refresh_mode, refresh_interval,
       last_refresh_scn
FROM   dba_pdbs;

-- More detailed
SELECT pdb_name, refresh_mode, refresh_interval,
       last_refresh_scn
FROM   cdb_pdbs;

-- Recent refresh operations
SELECT db_name, pdb_name, operation, op_timestamp
FROM   cdb_pdb_history
WHERE  operation IN ('REFRESH','SYNC')
ORDER  BY op_timestamp DESC;
```

## Common Operations

### Change refresh schedule

```sql
ALTER PLUGGABLE DATABASE hrpdb_qa REFRESH MODE EVERY 30 MINUTES;
```

### Change to manual

```sql
ALTER PLUGGABLE DATABASE hrpdb_qa REFRESH MODE MANUAL;
```

### Refresh now

```sql
ALTER PLUGGABLE DATABASE hrpdb_qa REFRESH;
```

## Common Issues

- **Refresh fails** — DB link broken, source PDB closed, network issue. Check `cdb_pdb_history` and alert log.
- **`ORA-65141` — cannot open READ WRITE** — Refreshable PDBs are read-only by design.
- **Gap too large** — If source has advanced too far since last refresh (log files not available), full re-clone required.
- **Convert failed** — Ensure `REFRESH MODE NONE` before OPEN R/W.

## Best Practices

1. Use for **hot standby-lite** where full Data Guard is overkill.
2. Refresh interval matched to acceptable staleness (5 min for reporting, 1 hour for QA).
3. Monitor `LAST_REFRESH_SCN` — alert if it's not advancing.
4. Retain sufficient archive logs on source to satisfy target refresh gaps.
5. Combine with data masking scripts before converting to independent PDB.
6. For DR that needs writes, use Data Guard instead.
7. Test convert-to-writable path in a runbook.

## Interview Questions

1. **Q:** What is a refreshable PDB?
   **A:** A PDB that periodically syncs incremental changes from a source PDB via a DB link.

2. **Q:** Prerequisite?
   **A:** Local UNDO on source and target CDBs; source in ARCHIVELOG; DB link.

3. **Q:** Can a refreshable PDB be opened READ WRITE?
   **A:** Not while refreshable. Must first `REFRESH MODE NONE`.

4. **Q:** After converting to writable, can you go back?
   **A:** No — one-way conversion.

5. **Q:** Difference from Data Guard?
   **A:** Refreshable is at PDB level; Data Guard is CDB level. Refreshable uses DB link, not redo transport; less rigorous durability guarantees.

6. **Q:** Refresh modes?
   **A:** `EVERY N MINUTES` or `MANUAL`.

## References

- Oracle Database Multitenant Administrator's Guide 19c — Refreshable PDB
- MOS Doc ID 2015998.1 — Hot Cloning
- MOS Doc ID 2058070.1 — Refresh PDB
