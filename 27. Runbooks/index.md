# Runbooks

Step-by-step response procedures for common production Oracle incidents. Every runbook starts with **triage → diagnosis → action → verification → post-mortem**. Use when paged; adapt to your environment.

## Contents

| Page                                                    | When to Use                               |
| ------------------------------------------------------- | ----------------------------------------- |
| [Database Down](database-down.md)                       | Instance/CRS unreachable                  |
| [Listener Down](listener-down.md)                       | ORA-12541 clientwide                      |
| [Tablespace Full](tablespace-full.md)                   | ORA-01653 / 01654                         |
| [Archive Destination Full](archive-destination-full.md) | ORA-00257                                 |
| [Blocking Sessions](blocking-sessions.md)               | App wait chain / lock contention          |
| [High CPU](high-cpu.md)                                 | DB host CPU saturated                     |
| [High IO](high-io.md)                                   | Storage queue depth or read latency spike |
| [Data Guard Lag](data-guard-lag.md)                     | Standby apply behind                      |
| [Backup Failure](backup-failure.md)                     | RMAN job failed                           |
| [ASM Disk Failure](asm-disk-failure.md)                 | Diskgroup missing / disk offline          |
| [RAC Node Eviction](rac-node-eviction.md)               | Node removed from cluster                 |

## Related

- [Errors (ORA-xxxx)](../26-errors/index.md) — deeper on specific errors.
- [Monitoring](../24-monitoring/index.md) — where signals come from.
- [Scripts](../30-scripts/index.md) — reusable SQL for these procedures.
