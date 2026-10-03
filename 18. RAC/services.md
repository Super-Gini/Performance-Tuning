# Services

## Overview

A **service** in RAC is a logical name applications use to connect to the database. Services enable:

- **Node targeting** — direct OLTP to nodes 1–2, reports to node 3.
- **Load balancing** — spread sessions across preferred instances.
- **Failover** — automatic reconnect to another instance when node fails (TAF, Application Continuity).
- **Workload isolation** — Resource Manager plans per service.
- **Rolling maintenance** — relocate service away from a node before patching.

Every RAC database should use services — never let clients connect via SID.

## Adding a Service

```bash
# OLTP: prefer node 1, fall back to nodes 2 and 3
srvctl add service -db prod -service oltp \
  -preferred prod1 \
  -available prod2,prod3 \
  -policy AUTOMATIC \
  -failovertype SELECT \
  -failovermethod BASIC \
  -commit_outcome TRUE \
  -retention 86400 \
  -replay_init_time 1800 \
  -notification TRUE

# Reports: prefer node 3
srvctl add service -db prod -service reports \
  -preferred prod3 -available prod1,prod2
```

## Managing

```bash
srvctl status service -db prod                            # all services
srvctl status service -db prod -service oltp              # specific
srvctl config service -db prod -service oltp

srvctl start service -db prod -service oltp
srvctl stop service -db prod -service oltp

srvctl relocate service -db prod -service oltp -oldinst prod1 -newinst prod2

srvctl remove service -db prod -service oltp
```

## PL/SQL alternative — single instance

```sql
BEGIN
  DBMS_SERVICE.CREATE_SERVICE(
    service_name => 'oltp.corp',
    network_name => 'oltp.corp');
  DBMS_SERVICE.START_SERVICE('oltp.corp');
END;
/
```

RAC users should use `srvctl` — `dbms_service` is single-instance.

## Failover Types

| Failover Type | Description                                            |
| ------------- | ------------------------------------------------------ |
| `NONE`        | No failover; new connect required                      |
| `BASIC`       | Client reconnects to new instance                      |
| `SELECT`      | Reconnect + preserve open cursors (TAF)                |
| `TRANSACTION` | Application Continuity — resume in-flight transactions |

## Application Continuity (AC)

Requires **AC** license. On failover, in-flight transactions replay automatically. Client sees a brief pause; no failed transactions.

Prerequisites:

- Service configured with `-failovertype TRANSACTION`.
- Client uses UCP (Universal Connection Pool), JDBC replay driver, or OCI 12c+.
- Application designed as idempotent transactions.

## FAN (Fast Application Notification)

Services publish events (up/down/reconfig) via **Oracle Notification Service (ONS)**. Clients subscribing to FAN react in milliseconds:

- Immediate connection cleanup on node failure.
- Load balancing based on real-time metrics.

Enable via `-notification TRUE` on service.

## Diagnostic Queries

```sql
-- Active services
SELECT name, network_name, pdb, con_id, failover_type, failover_method
FROM   v$active_services
ORDER  BY name;

-- Sessions per service per instance
SELECT inst_id, service_name, COUNT(*) AS sessions
FROM   gv$session
WHERE  type = 'USER'
GROUP  BY inst_id, service_name
ORDER  BY inst_id, service_name;

-- FAN subscriber activity
-- OS-level: onsctl debug
```

## Common Issues

- **Service not running on any instance** — `srvctl start service`.
- **Uneven load** — Check `srvctl config service` for preferred/available.
- **Client not failing over** — TNS entry uses individual node instead of SCAN; use SCAN + failover clauses.
- **AC not working** — client library too old, or service not `-failovertype TRANSACTION`.

## Best Practices

1. **One service per workload class** — OLTP, reports, batch, admin.
2. **Preferred + available**, not all-preferred — controls load and topology.
3. Use **SCAN** in client connect strings.
4. **TAF SELECT** for reporting clients; **AC TRANSACTION** for OLTP.
5. Notifications (FAN) enabled for fast client reconvergence.
6. Monitor sessions per service per instance — spot skew.
7. Rolling maintenance: `srvctl relocate service` away from node before patching.
8. Standardize service naming (`<app>_<workload>.corp.example.com`).
9. Attach Resource Manager consumer groups by service.

## Interview Questions

1. **Q:** What is a service?
   **A:** Logical connection endpoint that decouples applications from instances/nodes and enables load balancing, failover, and workload isolation.

2. **Q:** Preferred vs available instances?
   **A:** Preferred: normally run on. Available: failover targets.

3. **Q:** TAF types?
   **A:** NONE, BASIC, SELECT (preserves cursors), TRANSACTION (Application Continuity).

4. **Q:** Application Continuity?
   **A:** Replays in-flight transactions after node failure — Oracle 12c+, license required.

5. **Q:** FAN?
   **A:** Fast Application Notification — publishes service state changes; clients react in ms.

## References

- Oracle Real Application Clusters Administration 19c — Services
- MOS Doc ID 2564063.1 — Application Continuity
- MOS Doc ID 2018188.1 — RAC Best Practices
