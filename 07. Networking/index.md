# Networking

Oracle Net (formerly SQL\*Net) is the client-server communication protocol suite that connects applications to Oracle databases. Every SELECT, every commit, every backup ultimately traverses this stack: `tnsnames.ora` resolves the name, `sqlnet.ora` sets protocol options, the **listener** accepts on port 1521, and either a **dedicated** or **shared server** process handles the session.

## Contents

| Page                                            | Purpose                                             |
| ----------------------------------------------- | --------------------------------------------------- |
| [Listener](listener.md)                         | `tnslsnr` process — startup, status, load balancing |
| [Listener.ora](listener-ora.md)                 | Listener configuration file                         |
| [Tnsnames.ora](tnsnames-ora.md)                 | Client name resolution                              |
| [Sqlnet.ora](sqlnet-ora.md)                     | Client-and-server protocol settings                 |
| [Service Registration](service-registration.md) | How instances register with listeners (LREG)        |
| [Dedicated Server](dedicated-server.md)         | One server process per session                      |
| [Shared Server](shared-server.md)               | Dispatcher + shared server pool                     |

## Troubleshooting

| Page                                      | ORA   | Meaning                                       |
| ----------------------------------------- | ----- | --------------------------------------------- |
| [ORA-12154](troubleshooting/ora-12154.md) | 12154 | TNS: could not resolve the connect identifier |
| [ORA-12505](troubleshooting/ora-12505.md) | 12505 | TNS: listener does not know of SID            |
| [ORA-12514](troubleshooting/ora-12514.md) | 12514 | TNS: listener does not know of service        |
| [ORA-12516](troubleshooting/ora-12516.md) | 12516 | TNS: listener could not find matching handler |
| [ORA-12519](troubleshooting/ora-12519.md) | 12519 | TNS: no appropriate service handler           |
| [ORA-12520](troubleshooting/ora-12520.md) | 12520 | TNS: could not find available handler         |
| [ORA-12537](troubleshooting/ora-12537.md) | 12537 | TNS: connection closed                        |
| [ORA-12541](troubleshooting/ora-12541.md) | 12541 | TNS: no listener                              |

## Related

- [Errors](../26-errors/index.md) — full ORA-\* error encyclopedia.
- [Runbooks](../27-runbooks/index.md) — including [Listener Down](../27-runbooks/listener-down.md).
- [RAC](../18-rac/index.md) — SCAN listener architecture.
