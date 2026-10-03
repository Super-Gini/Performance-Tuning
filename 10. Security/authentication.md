# Authentication

## Overview

**Authentication** is verifying "who is this?" before granting access. Oracle supports multiple methods; you choose based on organizational identity management, regulatory requirements, and application architecture. Password auth is universal; enterprise deployments combine it with LDAP, Kerberos, or certificate-based (TCPS) authentication.

## Methods

### 1. Database Password (default)

```sql
CREATE USER hr_app IDENTIFIED BY "StrongPwd!";
```

Password hash stored in `SYS.USER$`. Verified locally.

### 2. External / OS

```sql
CREATE USER OPS$ORACLE IDENTIFIED EXTERNALLY;
```

OS username must be `<os_authent_prefix><username>` (default prefix `OPS$`). Login uses OS credentials — no password stored. Legacy pattern; usable for local admin scripts.

Parameter: `os_authent_prefix = 'OPS$'` (default). Set to `''` to disable prefix.

### 3. Global (LDAP + Enterprise User Security)

```sql
CREATE USER hr_app IDENTIFIED GLOBALLY AS 'CN=hr_app,OU=DB,DC=corp,DC=example,DC=com';
```

User authenticates via directory service (OID / OUD / AD via bridge). Centralized identity; per-user credentials never touch the database.

### 4. TCPS (SSL Certificate)

```sql
CREATE USER hr_app IDENTIFIED GLOBALLY AS 'CN=hr_app,OU=DB,DC=corp,DC=example,DC=com';
```

TCPS listener presents certificate; database validates via wallet. Requires client wallet with matching cert. Strong mutual authentication + encryption.

See [Network Encryption](network-encryption.md) and [Oracle Wallet](oracle-wallet.md).

### 5. Kerberos

Uses Kerberos tickets. See [Kerberos](kerberos.md).

### 6. Proxy

```sql
ALTER USER hr_app GRANT CONNECT THROUGH hr_bridge;
```

Bridge user connects "as" HR_APP without knowing its password. Auditable and useful for schema-only accounts.

### 7. No Authentication (Schema-Only)

```sql
CREATE USER hr_owner NO AUTHENTICATION;
GRANT CREATE SESSION TO hr_owner;
```

Cannot log in directly. Access only via proxy. Best practice for application schema owners (18c+).

## Configuration

### sqlnet.ora

```
SQLNET.AUTHENTICATION_SERVICES = (BEQ, NTS, TCPS, KERBEROS5)
SQLNET.ALLOWED_LOGON_VERSION_SERVER = 12a
```

- `BEQ` — bequeath (local IPC).
- `NTS` — Windows NT native.
- `TCPS` — SSL certificate.
- `KERBEROS5`, `RADIUS`, `NTS` — external.

### ALLOWED_LOGON_VERSION_SERVER

Minimum client password protocol version:

- `8` — legacy 8i (dangerous).
- `10` — 10g/11g case-insensitive.
- `11` — 11g SHA-1.
- `12` — 12c SHA-2 + PBKDF2.
- `12a` — 12c only (strictest).

Set to `12` or `12a` for modern databases.

### Password Hashes in `SYS.USER$`

Columns: `PASSWORD` (10g), `SPARE4` (11g/12c hashes). Storage of legacy 10g hashes is a compliance concern — drop them:

```sql
-- Drop 10g hash for all users
ALTER USER hr_app IDENTIFIED BY VALUES <spare4_value_only>;
-- Or set ALLOWED_LOGON_VERSION_SERVER=12 to reject 10g logins
```

## Diagnostic Queries

```sql
-- User authentication types
SELECT username, authentication_type, account_status
FROM   dba_users
WHERE  common = 'NO';

-- Password hash versions present
SELECT username, password_versions
FROM   dba_users
WHERE  password_versions IS NOT NULL;

-- Failed logins (from audit)
SELECT username, os_username, userhost, returncode, timestamp
FROM   dba_audit_session
WHERE  returncode <> 0
   AND timestamp > SYSDATE - 1
ORDER  BY timestamp DESC;

-- Proxy relationships
SELECT proxy, client, authentication FROM dba_proxies;
```

## Common Issues

- **`ORA-01017: invalid username/password`** — Wrong credentials, missing user, or old client's hash version not allowed.
- **`ORA-28040: no matching authentication protocol`** — `ALLOWED_LOGON_VERSION_SERVER` rejects client's version. Upgrade client or lower server setting (temporarily).
- **`ORA-01031: insufficient privileges`** — After OS auth, user missing CREATE SESSION.
- **`ORA-12547: TNS: lost contact`** — Certificate handshake failure in TCPS.
- **Kerberos ticket expired** — Client needs `kinit`.

## Best Practices

1. **Password + MFA at application tier**, not per-user in DB.
2. Use **schema-only accounts** with proxy for application owners.
3. Enforce complexity via [Password Verification Functions](password-verification-functions.md).
4. **`ALLOWED_LOGON_VERSION_SERVER = 12a`** where clients support it.
5. Drop 10g hashes (compliance).
6. **Kerberos or TCPS** for enterprise SSO.
7. Never disable password verify function on production.
8. Audit failed authentications (`AUDIT SESSION`).
9. Central LDAP with Enterprise User Security for large environments.
10. Rotate SYS password + separation of duty (`SYSBACKUP`, `SYSDG`, `SYSKM`).

## Interview Questions

1. **Q:** What authentication types does Oracle support?
   **A:** Password (default), external (OS), global (LDAP), TCPS (cert), Kerberos, proxy, schema-only.

2. **Q:** Where are password hashes stored?
   **A:** `SYS.USER$` — `PASSWORD` (10g) and `SPARE4` (11g/12c).

3. **Q:** What is `ALLOWED_LOGON_VERSION_SERVER`?
   **A:** Minimum client password protocol version accepted. Higher = stricter.

4. **Q:** Schema-only account?
   **A:** 18c+ user with no password; accessed only via proxy.

5. **Q:** What is proxy authentication?
   **A:** User A connects "as" user B without knowing B's password. Configured via `ALTER USER B GRANT CONNECT THROUGH A`.

6. **Q:** Kerberos advantage?
   **A:** Single sign-on integrated with corporate KDC; no password over wire.

7. **Q:** How do you audit failed logins?
   **A:** Traditional `AUDIT SESSION WHENEVER NOT SUCCESSFUL` or Unified Audit policy on `LOGON`.

## References

- Oracle Database Security Guide 19c — Authentication
- Oracle Database Enterprise User Security Administrator's Guide 19c
- MOS Doc ID 452411.1 — Kerberos Configuration
- MOS Doc ID 2249633.1 — Schema-Only Accounts
