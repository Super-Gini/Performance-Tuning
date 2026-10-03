# Runbook: Backup Failure

## Symptom

- RMAN job returned non-zero.
- Alert: "Last successful backup > N hours ago."
- V$RMAN_STATUS shows FAILED.

## Triage

```sql
SELECT   session_key, input_type, status,
         start_time, end_time, output_bytes/1024/1024/1024 gb
FROM     v$rman_backup_job_details
WHERE    start_time > SYSDATE - 2
ORDER BY start_time DESC;

-- Errors
SELECT   session_key, status, START_TIME
FROM     v$rman_status
WHERE    status LIKE '%ERROR%' OR status = 'FAILED'
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;
```

RMAN log:

```bash
tail -300 /var/log/rman/prd_$(date +%Y%m%d).log
grep -E 'ORA-|RMAN-' /var/log/rman/prd_$(date +%Y%m%d).log
```

## Common Failure Modes

### `RMAN-06054: media recovery requesting unknown archived log`

Missing archives. Restore from a redundant destination, or accept incomplete recovery.

### `ORA-19566: exceeded limit of MAXCORRUPT for file`

Datafile has more corrupt blocks than allowed:

```
RMAN> BACKUP DATAFILE 5 CHECK LOGICAL;

-- Increase tolerance temporarily
RMAN> SET MAXCORRUPT FOR DATAFILE 5 TO 100;
RMAN> BACKUP DATAFILE 5;
```

Post-fix: block recovery.

### `ORA-27070: async read/write failed`

Storage error — check `dmesg`, storage array logs.

### `RMAN-03002: failure of allocate command`

Backup channel config bad — MML tape library down.

```
RMAN> ALLOCATE CHANNEL c1 DEVICE TYPE 'SBT_TAPE';
```

### `RMAN-03014: implicit resync of recovery catalog failed`

Catalog DB unreachable. Backup without catalog:

```bash
rman target /
```

### `RMAN-06207: WARNING: 12 objects could not be deleted`

Backup pieces missing on disk but still in catalog:

```
RMAN> CROSSCHECK BACKUPSET;
RMAN> DELETE EXPIRED BACKUPSET;
```

### FRA / archive destination full

See [Archive Destination Full](archive-destination-full.md).

## Actions

### 1. Re-run the backup

Assuming the failure was transient:

```bash
rman target / cmdfile=/backup/rman/nightly.rcv log=/var/log/rman/prd_retry.log
```

Or the Scheduler job:

```sql
EXEC DBMS_SCHEDULER.RUN_JOB('OPS.RMAN_NIGHTLY');
```

### 2. Investigate & fix root cause

Read the error carefully. Look up on MOS by RMAN-nnnnn / ORA-nnnnn.

### 3. If a datafile is corrupt

```
RMAN> VALIDATE DATABASE;
RMAN> LIST FAILURE;
RMAN> ADVISE FAILURE;
RMAN> REPAIR FAILURE;
```

Block-level recovery (Enterprise Edition):

```
RMAN> BLOCKRECOVER DATAFILE 5 BLOCK 1234;
```

### 4. Verify restore-ability

Even without a fresh backup, ensure previous ones are usable:

```
RMAN> LIST BACKUP SUMMARY;
RMAN> RESTORE DATABASE VALIDATE;
```

## Verification

```sql
SELECT session_key, status FROM v$rman_status
WHERE session_key > &previous_session_key;
```

Application: nothing to do.

## Post-Mortem

- Root cause: infra, config, corruption?
- Are we monitoring backup **success**, not just running?
- Retention window still met?
- Restore rehearsal scheduled?

## Prevention

- Backup monitoring that alerts on **absence of a recent success**, not just failure of the last job.
- Backups + validate weekly.
- RMAN retention policy documented.
- Restore rehearsals quarterly.
- Catalog DB itself backed up.

## Related

- [RMAN Architecture](../15-rman/rman-architecture.md).
- [Backup Strategy](../15-rman/backup-strategy.md).
- [Block Media Recovery](../15-rman/block-media-recovery.md).
