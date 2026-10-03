# Data Guard Broker

## Overview

**Data Guard Broker** is a management framework wrapping Data Guard configuration into a coherent whole. It offers `dgmgrl` — a command-line client — plus automatic redo transport setup, health monitoring, and the enabler for **Fast-Start Failover**.

Optional but strongly recommended. Without the broker you manage Data Guard via `LOG_ARCHIVE_DEST_n` parameters and MRP0 startup commands — cumbersome and error-prone.

## Enabling

On both primary and every standby:

```sql
ALTER SYSTEM SET dg_broker_start = TRUE SCOPE=BOTH;

-- Confirm
SHOW PARAMETER dg_broker
```

Two broker config files land in `$ORACLE_HOME/dbs/` (or `$ORACLE_BASE/admin`):

```sql
ALTER SYSTEM SET dg_broker_config_file1 = '+DATA/prod/broker/dr1prod.dat' SCOPE=BOTH;
ALTER SYSTEM SET dg_broker_config_file2 = '+RECO/prod/broker/dr2prod.dat' SCOPE=BOTH;
```

## Creating the Configuration

From `dgmgrl` on primary:

```
dgmgrl sys/pwd@prod

DGMGRL> CREATE CONFIGURATION 'DG_PROD' AS
        PRIMARY DATABASE IS 'prod'
        CONNECT IDENTIFIER IS prod;

DGMGRL> ADD DATABASE 'prod_dr' AS
        CONNECT IDENTIFIER IS prod_dr
        MAINTAINED AS PHYSICAL;

DGMGRL> ENABLE CONFIGURATION;

DGMGRL> SHOW CONFIGURATION;
```

## Key `dgmgrl` Commands

```
SHOW CONFIGURATION;                  -- overall status
SHOW DATABASE 'prod';                -- primary detail
SHOW DATABASE 'prod_dr';             -- standby detail
SHOW DATABASE 'prod_dr' 'InconsistentProperties';

EDIT DATABASE 'prod_dr' SET PROPERTY 'LogXptMode' = 'ASYNC';
EDIT CONFIGURATION SET PROTECTION MODE AS MAXAVAILABILITY;

SWITCHOVER TO 'prod_dr';             -- planned switch
FAILOVER TO 'prod_dr';               -- unplanned takeover
REINSTATE DATABASE 'prod';           -- after failover

ENABLE FAST_START FAILOVER;          -- see FSFO
DISABLE FAST_START FAILOVER;

START OBSERVER;                      -- FSFO observer
STOP OBSERVER;

VALIDATE DATABASE 'prod_dr';
```

## Properties

Broker exposes DG behavior via database properties (`EDIT DATABASE ... SET PROPERTY`):

| Property                    | Purpose                   |
| --------------------------- | ------------------------- |
| `LogXptMode`                | SYNC / FASTSYNC / ASYNC   |
| `LogArchiveTrace`           | Debug tracing             |
| `NetTimeout`                | Network timeout           |
| `ApplyLagThreshold`         | Alert threshold (seconds) |
| `TransportLagThreshold`     | Alert threshold (seconds) |
| `RedoCompression`           | ENABLE / DISABLE          |
| `StaticConnectIdentifier`   | For FSFO reinstate        |
| `ObserverConnectIdentifier` | Observer TNS              |

## Standby Reinstatement

After failover, the old primary is _disabled_ in the broker. To bring it back as a standby:

```
DGMGRL> REINSTATE DATABASE 'prod';   -- from new primary
```

Uses flashback logs on the old primary if Flashback Database was enabled. Otherwise, must recreate.

## Diagnostic Queries

```sql
-- Broker configuration state
SELECT database, dgconfig_status, protection_mode
FROM   v$dg_broker_config;

-- Recent broker actions
SELECT * FROM v$dg_broker_history
FETCH FIRST 20 ROWS ONLY;
```

`dgmgrl`:

```
DGMGRL> SHOW CONFIGURATION VERBOSE;
DGMGRL> SHOW DATABASE VERBOSE 'prod_dr';
DGMGRL> VALIDATE DATABASE 'prod_dr';
```

## Common Issues

- **Configuration shows warnings** — `SHOW CONFIGURATION` output lists all databases and any warnings. Investigate each.
- **`ORA-16810: multiple errors or warnings detected`** — Broker's summary; drill down via `SHOW DATABASE`.
- **Property change not applied** — Broker requires database restart for some properties; check `PropertyPending`.
- **`ORA-16600: not connected to target standby database`** — Broker restarted; standby unreachable temporarily.

## Best Practices

1. **Always use the broker in production DG.**
2. Store broker config files on **shared storage** for RAC / on independent disks for single instance.
3. Match Oracle version + PSU across all DG members.
4. Set `ApplyLagThreshold` and `TransportLagThreshold` — broker will alert.
5. Automate `dgmgrl` health checks in monitoring.
6. Enable **FSFO** for critical databases.
7. Standby's `StaticConnectIdentifier` important for FSFO reinstate.
8. Use `VALIDATE DATABASE` before switchover.
9. Test broker commands in a lower environment.
10. Alert on `SHOW CONFIGURATION` output containing "WARNING" or "ERROR".

## Interview Questions

1. **Q:** What is Data Guard Broker?
   **A:** Management framework wrapping DG — provides `dgmgrl` CLI, automatic transport setup, monitoring, FSFO.

2. **Q:** How to enable?
   **A:** `ALTER SYSTEM SET dg_broker_start=TRUE`, then `CREATE CONFIGURATION` and `ADD DATABASE` in `dgmgrl`.

3. **Q:** Switchover command?
   **A:** `SWITCHOVER TO '<standby>';` from `dgmgrl`.

4. **Q:** FSFO?
   **A:** Fast-Start Failover — automatic failover when Observer confirms primary unreachable.

5. **Q:** Broker config files?
   **A:** Two files (`dr1<db>.dat`, `dr2<db>.dat`) storing topology metadata.

6. **Q:** Property changes require restart?
   **A:** Some do; `SHOW DATABASE VERBOSE` lists pending properties.

## References

- Oracle Data Guard Broker 19c
- MOS Doc ID 220970.1 — DG Broker Troubleshooting
- MOS Doc ID 1265700.1 — DG Best Practices
