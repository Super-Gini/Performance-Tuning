# LREG — Listener Registration

## Purpose

Registers database services with the listener — took over from `PMON` at Oracle 12.1. Publishes:

- `SERVICE_NAME` (from `SERVICE_NAMES`).
- Instance name.
- Load metric (session count, response time).
- Connection handoff information.

## Behavior

- Registers at startup with each listener in `LOCAL_LISTENER` / `REMOTE_LISTENER`.
- Re-registers every 60 s (default).
- Publishes load stats for TAF and Client Failover.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(LREG)%';

-- What LREG has registered
SHOW PARAMETER service_names
SHOW PARAMETER local_listener
SHOW PARAMETER remote_listener

-- Manually register (rare)
ALTER SYSTEM REGISTER;
```

Listener side:

```bash
lsnrctl status LISTENER
lsnrctl services LISTENER
```

You should see your instance under "Services Summary".

## Common Issues

- **Listener shows no services** — LREG never registered. Check `LOCAL_LISTENER` matches listener port; `ALTER SYSTEM REGISTER`.
- **`ORA-12514: TNS:listener does not currently know of service requested`** — Service not registered. Client is asking for `SERVICE_NAME` that LREG hasn't published. See [ORA-12514](../../26-errors/ora-12514.md).
- **Slow client connect** — LREG's load metric is stale. Force re-register.

## Related Views

- `V$SERVICES` — services offered.
- `V$SERVICEMETRIC` — LREG's load feed.

## References

- Oracle Database Net Services Reference 19c — Listener registration
- MOS Doc ID 1596793.1 — LREG (12.1+)
- [Service Registration](../../07-networking/service-registration.md)
