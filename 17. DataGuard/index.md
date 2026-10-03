# Data Guard

**Oracle Data Guard** is Oracle's built-in disaster recovery framework: a **standby database** (physical or logical) receives redo from a **primary** and applies it to stay in sync. On failure, the standby can become the new primary — a **failover** — with defined RPO and RTO characteristics.

Requires **Enterprise Edition**. Active Data Guard (read + apply concurrently) requires the **Active Data Guard** option.

## Contents

| Page                                        | Purpose                                           |
| ------------------------------------------- | ------------------------------------------------- |
| [Architecture](architecture.md)             | Primary/standby/broker/observer topology          |
| [Redo Transport](redo-transport.md)         | SYNC / ASYNC / FASTSYNC transport                 |
| [Log Apply Services](log-apply-services.md) | MRP0, physical/logical apply                      |
| [Broker](broker.md)                         | Data Guard Broker (`dgmgrl`)                      |
| [Standby Redo Logs](standby-redo-logs.md)   | SRLs — mandatory for SYNC + real-time apply       |
| [Active Data Guard](active-data-guard.md)   | Open read + apply; DML redirection                |
| [Switchover](switchover.md)                 | Planned role reversal                             |
| [Failover](failover.md)                     | Unplanned failover                                |
| [FSFO](fsfo.md)                             | Fast-Start Failover — automated                   |
| [Monitoring](monitoring.md)                 | `dgmgrl` show, `V$DATAGUARD_STATS`, gap detection |

## Related

- [Redo Transport / TT00](../03-instance-architecture/processes/tt00.md).
- [Archive Logs](../04-storage/archive-logs.md).
- [RMAN](../15-rman/index.md) — DG uses RMAN duplicate for standby creation.
