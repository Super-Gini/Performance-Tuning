# Failover

## Overview

**Failover** is an **unplanned** takeover: primary is down or unreachable, and a standby is promoted to primary. Under SYNC/AFFIRM: zero data loss. Under ASYNC: possible small loss.

Different from switchover — failover assumes primary can't participate. The old primary becomes an outsider and must be reinstated (or recreated) before rejoining the configuration.

## Types

- **Manual failover** — DBA runs `FAILOVER TO ...`.
- **Fast-Start Failover (FSFO)** — Observer initiates automatically. See [FSFO](fsfo.md).

## Broker-Based Failover

```
$ dgmgrl sys/pwd@prod_dr    # note: connect to standby (primary is down)

DGMGRL> SHOW CONFIGURATION;   -- shows primary WARNING/ERROR

DGMGRL> FAILOVER TO 'prod_dr';
```

Broker:

1. Confirms primary is unreachable.
2. Applies any redo already received on standby.
3. Promotes standby to primary.
4. Marks old primary DISABLED in configuration.

## Manual Failover (Without Broker)

```sql
-- On standby
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE FINISH FORCE;
ALTER DATABASE COMMIT TO SWITCHOVER TO PRIMARY WITH SESSION SHUTDOWN;
SHUTDOWN IMMEDIATE;
STARTUP;
```

## After Failover

- New primary is R/W.
- Services register.
- Old primary is in a disabled state — cannot rejoin without reinstatement.

## Reinstating the Old Primary

If **Flashback Database was enabled** on the old primary, broker can flashback and rejoin:

```
DGMGRL> REINSTATE DATABASE 'prod';
```

The former primary flashes back to the SCN at which the standby took over, then becomes the new standby applying redo from the new primary.

If Flashback wasn't enabled, you must **recreate the standby** from the new primary via RMAN duplicate.

## Data Loss Assessment

Even under SYNC, tiny loss windows can exist:

- Uncommitted transactions on primary at failover time are lost.
- If primary failed mid-write of the current redo log, the redo up to the last durable SCN on standby is preserved.

For zero-tolerance environments: **Maximum Protection** mode — primary halts if standby unreachable, ensuring zero data loss on failover.

## RTO — Recovery Time Objective

Failover time depends on:

- **Broker vs manual** — broker is faster (30–60 seconds).
- **Apply lag** — standby applies any queued redo before opening R/W.
- **Application reconnect logic** — how long until clients reconnect.

Target: RTO < 5 minutes for critical systems.

## Common Issues

- **`ORA-16700: standby database has diverged`** — Standby is too far ahead of what primary had before failing (rare edge case).
- **Reinstate fails** — Flashback not enabled or logs insufficient. Recreate standby.
- **Application not reconnecting** — Missing service failover config or wrong TNS.
- **Split-brain** — Old primary comes back before reinstate. Fence off (disable listeners) until reinstated.

## Best Practices

1. **Enable Flashback Database on primary and standby** — enables reinstate.
2. **Test failover in a lower environment** annually.
3. Use **FSFO** for critical databases where automated response is required.
4. Maintain **runbook** with exact commands.
5. **Fence** old primary immediately after failover (shut down listeners) until reinstate is complete.
6. Coordinate with application team — session loss is unavoidable.
7. Take fresh L0 backup immediately after failover.
8. Monitor `SHOW CONFIGURATION` for reinstate progress.
9. Retain observer / decision logs for postmortem.
10. Practice — muscle memory matters at 3 AM.

## Interview Questions

1. **Q:** Failover vs switchover?
   **A:** Failover: unplanned, primary lost or unreachable. Switchover: planned, both DBs available.

2. **Q:** Data loss in failover?
   **A:** Zero under SYNC/AFFIRM. Small window under ASYNC.

3. **Q:** How to bring old primary back?
   **A:** `REINSTATE DATABASE` if Flashback enabled; otherwise recreate via RMAN duplicate.

4. **Q:** Manual failover steps?
   **A:** Cancel apply, `RECOVER ... FINISH FORCE`, `COMMIT TO SWITCHOVER TO PRIMARY`, restart.

5. **Q:** FSFO?
   **A:** Fast-Start Failover — Observer automatically initiates failover when primary is unreachable.

## References

- Oracle Data Guard Broker 19c — Failover
- MOS Doc ID 1587855.1 — Failover Best Practices
- MOS Doc ID 1550116.1 — DG Failover Scenarios
