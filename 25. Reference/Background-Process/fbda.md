# FBDA — Flashback Data Archiver

## Purpose

The background process that populates **Flashback Data Archive (FDA)** — long-term row-history for tables enabled with `FLASHBACK ARCHIVE`. Reads undo asynchronously and writes into archive tables in the designated FDA tablespace.

## Behavior

- Wakes ~5 min or on demand.
- Extracts undo-based history for FDA-enabled tables.
- Writes to internal `SYS_FBA_HIST_...` tables in the FDA's tablespace.
- Deletes rows older than retention.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(FBDA)%';

-- FDA definitions
SELECT flashback_archive_name, retention_in_days, status, create_time
FROM   dba_flashback_archive;

-- Tables using FDA
SELECT owner_name, table_name, flashback_archive_name, archive_table_name
FROM   dba_flashback_archive_tables;

-- Backlog
SELECT owner_name, table_name, count_of_uncompleted_flashback_arc, count_of_completed_flashback_arc
FROM   sys.sys_fba_tracktbs;
```

## Related Views

- `DBA_FLASHBACK_ARCHIVE` — configured archives.
- `DBA_FLASHBACK_ARCHIVE_TABLES` — tables using FDA.
- `DBA_FLASHBACK_ARCHIVE_TS` — tablespaces backing FDA.
- `SYS.SYS_FBA_*` — internal history tables.

## Common Issues

- **FBDA lag — history not up to date** — `FBDA` process behind. Check its trace file; may need more FDA workers via `_fba_maxparallel_workers`.
- **FDA tablespace full** — Retention too long or busy tables. Extend or reduce retention.
- **`ORA-55620` FBDA errors** — Underlying undo issues or corrupt archive tables. See MOS Doc ID 1416878.1.

## References

- Oracle Database Development Guide 19c — Flashback Data Archive
- MOS Doc ID 1416878.1 — FDA troubleshooting
- [Flashback Query](../../16-flashback/flashback-query.md)
