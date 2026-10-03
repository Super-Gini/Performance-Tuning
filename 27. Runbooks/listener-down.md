# Runbook: Listener Down

## Symptom

- Clients getting ORA-12541 (`TNS: no listener`).
- Monitoring: listener target down.

## Triage

Server side:

```bash
lsnrctl status
lsnrctl status LISTENER_PRD    # named listener

# Process
ps -ef | grep tnslsnr | grep -v grep

# Port
netstat -tlnp | grep 1521
ss -tlnp | grep 1521
```

If not running:

```bash
lsnrctl start
lsnrctl start LISTENER_PRD
```

## If Start Fails

Check listener log:

```bash
tail -100 $ORACLE_BASE/diag/tnslsnr/<host>/listener/trace/listener.log
```

Common causes:

### Port in use

```bash
netstat -tlnp | grep 1521
# something else has it - move Oracle or the other app
```

### Listener config invalid

Validate `listener.ora`:

```bash
$ORACLE_HOME/bin/adrci exec="show alert -tail 50"
```

Try local naming:

```
LISTENER=
  (DESCRIPTION_LIST=
    (DESCRIPTION=
      (ADDRESS=(PROTOCOL=TCP)(HOST=prd-db01)(PORT=1521))))
```

### GI-managed — use srvctl

```bash
srvctl status listener
srvctl start listener
srvctl start listener -listener LISTENER_SCAN1
```

### Wrong ownership

```bash
ls -la $ORACLE_BASE/diag/tnslsnr/
sudo chown -R oracle:oinstall $ORACLE_BASE/diag/tnslsnr/
```

## Restore Instance Registration

After listener restart, instances re-register within 60 s. Force immediately:

```sql
ALTER SYSTEM REGISTER;
```

Verify:

```bash
lsnrctl services
```

## Verification

```bash
tnsping PRD
sqlplus app/pw@PRD <<EOF
SELECT 1 FROM dual;
EOF
```

## Post-Mortem

- Why did it die? Alert log + syslog + `dmesg`.
- Any pattern (weekly restart? memory pressure?)
- Auto-restart configured (systemd unit / `srvctl`)?

## Prevention

- Systemd/CRS auto-restart.
- Monitoring pings listener port every 30s.
- Alert when listener down.

## Related

- [ORA-12541](../26-errors/ora-12541.md).
- [Listener](../07-networking/listener.md).
- [Service Registration](../07-networking/service-registration.md).
