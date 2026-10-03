# Security

Oracle Database ships with a broad set of security features covering **identity** (authentication), **authorization** (privileges, roles, VPD, Database Vault), **encryption at rest and in motion** (TDE, native crypto, TCPS), **auditing** (unified audit, FGA), and **data masking** (redaction). This section is the DBA's practical guide to configuring and operating each feature — not a replacement for the _Oracle Database Security Guide_, but a distilled reference for the most common tasks.

## Contents

### Identity

| Page                                                                  | Purpose                                    |
| --------------------------------------------------------------------- | ------------------------------------------ |
| [Authentication](authentication.md)                                   | Password, external, global, TCPS, Kerberos |
| [Kerberos](kerberos.md)                                               | Kerberos integration                       |
| [Password Verification Functions](password-verification-functions.md) | Complexity enforcement                     |

### Authorization

| Page                                | Purpose                                       |
| ----------------------------------- | --------------------------------------------- |
| [Authorization](authorization.md)   | Privileges, roles, secure roles               |
| [VPD](vpd.md)                       | Virtual Private Database — row-level security |
| [Database Vault](database-vault.md) | Segregation of duty, realms, command rules    |
| [Data Redaction](data-redaction.md) | On-the-fly column masking                     |

### Encryption

| Page                                        | Purpose                             |
| ------------------------------------------- | ----------------------------------- |
| [TDE](tde.md)                               | Transparent Data Encryption         |
| [Encryption](encryption.md)                 | Overview and algorithm choice       |
| [Network Encryption](network-encryption.md) | Data in motion (native + TCPS)      |
| [Oracle Wallet](oracle-wallet.md)           | Wallet fundamentals                 |
| [Wallets](wallets.md)                       | Auto-login, external password store |

### Auditing

| Page                              | Purpose                                |
| --------------------------------- | -------------------------------------- |
| [Auditing](auditing.md)           | Traditional and Unified Audit overview |
| [Unified Audit](unified-audit.md) | Modern audit framework (12c+)          |

## Related

- [User Management](../09-user-management/index.md)
- [Multitenant](../08-multitenant/index.md) — Common vs local users, PDB lockdown.
