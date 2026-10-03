# Runbook: Data Guard Lag

## Symptom

- Standby apply lag > threshold (typical alert: 15 min).
- `V$DATAGUARD_STATS` `apply lag` growing.
- FRA on primary filling with archives.

## Triage

Primary:

```sql
SELECT database_role, protection_mode, protection_level FROM v$database;

SELECT   dest_id, destination, status, error, log_sequence
FROM     v$archive_dest_status
WHERE    status <> 'INACTIVE';

SELECT MAX(sequence#) primary_current FROM v$archived_log WHERE thread# = 1;
```

Standby:

```sql
SELECT database_role, open_mode FROM v$database;

SELECT name, value, time_computed FROM v$dataguard_stats
ORDER  BY name;

SELECT process, status, thread#, sequence#, block#, blocks
FROM   v$managed_standby
WHERE  process IN ('MRP0','RFS')
ORDER  BY process;

-- Gaps
SELECT * FROM v$archive_gap;
```

## Actions

### 1. Determine bottleneck

Compare primary latest vs standby latest applied:

```sql
-- On primary
SELECT MAX(sequence#) FROM v$archived_log WHERE thread# = 1;

-- On standby
SELECT MAX(sequence#) FROM v$archived_log
WHERE  thread# = 1 AND applied = 'YES';
```

If applied is behind arrived: **apply lag** — MRP0 is slow.
If arrived is behind primary's shipped: **transport lag** — network / redo destination issue.

### 2. Transport lag

- Check `V$ARCHIVE_DEST_STATUS.ERROR` — usually shows why.
- Reset destination:
  ```sql
  ALTER SYSTEM SET log_archive_dest_state_2 = DEFER;
  ALTER SYSTEM SET log_archive_dest_state_2 = ENABLE;
  ```
- Network — check bandwidth between primary and standby.
- ASYNC vs SYNC — SYNC waits for standby ACK.

### 3. Apply lag

Restart MRP0:

```sql
-- On standby
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```

Enable real-time apply (if not):

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

Multi-instance apply:

```sql
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE PARALLEL 4 DISCONNECT;
```

### 4. Fill archive gap

```sql
-- On standby, find gap
SELECT thread#, low_sequence#, high_sequence# FROM v$archive_gap;

-- On primary, extract needed archives
BACKUP AS COPY ARCHIVELOG FROM SEQUENCE 12345 UNTIL SEQUENCE 12400
    FORMAT '/tmp/arch_%s.arc';

-- Ship + register on standby
ALTER DATABASE REGISTER LOGFILE '/tmp/arch_12345.arc';
```

Or use RMAN:

```bash
rman target sys@primary auxiliary sys@standby
RMAN> RECOVER STANDBY DATABASE FROM SERVICE primary;
```

### 5. Broker managed (DGMGRL)

```bash
dgmgrl sys@primary
DGMGRL> SHOW CONFIGURATION VERBOSE;
DGMGRL> SHOW DATABASE VERBOSE 'stby_db';
DGMGRL> EDIT DATABASE 'stby_db' SET STATE = 'APPLY-OFF';
DGMGRL> EDIT DATABASE 'stby_db' SET STATE = 'APPLY-ON';
```

## Verification

```sql
SELECT name, value FROM v$dataguard_stats WHERE name IN ('apply lag','transport lag');
```

Both should trend to zero.

## Post-Mortem

- Root cause: network, MRP0 slow, primary redo spike, standby overloaded (Active Data Guard reports)?
- Adequate BW for redo rate?
- Should we run real-time apply?
- MRP0 parallel level right?

## Prevention

- Alert on apply lag > 15 min.
- Standby storage sized for peak.
- Standby CPU sized for MRP0 + Active DG reports.
- Broker enabled — automates most of this.
- Network monitoring PRI ↔ STBY.

## Related

- [Data Guard Architecture](../17-data-guard/architecture.md).
- [Log Apply Services](../17-data-guard/log-apply-services.md).
- [Data Guard Monitoring](../17-data-guard/monitoring.md).
