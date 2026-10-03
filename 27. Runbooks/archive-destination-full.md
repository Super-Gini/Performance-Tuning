# Runbook: Archive Destination Full

## Symptom

- ORA-00257 in alert log.
- DB refuses new connections.
- FRA / archive filesystem at 100%.

**Critical**: On ORA-00257, LGWR eventually blocks and the DB freezes. Act quickly.

## Triage

```sql
SELECT name, ROUND(space_used*100/space_limit,2) pct_used,
       ROUND(space_reclaimable*100/space_limit,2) pct_reclaim,
       number_of_files
FROM   v$recovery_file_dest;

SELECT   file_type, percent_space_used, percent_space_reclaimable
FROM     v$flash_recovery_area_usage;

-- Any recent errors on archive dest
SELECT dest_id, status, error FROM v$archive_dest_status WHERE status <> 'INACTIVE';
```

Filesystem:

```bash
df -h /u01/fast_recovery_area
```

## Actions

### 1. Verify backups exist for archives you're about to delete

```bash
rman target /
RMAN> LIST BACKUP OF ARCHIVELOG UNTIL TIME 'SYSDATE-1';
```

### 2. Delete already-backed-up archives

```bash
rman target /
RMAN> DELETE NOPROMPT ARCHIVELOG UNTIL TIME 'SYSDATE-3' BACKED UP 1 TIMES TO DISK;
```

### 3. Backup + delete simultaneously

```bash
RMAN> BACKUP ARCHIVELOG ALL DELETE INPUT;
```

### 4. Clean up expired/obsolete

```bash
RMAN> CROSSCHECK ARCHIVELOG ALL;
RMAN> DELETE EXPIRED ARCHIVELOG ALL;
RMAN> DELETE OBSOLETE;
```

### 5. If FRA holds flashback logs and you can afford to lose that safety net

```sql
-- Only in emergency
ALTER DATABASE FLASHBACK OFF;
```

### 6. Grow FRA

```sql
ALTER SYSTEM SET db_recovery_file_dest_size = 1T SCOPE=BOTH;
```

### 7. Move archives to different destination temporarily

```sql
ALTER SYSTEM SET log_archive_dest_10 = 'LOCATION=/u02/archive' SCOPE=BOTH;
ALTER SYSTEM SWITCH LOGFILE;
```

Data Guard: **do not defer** the standby destination unless you have to — creates a gap.

## Verification

```sql
SELECT name, ROUND(space_used*100/space_limit,2) pct FROM v$recovery_file_dest;

-- Confirm no ORA-00257 recurring
SELECT * FROM v$diag_alert_ext
WHERE  message_text LIKE '%ORA-00257%'
  AND  originating_timestamp > SYSDATE - 5/1440;
```

Application: reconnect and test.

## Post-Mortem

- Why did the FRA fill?
- Backup schedule broken?
- Retention too generous?
- Redo generation spike (batch job)?
- Data Guard gap → archives piling up on primary?

## Prevention

- Alert at FRA 70/80/90.
- Backup archives every 15 min via RMAN + delete input.
- Retention: minimum you need, not maximum you can afford.
- Right-size FRA for peak redo rate (5–10× daily archive).
- Monitor `V$ARCHIVE_DEST_STATUS` — any `ERROR` = paging condition.

## Related

- [ORA-00257](../26-errors/ora-00257.md).
- [Archive Logs](../04-storage/archive-logs.md).
- [Backup Failure runbook](backup-failure.md).
