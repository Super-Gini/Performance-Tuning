# Network Encryption

## Overview

Encrypt Oracle Net traffic between clients and the database to prevent eavesdropping and tampering on the wire. Two mechanisms:

1. **Native Network Encryption** — Oracle's proprietary encryption negotiated via SQL\*Net. Fast, no wallet needed for encryption alone.
2. **TCPS (TLS/SSL)** — Standard TLS over port 2484 (typical). Requires wallets on both sides. Provides encryption **and** mutual certificate authentication.

Native encryption is easier to deploy; TCPS is standards-based and works well with cert-based auth (Kerberos alternative).

## Native Network Encryption

### Server sqlnet.ora

```
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, AES192, AES128)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA512, SHA384, SHA256)
```

Values for `ENCRYPTION_SERVER` / `_CLIENT`:

- `REJECTED` — no encryption.
- `ACCEPTED` — accept if requested.
- `REQUESTED` — request; accept if not offered.
- `REQUIRED` — refuse unencrypted.

### Client sqlnet.ora

```
SQLNET.ENCRYPTION_CLIENT = REQUIRED
SQLNET.ENCRYPTION_TYPES_CLIENT = (AES256)
SQLNET.CRYPTO_CHECKSUM_CLIENT = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_CLIENT = (SHA512)
```

### Negotiation

Client and server exchange supported algorithms. Highest common choice used. `REQUIRED` on either side means no plaintext allowed — connection fails if no match.

## TCPS (SSL/TLS)

### Server-side setup

1. **Create wallet with server certificate**:

```bash
orapki wallet create -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd -auto_login

orapki wallet add -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -dn 'CN=dbhost.corp.example.com,OU=DB,O=Corp' \
  -keysize 2048 -self_signed -validity 365

# Export cert (share with clients)
orapki wallet export -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -dn 'CN=dbhost.corp.example.com,OU=DB,O=Corp' \
  -cert /tmp/server_cert.pem
```

2. **listener.ora**:

```
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = dbhost)(PORT = 1521))
      (ADDRESS = (PROTOCOL = TCPS)(HOST = dbhost)(PORT = 2484))
    )
  )

SSL_CLIENT_AUTHENTICATION = TRUE
WALLET_LOCATION =
  (SOURCE = (METHOD = FILE)
            (METHOD_DATA = (DIRECTORY = /u01/app/oracle/wallet/tls)))
```

3. **sqlnet.ora**:

```
WALLET_LOCATION =
  (SOURCE = (METHOD = FILE)
            (METHOD_DATA = (DIRECTORY = /u01/app/oracle/wallet/tls)))

SQLNET.AUTHENTICATION_SERVICES = (TCPS, BEQ)
SSL_CLIENT_AUTHENTICATION = TRUE
SSL_VERSION = 1.2   -- or 1.3 if supported
SSL_CIPHER_SUITES = (TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384)
```

4. **Restart listener**:

```bash
lsnrctl reload
```

### Client-side

1. Import server cert into client wallet (for cert validation).
2. `tnsnames.ora`:

```
PROD_TCPS =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCPS)(HOST = dbhost)(PORT = 2484))
    (CONNECT_DATA = (SERVICE_NAME = prod.corp))
    (SECURITY = (SSL_SERVER_CERT_DN = 'CN=dbhost.corp.example.com,OU=DB,O=Corp')))
```

## Verification

```sql
-- Session-level check
SELECT sys_context('userenv','network_protocol') AS protocol FROM dual;
-- Returns 'tcps' if TLS used

-- Native encryption in use
SELECT s.sid, s.username, s.network_service_banner
FROM   v$session s WHERE s.sid = SYS_CONTEXT('userenv','sid');
```

## Common Issues

- **`ORA-12660: encryption or crypto-checksum mismatch`** — Client and server require different algorithms. Ensure overlap.
- **`ORA-28860: fatal SSL error`** — Wallet missing/wrong cert; check paths and permissions.
- **`ORA-29024: certificate validation failure`** — Client wallet does not trust server cert.
- **Handshake slow** — Cert chain too long or CRL check slow. Consider OCSP stapling or shorter chain.

## Best Practices

1. **Require encryption** in production (`REQUIRED` on both sides).
2. Prefer **AES256** and **SHA512**.
3. Native for internal networks; **TCPS for external / partner** connections.
4. TCPS supports mutual auth — use for admin sessions.
5. Certificate management: use CA-signed, not self-signed, in production.
6. Rotate certs annually; automate with cert management platform.
7. Auto-login wallet (`cwallet.sso`) for unattended servers.
8. Monitor for `ORA-28860` in alert log.
9. Test client connectivity with `openssl s_client -connect dbhost:2484 -showcerts`.

## Interview Questions

1. **Q:** Native encryption vs TCPS?
   **A:** Native: proprietary, easy, encryption only. TCPS: TLS-based, wallet required, supports mutual cert auth.

2. **Q:** How do you require encryption?
   **A:** `SQLNET.ENCRYPTION_SERVER = REQUIRED`.

3. **Q:** Default TCPS port?
   **A:** 2484.

4. **Q:** How do you check if a session is using TLS?
   **A:** `SELECT sys_context('userenv','network_protocol') FROM dual;`.

5. **Q:** `SSL_CLIENT_AUTHENTICATION = TRUE`?
   **A:** Server requires client cert (mutual TLS).

## References

- Oracle Database Advanced Security Guide 19c
- Oracle Database Security Guide 19c — Configuring Network Encryption
- MOS Doc ID 1926511.1 — TCPS Configuration
- MOS Doc ID 218089.1 — Native Encryption
