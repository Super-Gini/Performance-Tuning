# sqlnet.ora

## Overview

`sqlnet.ora` is the Oracle Net protocol configuration file — shared between clients and the database server. It controls name resolution order, connection timeouts, encryption, checksumming, tracing, and authentication methods. Located in `$ORACLE_HOME/network/admin/sqlnet.ora` (or `$TNS_ADMIN`).

Server and client machines can each have their own `sqlnet.ora`; the settings apply to whichever role that machine is playing.

## Structure

```
# sqlnet.ora
NAMES.DIRECTORY_PATH = (TNSNAMES, EZCONNECT, LDAP)
NAMES.DEFAULT_DOMAIN = corp.example.com

SQLNET.INBOUND_CONNECT_TIMEOUT = 60
SQLNET.EXPIRE_TIME = 10
SQLNET.SEND_TIMEOUT = 60
SQLNET.RECV_TIMEOUT = 60

SQLNET.AUTHENTICATION_SERVICES = (BEQ, NTS, TCPS)

# Encryption
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, AES192, AES128)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA512, SHA384)

# Wallet path (for TCPS, TDE)
WALLET_LOCATION =
   (SOURCE = (METHOD = FILE)
             (METHOD_DATA = (DIRECTORY = /u01/app/oracle/wallet)))
```

## Key Parameters

### Name Resolution

| Parameter              | Purpose                                                       |
| ---------------------- | ------------------------------------------------------------- |
| `NAMES.DIRECTORY_PATH` | Order of methods: `TNSNAMES`, `EZCONNECT`, `LDAP`, `HOSTNAME` |
| `NAMES.DEFAULT_DOMAIN` | Appended to unqualified aliases                               |

### Timeouts

| Parameter                         | Default | Purpose                             |
| --------------------------------- | ------- | ----------------------------------- |
| `SQLNET.INBOUND_CONNECT_TIMEOUT`  | 60 s    | Server-side handshake timeout       |
| `SQLNET.OUTBOUND_CONNECT_TIMEOUT` | none    | Client TCP connect timeout          |
| `SQLNET.SEND_TIMEOUT`             | none    | Server side send-op timeout         |
| `SQLNET.RECV_TIMEOUT`             | none    | Server side receive-op timeout      |
| `SQLNET.EXPIRE_TIME`              | 0 (off) | Dead connection detection (minutes) |

### Dead Connection Detection

`SQLNET.EXPIRE_TIME = 10` — probes idle sessions every 10 minutes; dead ones get cleaned up. Server-side setting; important for preventing session pileup after client-side abort.

### Encryption (Native, no wallet needed)

```
SQLNET.ENCRYPTION_SERVER = REQUIRED   # REJECTED / ACCEPTED / REQUESTED / REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA512)
```

Client side uses `SQLNET.ENCRYPTION_CLIENT`, etc.

### Authentication

| Value       | Purpose          |
| ----------- | ---------------- |
| `BEQ`       | Bequeath (local) |
| `NTS`       | Windows native   |
| `KERBEROS5` | Kerberos         |
| `TCPS`      | SSL certificate  |
| `RADIUS`    | RADIUS           |

Set `SQLNET.AUTHENTICATION_SERVICES = (KERBEROS5, TCPS)` to require specific methods.

### ADR (Automatic Diagnostic Repository)

```
DIAG_ADR_ENABLED = ON
ADR_BASE = /u01/app/oracle
```

### Tracing (for troubleshooting)

```
TRACE_LEVEL_CLIENT = 16         # 0=OFF, 4=USER, 10=ADMIN, 16=SUPPORT
TRACE_DIRECTORY_CLIENT = /tmp
TRACE_FILE_CLIENT = client.trc
TRACE_UNIQUE_CLIENT = ON
LOG_LEVEL_CLIENT = 16
LOG_FILE_CLIENT = client.log
```

## Restrict Access

```
# Only allow specific hosts (server side)
TCP.VALIDNODE_CHECKING = YES
TCP.INVITED_NODES = (10.0.1.10, 10.0.1.11, 10.0.1.0/24)
TCP.EXCLUDED_NODES = (0.0.0.0/0)
```

Requires listener restart. Blocks TCP-level connect attempts.

## Applying Changes

- Client changes: pick up on **next new connection** — no restart.
- Server timeout / auth changes affecting listener: `lsnrctl reload`.
- Node checking / TCP.VALIDNODE: listener restart.

## Diagnostic

```bash
# Verify settings recognized
tnsping <alias> 3

# Server-side check
lsnrctl show inbound_connect_timeout

# Client-side test
sqlplus hr/pwd@service
```

## Common Issues

- **`ORA-12154`** — name resolution failed. Check `NAMES.DIRECTORY_PATH` order and file locations.
- **`ORA-12170: TNS: connect timeout`** — `OUTBOUND_CONNECT_TIMEOUT` fired.
- **`ORA-12547: TNS: lost contact`** — Server-side kill or timeout mid-connection.
- **`ORA-28860: Fatal SSL error`** — Wallet or cert misconfigured (TCPS).
- **Dead sessions accumulating** — Set `SQLNET.EXPIRE_TIME`.

## Best Practices

1. **Set `SQLNET.EXPIRE_TIME = 10`** (server) — cleans up dead sessions.
2. **`SQLNET.INBOUND_CONNECT_TIMEOUT = 60`** (server) — prevents lingering sockets.
3. **Native encryption** for data-in-motion: `ENCRYPTION_SERVER = REQUIRED`.
4. **Use TCPS + wallet** for strong authentication + encryption in high-security environments.
5. Restrict registration and connections with `TCP.VALIDNODE_CHECKING`.
6. Enable tracing only during troubleshooting; disable immediately after.
7. Consistent `NAMES.DIRECTORY_PATH` across all clients.
8. Version-control `sqlnet.ora`.
9. Never leave `TRACE_LEVEL_CLIENT = 16` on in production — huge disk consumption.

## Interview Questions

1. **Q:** What is `sqlnet.ora` for?
   **A:** Oracle Net protocol settings for name resolution, timeouts, encryption, authentication, and tracing.

2. **Q:** `NAMES.DIRECTORY_PATH` — what does it control?
   **A:** Order of name resolution methods (TNSNAMES, EZCONNECT, LDAP, HOSTNAME).

3. **Q:** `SQLNET.EXPIRE_TIME`?
   **A:** Server-side dead-connection detection interval in minutes. Non-zero enables periodic client keepalive.

4. **Q:** How do you require TLS encryption?
   **A:** Set `SQLNET.ENCRYPTION_SERVER = REQUIRED` for native; or use TCPS with wallet.

5. **Q:** `TCP.VALIDNODE_CHECKING`?
   **A:** Restricts which client IPs can connect. Requires listener restart to take effect.

6. **Q:** How to enable client tracing?
   **A:** `TRACE_LEVEL_CLIENT = SUPPORT` (16) in client's `sqlnet.ora`. Also set `TRACE_DIRECTORY_CLIENT` and `TRACE_FILE_CLIENT`.

7. **Q:** ADR vs old ORACLE_HOME logs?
   **A:** ADR (Automatic Diagnostic Repository) is 11g+ centralized location: `$ORACLE_BASE/diag/`. Old locations still work if `DIAG_ADR_ENABLED = OFF`.

## References

- Oracle Database Net Services Reference 19c
- Oracle Database Security Guide 19c — Native Network Encryption
- MOS Doc ID 2019175.1 — Node Filtering
- MOS Doc ID 730066.1 — SQLNET.EXPIRE_TIME
