# ARCn — Archiver

## Purpose

Copies filled online redo log files to **archive log destinations** when the database is in `ARCHIVELOG` mode. Multiple ARCn instances run in parallel (default 4, up to 30).

## Triggers

- Log switch — LGWR filled a log and moved on.
- Manual `ALTER SYSTEM ARCHIVE LOG ALL`.
- Redo transport in Data Guard (ARCn ships to standby).

## Multiple Archivers

`LOG_ARCHIVE_MAX_PROCESSES` (default: 4; max 30):

```sql
SHOW PARAMETER log_archive_max_processes
```

More = faster catch-up after a burst; wastes memory when idle.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(ARC%)%';

-- Archiver state
SELECT   log_mode FROM v$database;
SELECT   dest_id, status, error, target
FROM     v$archive_dest_status
WHERE    status <> 'INACTIVE';

-- Archives completed
SELECT   thread#, sequence#, first_change#, applied,
         status, name
FROM     v$archived_log
ORDER BY sequence# DESC
FETCH FIRST 20 ROWS ONLY;

-- Archive log destination stats
SELECT   dest_id, dest_name, destination, log_sequence,
         status, error
FROM     v$archive_dest;
```

## Related Views

- `V$ARCHIVED_LOG` — history.
- `V$ARCHIVE_DEST` — destination config.
- `V$ARCHIVE_DEST_STATUS` — live state.
- `V$ARCHIVE_PROCESSES` — per-ARCn status.

## Common Issues

- **`ORA-00257: archiver error` — DB halts** — Archive destination full. Free space or move archives.
- **ARC0 stuck** — Bad path in `LOG_ARCHIVE_DEST_n`. Test with `ALTER SYSTEM ARCHIVE LOG STOP/START`.
- **`log file switch (archiving needed)`** — LGWR wants a log ARCn hasn't archived yet. Bump `LOG_ARCHIVE_MAX_PROCESSES`.
- **Standby transport lag** — ARCn's remote destination slow; check network, standby storage.

## Best Practices

1. Multiple **local + remote** destinations — `LOG_ARCHIVE_DEST_1`, `LOG_ARCHIVE_DEST_2`, etc.
2. Set `LOG_ARCHIVE_MAX_PROCESSES` = 4 minimum, 8 on Data Guard.
3. Alert on FRA % or archive destination fill.
4. Never `rm` archives — RMAN with retention policy.
5. `DB_RECOVERY_FILE_DEST` = default `LOG_ARCHIVE_DEST_1` — simplest layout.

## References

- Oracle Database Backup and Recovery Guide 19c — Archived Redo
- [Archive Logs](../../04-storage/archive-logs.md)
