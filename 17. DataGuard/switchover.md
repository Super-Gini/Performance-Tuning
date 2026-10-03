# Switchover

## Overview

**Switchover** is a **planned** role reversal: primary becomes standby, standby becomes primary. Zero data loss. Used for planned maintenance (patching primary hardware, upgrading Oracle Home, testing DR).

The whole database — CDB and all PDBs — switches together. Duration: typically 30 seconds to a few minutes with the broker.

## Prerequisites

- Configuration healthy (`SHOW CONFIGURATION` shows SUCCESS).
- Both primary and standby have Standby Redo Logs.
- No apply lag or transport lag (or acceptably small).
- Users can reconnect via service names (they'll disconnect briefly).

## Broker-Based Switchover (Recommended)

```
$ dgmgrl sys/pwd@prod

DGMGRL> SHOW CONFIGURATION;   -- verify healthy

DGMGRL> VALIDATE DATABASE 'prod_dr';   -- pre-flight check

DGMGRL> SWITCHOVER TO 'prod_dr';
```

Broker walks through:

1. Ensures no transport / apply lag.
2. Converts primary to physical standby.
3. Converts standby to primary.
4. Re-establishes redo transport in the new direction.
5. Restarts services.

## Manual Switchover (Without Broker)

Not recommended, but for completeness:

```sql
-- On primary
ALTER DATABASE COMMIT TO SWITCHOVER TO PHYSICAL STANDBY WITH SESSION SHUTDOWN;
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;

-- On new primary (former standby)
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
SHUTDOWN IMMEDIATE;
STARTUP;
```

## After Switchover

- New primary services register.
- Applications reconnect via service.
- New standby (former primary) starts applying redo from new primary.
- Broker configuration continues to work — no changes needed.

Common actions:

```sql
-- Verify roles
SELECT open_mode, database_role, db_unique_name FROM v$database;

-- On new primary, may need to start services if not automatic
srvctl start service -db prod -service prod_oltp
```

## Service Failover for Applications

For seamless client failover, set up services with:

- `srvctl add service ... -role PRIMARY` — starts on new primary.
- Client connect strings using SCAN + multiple hosts.
- Application Continuity (AC) for OLTP.

## Common Issues

- **`ORA-16416: Switchover target is not ready`** — Standby has lag or missing SRLs. Wait or investigate.
- **`ORA-16775: target standby database in broker operation had unexpected state`** — Broker cannot proceed. `SHOW DATABASE` for details.
- **Application unaware of new primary** — Wrong TNS entries; use SCAN or client-side failover.
- **Sessions disconnect** — Expected. Design for reconnect (retry, AC).

## Best Practices

1. **Regular switchover drills** — annually at minimum.
2. **VALIDATE DATABASE** before every switchover.
3. Notify application team; schedule brief window.
4. Application Continuity for critical apps.
5. Services with `-role PRIMARY` for automatic activation.
6. Monitor lag before initiating — should be zero.
7. Post-switchover: verify new standby applies redo.
8. Keep broker enabled — manual switchover is tedious.
9. Practice in staging first.
10. Document exact commands used and expected outcomes.

## Interview Questions

1. **Q:** Switchover vs failover?
   **A:** Switchover: planned, no data loss, both DBs available. Failover: unplanned; standby takes over, primary may be lost.

2. **Q:** How long does it take?
   **A:** Typically 30 seconds to a few minutes with broker.

3. **Q:** Application impact?
   **A:** Sessions disconnect briefly; reconnect via service. Application Continuity smooths this.

4. **Q:** Pre-flight check?
   **A:** `dgmgrl VALIDATE DATABASE '<standby>'` and `SHOW CONFIGURATION`.

5. **Q:** After switchover, what happens to broker?
   **A:** Continues functioning; roles auto-updated in configuration.

## References

- Oracle Data Guard Broker 19c — Switchover
- MOS Doc ID 1587855.1 — Switchover Best Practices
- MOS Doc ID 1265700.1 — DG Best Practices
