# Oracle Wallet

## Overview

An **Oracle Wallet** is a PKCS#12-encrypted container storing:

- **TDE master keys**.
- **TCPS server/client certificates**.
- **External Password Store (SEPS)** — usernames/passwords for connect strings.

Wallets are the trust anchor for TDE, TLS, and Enterprise User Security. Managed via `orapki` (command-line) or `mkstore` (external password store).

## Types

| Type               | File          | Access                           |
| ------------------ | ------------- | -------------------------------- |
| Password-protected | `ewallet.p12` | Prompt for password              |
| Auto-login         | `cwallet.sso` | Any process on any host          |
| Local auto-login   | `cwallet.sso` | Only on the host that created it |

**Local auto-login** is preferred for unattended databases — password-free but tied to host.

## Location

19c preferred: `WALLET_ROOT` parameter:

```sql
ALTER SYSTEM SET wallet_root = '/u01/app/oracle/wallet' SCOPE=SPFILE;
-- Restart
```

Under `WALLET_ROOT`:

- `<wallet_root>/tde/` — TDE keystore
- `<wallet_root>/tls/` — TLS wallet
- `<wallet_root>/dpwallet/` — Data Pump wallet
- `<wallet_root>/okv/` — OKV wallet

Legacy: `WALLET_LOCATION` in `sqlnet.ora`.

## Creation

```bash
# Create wallet
orapki wallet create -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd

# Auto-login (from same host later)
orapki wallet create -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd -auto_login

# Local auto-login
orapki wallet create -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd -auto_login_local
```

## Adding Certificates

### Self-signed for testing

```bash
orapki wallet add -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -dn 'CN=dbhost.corp.example.com' \
  -keysize 2048 -self_signed -validity 365
```

### CA-signed

```bash
# Generate CSR
orapki wallet add -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -dn 'CN=dbhost.corp.example.com' -keysize 2048

orapki wallet export -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -dn 'CN=dbhost.corp.example.com' \
  -request /tmp/dbhost.csr

# Send CSR to your CA; get back signed cert

# Import CA root
orapki wallet add -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -trusted_cert -cert /tmp/ca_root.pem

# Import intermediate (if any)
orapki wallet add -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -trusted_cert -cert /tmp/ca_intermediate.pem

# Import signed cert
orapki wallet add -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd \
  -user_cert -cert /tmp/dbhost.pem
```

### List contents

```bash
orapki wallet display -wallet /u01/app/oracle/wallet/tls -pwd StrongPwd
```

## External Password Store (SEPS)

Store connect passwords in a wallet so scripts don't embed cleartext:

```bash
mkstore -wrl /u01/app/oracle/wallet/seps -createALO

mkstore -wrl /u01/app/oracle/wallet/seps -createCredential PROD_MONITOR monuser MonPwd
```

`sqlnet.ora`:

```
WALLET_LOCATION =
  (SOURCE = (METHOD = FILE)
            (METHOD_DATA = (DIRECTORY = /u01/app/oracle/wallet/seps)))
SQLNET.WALLET_OVERRIDE = TRUE
```

Then:

```bash
sqlplus /@PROD_MONITOR
# No password needed — pulled from wallet
```

Great for RMAN scripts, monitoring, batch jobs.

## Permissions

- Wallet directory: `oracle:oinstall 0750`
- Wallet files: `oracle:oinstall 0640`
- Never world-readable.

## Backup

```bash
# Simple OS backup of the wallet
tar czf wallet_backup_$(date +%F).tgz /u01/app/oracle/wallet/tde/
```

**Store off-site.** Loss of the TDE wallet = loss of encrypted data.

## Diagnostic Queries

```sql
-- Wallet status
SELECT wrl_type, wrl_parameter, status, wallet_type, keystore_mode
FROM   v$encryption_wallet;

-- TDE keystore contents (for TDE wallet)
SELECT masterkey_id, tag, activation_time, key_use
FROM   v$encryption_keys
ORDER  BY activation_time DESC;
```

## Common Issues

- **`ORA-28365: wallet not open`** — Password-protected wallet needs `ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN`.
- **`ORA-28421: could not open configured wallet`** — Path wrong, permissions, or `WALLET_ROOT` mismatched.
- **`ORA-46620: keystore file not found`** — File missing at expected path.
- **RAC — wallet not shared** — Copy or use ASM-based storage.
- **`ORA-28362: master key not found`** — Restored wrong wallet; keys don't match.

## Best Practices

1. **`WALLET_ROOT`** — modern location parameter.
2. **Local auto-login** for TDE — unattended startup, host-tied.
3. **Back up wallets** before every master key rotation.
4. Store wallets on separate storage from datafiles.
5. **RAC**: replicate wallet to all nodes or use shared storage/OKV.
6. Use CA-signed certs, not self-signed.
7. Rotate certs annually.
8. Never commit wallet files to version control.
9. Monitor `V$ENCRYPTION_WALLET.STATUS` — alert if not OPEN.
10. Test disaster recovery: restore from wallet backup periodically.

## Interview Questions

1. **Q:** What is an Oracle Wallet?
   **A:** PKCS#12 container for TDE master keys, TLS certs, and external passwords.

2. **Q:** Wallet types?
   **A:** Password-protected, auto-login, local auto-login.

3. **Q:** `orapki` vs `mkstore`?
   **A:** `orapki` for general wallet + cert management. `mkstore` for the External Password Store.

4. **Q:** Where should wallets go in 19c?
   **A:** Under `WALLET_ROOT` parameter path.

5. **Q:** RAC wallet strategy?
   **A:** Copy to each node's local path or use shared storage/OKV.

6. **Q:** Loss of wallet with TDE data?
   **A:** Catastrophic — cannot decrypt. Always backup wallets.

## References

- Oracle Database Advanced Security Guide 19c
- Oracle Database Security Guide 19c
- MOS Doc ID 2314601.1 — WALLET_ROOT
- MOS Doc ID 340559.1 — Oracle Wallet Management
