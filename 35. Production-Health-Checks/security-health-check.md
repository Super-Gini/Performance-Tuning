# Security Health Check

Comprehensive security posture audit. Feeds into SOX / HIPAA / PCI / STIG / CIS compliance.

## 1. Users & Account Status

```sql
-- Every account
SELECT username, account_status, expiry_date, default_tablespace,
       profile, common, inherited, oracle_maintained, last_login
FROM   dba_users
ORDER  BY oracle_maintained, username;

-- Default passwords
SELECT username FROM dba_users_with_defpwd;

-- Idle users
SELECT username, account_status, last_login
FROM   dba_users
WHERE  (last_login < SYSDATE - 90 OR last_login IS NULL)
   AND account_status = 'OPEN'
   AND oracle_maintained = 'N';
```

Verify: no default passwords; idle users locked.

## 2. Privileged Roles

```sql
-- DBA and equivalents
SELECT grantee, granted_role, admin_option, default_role
FROM   dba_role_privs
WHERE  granted_role IN ('DBA','IMP_FULL_DATABASE','EXP_FULL_DATABASE',
                        'SELECT_CATALOG_ROLE','EXECUTE_CATALOG_ROLE',
                        'DELETE_CATALOG_ROLE','SYSDBA','SYSOPER',
                        'SYSKM','SYSBACKUP','SYSDG');

-- SYSDBA/SYSOPER via password file
SELECT * FROM v$pwfile_users WHERE username NOT IN ('SYS','SYSTEM');
```

Verify: minimum SYSDBA / DBA grants; no unnecessary elevation.

## 3. System Privileges

```sql
-- Direct grants
SELECT grantee, privilege, admin_option
FROM   dba_sys_privs
WHERE  grantee NOT IN (SELECT role FROM dba_roles)
   AND grantee NOT IN ('SYS','SYSTEM','DBSNMP','MDSYS')
ORDER  BY grantee;

-- ANY privileges (dangerous)
SELECT grantee, privilege FROM dba_sys_privs
WHERE  privilege LIKE '%ANY%'
   AND grantee NOT IN ('SYS','SYSTEM','DBA');
```

Verify: `ANY` privileges limited; no `GRANT ANY` handed out.

## 4. Password Profiles

```sql
SELECT p.profile, p.resource_name, p.limit
FROM   dba_profiles p
WHERE  resource_name IN ('PASSWORD_LIFE_TIME','PASSWORD_GRACE_TIME',
                         'PASSWORD_REUSE_MAX','PASSWORD_REUSE_TIME',
                         'FAILED_LOGIN_ATTEMPTS','PASSWORD_LOCK_TIME',
                         'PASSWORD_VERIFY_FUNCTION')
ORDER  BY p.profile, p.resource_name;
```

Verify: `DEFAULT` profile has:

- `PASSWORD_LIFE_TIME` <= 90 (per policy).
- `PASSWORD_REUSE_MAX` = 10.
- `FAILED_LOGIN_ATTEMPTS` <= 10.
- `PASSWORD_VERIFY_FUNCTION` set (`ORA12C_STRONG_VERIFY_FUNCTION` or custom).

## 5. Authentication

```sql
SHOW PARAMETER sec_case_sensitive_logon
SHOW PARAMETER sqlnet.allowed_logon_version_server
SHOW PARAMETER sec_max_failed_login_attempts

SELECT username, password_versions FROM dba_users
WHERE  oracle_maintained='N'
ORDER  BY password_versions;
```

Verify: `SEC_CASE_SENSITIVE_LOGON=TRUE`, `ALLOWED_LOGON_VERSION_SERVER=12` (drops old 10G-only clients).

## 6. Encryption at Rest (TDE)

```sql
-- Wallet
SELECT * FROM v$encryption_wallet;

-- Encrypted tablespaces
SELECT tablespace_name, encrypted FROM dba_tablespaces WHERE encrypted='YES';

-- Encrypted columns
SELECT owner, table_name, column_name, encryption_alg
FROM   dba_encrypted_columns;
```

Verify: wallet OPEN; sensitive tablespaces encrypted.

## 7. Encryption in Transit

Check `sqlnet.ora` on server side:

```
SQLNET.ENCRYPTION_SERVER = REQUIRED
SQLNET.ENCRYPTION_TYPES_SERVER = (AES256, AES192, AES128)
SQLNET.CRYPTO_CHECKSUM_SERVER = REQUIRED
SQLNET.CRYPTO_CHECKSUM_TYPES_SERVER = (SHA512, SHA384)
```

Test:

```sql
SELECT network_service_banner FROM v$session_connect_info WHERE sid = SYS_CONTEXT('USERENV','SID');
```

Look for "AES256 Encryption" and "SHA512 Crypto-checksumming".

## 8. Auditing

```sql
-- Standard audit
SELECT * FROM audit_actions;
SELECT * FROM dba_stmt_audit_opts;
SELECT * FROM dba_priv_audit_opts;

-- Unified audit
SELECT policy_name, enabled_option FROM audit_unified_enabled_policies;

-- Recent audit trail volume
SELECT event_timestamp, COUNT(*)
FROM   (SELECT TO_CHAR(event_timestamp,'YYYY-MM-DD') event_timestamp
        FROM unified_audit_trail
        WHERE event_timestamp > SYSDATE - 7)
GROUP  BY event_timestamp
ORDER  BY 1 DESC;
```

Verify:

- Mandatory audits: LOGON failures, DDL on sensitive schemas, SYS actions.
- Unified audit enabled.
- Trail retention aligned with compliance.

## 9. Network Security

```bash
lsnrctl status
```

Verify:

- Not `LOGON_LOGGING` disabled without reason.
- `ADMIN_RESTRICTIONS_LISTENER = ON`.
- `SECURE_REGISTER_LISTENER` limits registration hosts.
- No TCP wrapper bypass.

`sqlnet.ora`:

```
SQLNET.ALLOWED_LOGON_VERSION_SERVER = 12
SQLNET.EXPIRE_TIME = 10
```

## 10. Database Vault

```sql
SELECT * FROM dba_dv_realm;
SELECT * FROM dba_dv_rule_set;
```

If DV is licensed and required, verify realms cover sensitive schemas.

## 11. VPD / Fine-Grained Access

```sql
SELECT object_name, policy_name, function, statement_types, sel_columns
FROM   dba_policies;
```

Verify: policies present on sensitive tables.

## 12. Data Redaction

```sql
SELECT object_name, policy_name, column_name, function_type
FROM   dba_redaction_columns;
```

Verify: PII / sensitive columns redacted for non-privileged users.

## 13. Public Grants (Anti-Pattern)

```sql
-- Any GRANT to PUBLIC on sensitive objects
SELECT owner, table_name, privilege
FROM   dba_tab_privs
WHERE  grantee='PUBLIC'
   AND owner NOT IN ('SYS','SYSTEM','MDSYS','CTXSYS','XDB');
```

Verify: no PUBLIC grants on custom schemas.

## 14. Java in DB

If Java isn't used, disable:

```sql
SELECT comp_id, status FROM dba_registry WHERE comp_id='JAVAVM';

-- Uninstall if unused
@$ORACLE_HOME/rdbms/admin/rmjvm.sql
```

Reduces attack surface.

## 15. DB Links

```sql
SELECT owner, db_link, username, host FROM dba_db_links;
```

Verify: no db_links pointing to insecure targets; passwords use secure external (Kerberos) not embedded.

## 16. Directory Objects

```sql
SELECT owner, directory_name, directory_path FROM dba_directories;

SELECT grantee, privilege FROM dba_tab_privs
WHERE  owner='SYS' AND table_name IN (SELECT directory_name FROM dba_directories);
```

Verify: no `SYS`-owned directories point to sensitive paths; grants limited.

## 17. CIS / STIG Compliance

Run OEM Compliance Framework or scripts against a benchmark:

- CIS Oracle 19c Benchmark.
- DoD STIG Oracle 12c/19c.

## 18. Deliverable

Structure:

- Green: compliant.
- Yellow: partial / warning.
- Red: critical finding.

Per finding: severity, CVSS-like score, evidence, recommendation, owner.

## Related

- [Security chapter](../10-security/index.md).
- [Authentication](../10-security/authentication.md).
- [Authorization](../10-security/authorization.md).
- [Auditing](../10-security/auditing.md).
- [TDE](../10-security/tde.md).
- [Unified Audit](../10-security/unified-audit.md).
