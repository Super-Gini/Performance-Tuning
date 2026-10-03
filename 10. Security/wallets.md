# Wallets

## Overview

Beyond the basic wallet concept (see [Oracle Wallet](oracle-wallet.md)), this page covers **wallet variations** encountered in real deployments: SEPS wallets for connection strings, wallet auto-login variants, RAC / Data Guard wallet coordination, and Oracle Key Vault (OKV) integration.

## Wallet Variants Recap

| Variant                             | Purpose                            |
| ----------------------------------- | ---------------------------------- |
| **TDE wallet**                      | Master keys for at-rest encryption |
| **TLS/SSL wallet**                  | Certificates for TCPS              |
| **SEPS wallet**                     | Connect passwords for automation   |
| **Enterprise User Security wallet** | LDAP authentication credentials    |
| **OKV wallet**                      | Bridge to Oracle Key Vault         |

## SEPS — External Password Store

Purpose: keep passwords out of scripts.

### Create

```bash
mkstore -wrl /u01/app/oracle/wallet/seps -createALO   # auto-login
mkstore -wrl /u01/app/oracle/wallet/seps -createCredential PROD_RMAN rman_op RMAN_pwd
mkstore -wrl /u01/app/oracle/wallet/seps -createCredential PROD_MON  monuser Mon_pwd
```

### List

```bash
mkstore -wrl /u01/app/oracle/wallet/seps -listCredential
```

### Use

`sqlnet.ora` on client machine (or `TNS_ADMIN` path):

```
WALLET_LOCATION =
  (SOURCE = (METHOD = FILE)
            (METHOD_DATA = (DIRECTORY = /u01/app/oracle/wallet/seps)))

SQLNET.WALLET_OVERRIDE = TRUE
```

Connect string in `tnsnames.ora` must match:

```
PROD_RMAN =
  (DESCRIPTION = ...)

PROD_MON =
  (DESCRIPTION = ...)
```

Connect:

```bash
sqlplus /@PROD_MON
rman target /@PROD_RMAN
```

Great for cron / monitoring / scheduled RMAN.

## Auto-Login Wallets

### Standard auto-login

`cwallet.sso` alongside `ewallet.p12`. Any process that can read files can open the wallet.

### Local auto-login (18c+)

Tied to the host that created it. Cannot be moved to another server. Preferred security posture.

Create both:

```bash
orapki wallet create -wallet /u01/app/oracle/wallet/tde -pwd Strong_Pwd -auto_login_local
```

Or convert existing:

```sql
ADMINISTER KEY MANAGEMENT CREATE LOCAL AUTO_LOGIN KEYSTORE
  FROM KEYSTORE '/u01/app/oracle/wallet/tde'
  IDENTIFIED BY "Strong_Pwd";
```

## RAC Wallet Coordination

Every RAC node needs access to the wallet. Options:

1. **Per-node copy** — copy `ewallet.p12` and `cwallet.sso` to each `$ORACLE_HOME/dbs` or path pointed to by `WALLET_ROOT`. Simple; hard to keep in sync after rotations.
2. **Shared filesystem (ACFS, NFS)** — one wallet, all nodes read. Requires proper permissions.
3. **ASM** (19c+) — wallet in an ASM diskgroup; nodes access via ASM.
4. **Oracle Key Vault** — centralized key management; every node fetches key material at runtime.

### Recommended for RAC

- Use `WALLET_ROOT` on **shared storage** (ACFS or ASM).
- OR use **OKV** — single source of truth, tamper-evident, audit trail.

## Data Guard Wallet Sync

Standby needs the same TDE master keys as primary. Options:

1. Copy primary wallet to standby (before enabling encryption).
2. Use OKV — both primary and standby fetch from vault.
3. Configure `TDE_CONFIGURATION` to use HSM at both sites.

Failure to sync = standby cannot open encrypted files after switchover. Test in stage first.

## Oracle Key Vault (OKV)

OKV is a hardened Linux appliance that centralizes key management:

- Certificates
- TDE master keys
- SEPS credentials
- Kerberos keytabs

Connects via `okvutil` and `PKCS#11` provider. Recommended for regulated environments (PCI, HIPAA).

Configuration snippet:

```sql
ALTER SYSTEM SET tde_configuration = 'KEYSTORE_CONFIGURATION=OKV' SCOPE=BOTH;

ADMINISTER KEY MANAGEMENT SET KEYSTORE OPEN
  IDENTIFIED BY "OKV_endpoint_pwd";
```

## Diagnostic Queries

```sql
-- TDE wallet state
SELECT wrl_type, wrl_parameter, status, wallet_type, keystore_mode, con_id
FROM   v$encryption_wallet;

-- Key rotation history
SELECT masterkey_id, tag, activation_time, key_use
FROM   v$encryption_keys
ORDER  BY activation_time DESC;
```

## Common Issues

- **`ORA-46658: wallet not opened` after restart** — Auto-login wallet missing. Recreate `cwallet.sso`.
- **`ORA-28362: master key not found`** — Wallet mismatch. Restore correct wallet.
- **RAC standby fails after failover** — TDE wallet not on new primary. Copy from previous primary or use OKV.
- **Wallet permission denied** — File mode wrong. `chmod 640 ewallet.p12 cwallet.sso`.
- **SEPS not working** — `SQLNET.WALLET_OVERRIDE = TRUE` missing or wrong alias.

## Best Practices

1. **Local auto-login** for TDE on database hosts.
2. **SEPS auto-login** for automation.
3. **OKV** for enterprise / regulated environments.
4. **Backup wallets** with every rotation.
5. Never commit wallets to source control.
6. Use ASM- or ACFS-shared wallet for RAC.
7. Coordinate wallets across Data Guard primary + standby.
8. Test wallet restore from backup.
9. Audit wallet access — file-system audits + `V$ENCRYPTION_WALLET` history.
10. Rotate cert/keys per policy (annual).

## Interview Questions

1. **Q:** Auto-login vs local auto-login?
   **A:** Auto-login: usable on any host with the wallet. Local: tied to the host that created it — safer.

2. **Q:** What is SEPS?
   **A:** Secure External Password Store — a wallet holding connect string passwords for scripts.

3. **Q:** How do RAC nodes share TDE keys?
   **A:** Per-node copy, shared filesystem (ACFS), ASM-stored wallet, or Oracle Key Vault.

4. **Q:** Data Guard TDE consideration?
   **A:** Standby must have the same master keys — copy wallet or use central OKV.

5. **Q:** What is Oracle Key Vault?
   **A:** Central appliance for managing TDE keys, certs, SEPS credentials with audit + tamper-evidence.

## References

- Oracle Database Advanced Security Guide 19c
- Oracle Database Security Guide 19c
- Oracle Key Vault Administrator's Guide
- MOS Doc ID 2314601.1 — WALLET_ROOT and TDE_CONFIGURATION
