# tnsnames.ora

## Overview

`tnsnames.ora` is the **client-side** file that maps a short connect string (called a **TNS alias** or **net service name**) to a network address + service name. When an application connects as `HR@PROD`, the alias `PROD` is looked up in `tnsnames.ora` and resolved to a connect descriptor like `(DESCRIPTION=(ADDRESS=...)(CONNECT_DATA=...))`.

Located by default in `$ORACLE_HOME/network/admin/tnsnames.ora`; overridden by `TNS_ADMIN` env var. In LDAP-based name resolution or Easy Connect (EZCONNECT), `tnsnames.ora` is not needed.

## Structure

```
PROD =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (LOAD_BALANCE = ON)
      (FAILOVER = ON)
      (ADDRESS = (PROTOCOL = TCP)(HOST = scan.corp.example.com)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = prod.corp.example.com)
      (FAILOVER_MODE = (TYPE = SELECT)(METHOD = BASIC)(RETRIES = 20)(DELAY = 5))
    )
  )

PROD_DR =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = dr-scan.corp.example.com)(PORT = 1521))
    (CONNECT_DATA =
      (SERVICE_NAME = prod_dr.corp.example.com)
    )
  )
```

## Key Elements

### DESCRIPTION

The top-level container for one alias resolution.

### ADDRESS / ADDRESS_LIST

Physical protocol addresses. Multiple addresses for HA (SCAN, DG, multiple listeners).

```
(ADDRESS_LIST =
  (LOAD_BALANCE = ON)   # randomly pick one on connect
  (FAILOVER = ON)       # try next on failure
  (ADDRESS = (PROTOCOL=TCP)(HOST=host1)(PORT=1521))
  (ADDRESS = (PROTOCOL=TCP)(HOST=host2)(PORT=1521))
)
```

### CONNECT_DATA

- `SERVICE_NAME` — logical service (preferred).
- `SID` — legacy instance identifier.
- `SERVER = DEDICATED | SHARED | POOLED` — server mode.
- `INSTANCE_NAME` — request specific instance in RAC.

### FAILOVER_MODE (TAF — Transparent Application Failover)

```
(FAILOVER_MODE =
  (TYPE = SELECT)        # SESSION or SELECT
  (METHOD = BASIC)       # BASIC or PRECONNECT
  (RETRIES = 20)
  (DELAY = 5))
```

- **TYPE=SESSION** — new session on failover; in-flight query is aborted.
- **TYPE=SELECT** — preserves the select cursor state on failover.
- **METHOD=BASIC** — reconnect on failure.
- **METHOD=PRECONNECT** — pre-open backup connection (deprecated).

## Easy Connect (EZCONNECT)

Since 10g, you can skip `tnsnames.ora` entirely:

```
sqlplus hr/pwd@//dbhost:1521/prod.corp.example.com
```

Enhanced in 19c+:

```
# 19c EZCONNECT Plus
sqlplus hr/pwd@'tcp://dbhost:1521/prod.corp.example.com?
  connect_timeout=30&retry_count=3&retry_delay=5'
```

## Multiple Alias Files

`TNS_ADMIN` env var overrides the default location. Alternately, LDAP for centralized name resolution:

```
# sqlnet.ora
NAMES.DIRECTORY_PATH = (LDAP, EZCONNECT, TNSNAMES)
```

## Applying Changes

Clients pick up `tnsnames.ora` changes **automatically** at next connect. No reload required.

## Diagnostic

```bash
# Client-side test
tnsping PROD
tnsping PROD 3        # 3 attempts

# Manual resolution
sqlplus hr/pwd@PROD
```

## Common Issues

- **`ORA-12154: TNS: could not resolve the connect identifier`** — Alias not found in `tnsnames.ora` or `sqlnet.ora` `NAMES.DIRECTORY_PATH` misconfigured.
- **`ORA-12514: TNS: listener does not currently know of service`** — Reached listener but wrong service name.
- **`ORA-12541: TNS: no listener`** — Reached host but no listener on port.
- **Syntax errors** — Missing parenthesis, misspelled keyword.
- **`TNS_ADMIN` inconsistent** — Different apps looking at different `tnsnames.ora`.

## Best Practices

1. **Version-control** `tnsnames.ora` for each application tier.
2. **Standardize** with a single canonical `tnsnames.ora` distributed to all clients via config management.
3. Use **SERVICE_NAME**, not SID.
4. For RAC, target **SCAN name** (`ADDRESS=SCAN:1521`) — single entry, Oracle resolves.
5. Include Data Guard alternates for DR readiness.
6. **EZCONNECT** or **LDAP** for large environments avoid manual file distribution.
7. Test with `tnsping` after any change.
8. Set `CONNECT_TIMEOUT` in DESCRIPTION to bound TCP connect delay.
9. Never store passwords in `tnsnames.ora`.

## Advanced Features

### Load Balancing

```
(DESCRIPTION =
  (LOAD_BALANCE = ON)             # client-side CLB
  (FAILOVER = ON)
  (ADDRESS_LIST = ...))
```

### Connect timeout

```
(DESCRIPTION =
  (CONNECT_TIMEOUT = 30)
  (RETRY_COUNT = 3)
  (RETRY_DELAY = 5)
  (ADDRESS = ...))
```

### Application Continuity (AC) hooks

```
(CONNECT_DATA =
  (SERVICE_NAME = prod.corp)
  (SERVER = DEDICATED)
  (POOL_BOUNDARY = TRANSACTION))
```

## Interview Questions

1. **Q:** What is `tnsnames.ora`?
   **A:** Client-side file mapping short aliases to full Oracle Net connect descriptors.

2. **Q:** Where does `TNS_ADMIN` fit in?
   **A:** Overrides the default location `$ORACLE_HOME/network/admin`.

3. **Q:** SERVICE_NAME vs SID?
   **A:** SERVICE_NAME is the modern, logical service name — works with RAC, DG failover. SID is the legacy instance identifier.

4. **Q:** What is EZCONNECT?
   **A:** `//host:port/service` syntax — skip `tnsnames.ora`.

5. **Q:** How to test resolution?
   **A:** `tnsping <alias>`.

6. **Q:** TAF: SELECT vs SESSION?
   **A:** SELECT preserves cursor state after failover. SESSION just reconnects; in-flight query is aborted.

7. **Q:** How do RAC clients avoid manually listing all nodes?
   **A:** Use SCAN — single VIP that resolves to any RAC node's listener.

## References

- Oracle Database Net Services Administrator's Guide 19c
- MOS Doc ID 216416.1 — Configuring Client Files
- MOS Doc ID 2117398.1 — EZCONNECT Plus
