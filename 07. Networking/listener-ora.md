# listener.ora

## Overview

`listener.ora` is the configuration file for the Oracle Net Listener. It defines the listener's name, the protocol addresses it listens on (typically TCP port 1521), optional static service registrations, and logging/tracing behavior. Located in `$ORACLE_HOME/network/admin` unless `TNS_ADMIN` is set.

## Structure

```
# listener.ora
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost.example.com)(PORT = 1521))
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (GLOBAL_DBNAME = prod.example.com)
      (SID_NAME = prod)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
    )
  )

INBOUND_CONNECT_TIMEOUT_LISTENER = 60
SUBSCRIBE_FOR_NODE_DOWN_EVENT_LISTENER = OFF
LOG_FILE_LISTENER = listener
LOG_DIRECTORY_LISTENER = /u01/app/oracle/diag/tnslsnr/dbhost/listener/alert
TRACE_DIRECTORY_LISTENER = /u01/app/oracle/diag/tnslsnr/dbhost/listener/trace
TRACE_LEVEL_LISTENER = OFF
```

## Key Sections

### LISTENER name and addresses

```
<LISTENER_NAME> =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = ...)(PORT = ...))
    )
  )
```

- Multiple `ADDRESS` for multiple protocols/ports (TCP + IPC).
- Multiple `DESCRIPTION` blocks for advanced routing (rare).

### SID_LIST — Static Registration

Declare services statically for cases where dynamic registration isn't possible:

- Database in NOMOUNT (RMAN restore).
- Pre-startup connections.
- External procedures.

```
SID_LIST_LISTENER =
  (SID_LIST =
    (SID_DESC =
      (SID_NAME = prod)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (GLOBAL_DBNAME = prod.example.com)
    )
    (SID_DESC =
      (SID_NAME = PLSExtProc)
      (ORACLE_HOME = /u01/app/oracle/product/19.0.0/dbhome_1)
      (PROGRAM = extproc)
    )
  )
```

### Parameters

| Parameter                                   | Default         | Purpose                           |
| ------------------------------------------- | --------------- | --------------------------------- |
| `INBOUND_CONNECT_TIMEOUT_LISTENER`          | 60 s            | Handshake timeout                 |
| `LOG_FILE_LISTENER`                         | `listener`      | Log filename (in ADR)             |
| `LOG_DIRECTORY_LISTENER`                    | ADR path        | Log dir                           |
| `TRACE_LEVEL_LISTENER`                      | OFF             | OFF / USER / ADMIN / SUPPORT      |
| `TRACE_DIRECTORY_LISTENER`                  | ADR trace dir   | Trace file location               |
| `SUBSCRIBE_FOR_NODE_DOWN_EVENT_LISTENER`    | ON in RAC       | ONS notifications                 |
| `ADR_BASE_LISTENER`                         | `$ORACLE_BASE`  | ADR home                          |
| `DEDICATED_THROUGH_BROKER_LISTENER`         | OFF             | Force dedicated                   |
| `USE_DEDICATED_SERVER`                      | OFF             | Always dedicated                  |
| `SECURE_REGISTER_LISTENER`                  | (IPC)           | Only allow local registration     |
| `SECURE_PROTOCOL_LISTENER`                  |                 | Restrict protocols                |
| `VALID_NODE_CHECKING_REGISTRATION_LISTENER` | ON in some 12c+ | Restrict which hosts can register |

### Node Filtering

```
VALID_NODE_CHECKING_REGISTRATION_LISTENER = SUBNET
REGISTRATION_INVITED_NODES_LISTENER = (10.0.1.10, 10.0.1.11)
REGISTRATION_EXCLUDED_NODES_LISTENER = (0.0.0.0/0)
```

Prevents rogue instances on other hosts from registering.

## Multiple Listeners

Define more than one listener on different ports:

```
LISTENER =
  (DESCRIPTION = (ADDRESS = (PROTOCOL=TCP)(HOST=dbhost)(PORT=1521)))

LISTENER_DR =
  (DESCRIPTION = (ADDRESS = (PROTOCOL=TCP)(HOST=dbhost)(PORT=1522)))
```

Start each separately: `lsnrctl start LISTENER_DR`.

## Applying Changes

```bash
# Reload without restart
lsnrctl reload LISTENER

# Full restart (needed for certain parameters)
lsnrctl stop LISTENER
lsnrctl start LISTENER
```

## Common Issues

- **Listener won't start** — syntax error, port in use, permission on log directory.
- **New service still missing after reload** — dynamic registration required; check LREG on instance.
- **`ORA-12546: TNS: permission denied`** — Wrong port privilege (< 1024 needs root/setuid).
- **Registration rejected** — `VALID_NODE_CHECKING_REGISTRATION` blocks source IP.

## Best Practices

1. Keep it simple — don't add addresses/parameters you don't need.
2. Use ADR paths for log and trace directories (already default in 11g+).
3. Set `INBOUND_CONNECT_TIMEOUT_LISTENER = 60` (default) to prevent hung sockets.
4. Enable `VALID_NODE_CHECKING_REGISTRATION_LISTENER = ON` (default in newer versions) — prevents unauthorized registration.
5. Keep static entries only where necessary (RMAN, extproc).
6. Version-control `listener.ora`.
7. Use `SECURE_REGISTER_LISTENER = (IPC)` to only allow local dynamic registration in high-security environments.
8. Document non-default parameter changes in a header comment.

## Interview Questions

1. **Q:** Where is `listener.ora`?
   **A:** `$ORACLE_HOME/network/admin/` or `$TNS_ADMIN` if set.

2. **Q:** Static vs dynamic registration in listener.ora?
   **A:** Static uses `SID_LIST_LISTENER`. Dynamic uses LREG at runtime — no listener.ora entry needed.

3. **Q:** What does `INBOUND_CONNECT_TIMEOUT_LISTENER` do?
   **A:** Bounds how long a client has to complete the initial connect handshake — prevents lingering sockets.

4. **Q:** When do you need `lsnrctl reload` vs `stop/start`?
   **A:** `reload` picks up most changes. Address changes and some security parameters need `stop/start`.

5. **Q:** What is `VALID_NODE_CHECKING_REGISTRATION`?
   **A:** Restricts which hosts/IPs are allowed to dynamically register services.

## References

- Oracle Database Net Services Reference 19c — Listener Parameters
- MOS Doc ID 465570.1 — Listener Configuration
- MOS Doc ID 2019175.1 — Valid Node Checking
