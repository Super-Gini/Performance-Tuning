# Runbook: Database Down

## Symptom

- Application timeouts, `ORA-01034: not available`.
- Monitoring shows target UNREACHABLE.
- `sqlplus / as sysdba` says instance not running.

## Triage (Confirm)

```bash
# Host reachable?
ping <db_host>
ssh <db_host>

# Oracle processes?
ps -ef | grep -E "pmon|smon|lgwr" | grep -v grep

# Instance status
sqlplus -S / as sysdba <<EOF
SELECT status FROM v$instance;
EOF
```

If PMON not running → instance is down.

## Immediate Actions

### 1. Read the alert log

```bash
tail -200 $ORACLE_BASE/diag/rdbms/<db>/<inst>/trace/alert_<SID>.log
```

Look for ORA-00600, 07445, 01578, IPC send timeout, "instance terminated", "process ... shutting down instance".

### 2. Check OS resources

```bash
df -h                    # filesystem
free -h                  # memory
vmstat 1 3               # CPU / IO
dmesg | tail -50         # OOM-kill, hardware
```

### 3. Try to start

```bash
# If GI-managed
srvctl start database -db PRD
srvctl status database -db PRD

# Non-GI
sqlplus / as sysdba <<EOF
STARTUP
EXIT
EOF
```

### 4. If startup fails

- **ORA-00205 (control file)** — [runbook](../26-errors/ora-00205.md).
- **ORA-01157 (datafile)** — [runbook](../26-errors/ora-01157.md).
- **ORA-01113 (media recovery needed)** — recover:
  ```sql
  STARTUP MOUNT;
  RECOVER DATABASE;
  ALTER DATABASE OPEN;
  ```
- **ORA-01102 (cannot mount, exclusive)** — stale PMON/lock file:
  ```bash
  rm $ORACLE_HOME/dbs/lk<SID>
  ipcs -sm | grep oracle    # clean stale SHM/SEM if needed
  ipcrm ...
  ```

## Verification

```sql
SELECT instance_name, status, database_status FROM v$instance;
SELECT name, open_mode FROM v$database;

-- CDB
SELECT name, open_mode FROM v$pdbs;
ALTER PLUGGABLE DATABASE ALL OPEN;
```

Application: run a smoke test SQL.

## Communication

Notify app owners, incident bridge. Set expectation for post-mortem.

## Post-Mortem Checklist

- Root cause (crash, human error, OS, hardware).
- Was monitoring adequate?
- Was auto-restart configured?
- RCA doc within 24 h.

## Prevention

- GI-managed instances auto-restart on crash.
- Multiplex control files.
- ADR retention configured (see incident post-mortem).
- Cluster health monitoring.
- RMAN backups + Data Guard for real DR.

## Related

- [Alert Log](../24-monitoring/alert-log.md).
- [Startup Process](../01-fundamentals/startup-process.md).
- [ORA-00205](../26-errors/ora-00205.md), [ORA-01157](../26-errors/ora-01157.md).
