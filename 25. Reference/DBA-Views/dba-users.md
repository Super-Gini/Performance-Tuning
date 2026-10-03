# DBA_USERS

## Purpose

Every database user account and its properties.

## Key Columns

| Column                        | Meaning                                                           |
| ----------------------------- | ----------------------------------------------------------------- |
| `USERNAME`                    | User name.                                                        |
| `USER_ID`                     | Numeric ID.                                                       |
| `ACCOUNT_STATUS`              | `OPEN`, `LOCKED`, `EXPIRED`, `LOCKED(TIMED)`, `EXPIRED & LOCKED`. |
| `LOCK_DATE`                   | When locked.                                                      |
| `EXPIRY_DATE`                 | Password expiry.                                                  |
| `DEFAULT_TABLESPACE`          | Default TS for new objects.                                       |
| `TEMPORARY_TABLESPACE`        | Sort/temp TS.                                                     |
| `CREATED`                     | Account creation.                                                 |
| `PROFILE`                     | Profile name.                                                     |
| `INITIAL_RSRC_CONSUMER_GROUP` | Resource Manager group.                                           |
| `EXTERNAL_NAME`               | External (OS/LDAP) name.                                          |
| `PASSWORD_VERSIONS`           | `10G`, `11G`, `12C` — controls SEC_CASE_SENSITIVE_LOGON.          |
| `EDITIONS_ENABLED`            | `Y`/`N` — editioning.                                             |
| `AUTHENTICATION_TYPE`         | `PASSWORD`, `EXTERNAL`, `GLOBAL`, `NONE`.                         |
| `PROXY_ONLY_CONNECT`          | `Y` if only via proxy user.                                       |
| `COMMON`                      | `YES` = common user (CDB); `NO` = local.                          |
| `LAST_LOGIN`                  | Last successful logon.                                            |
| `INHERITED`                   | For common users in PDBs.                                         |
| `IMPLICIT`                    | Oracle-internal accounts.                                         |

## Common Queries

```sql
-- All non-Oracle-owned accounts
SELECT username, account_status, default_tablespace, temporary_tablespace, created
FROM   dba_users
WHERE  oracle_maintained = 'N'
ORDER  BY username;

-- Locked / expired
SELECT username, account_status, expiry_date, lock_date
FROM   dba_users
WHERE  account_status <> 'OPEN';

-- Password expiring soon
SELECT username, expiry_date, expiry_date - SYSDATE days_left
FROM   dba_users
WHERE  expiry_date IS NOT NULL AND expiry_date < SYSDATE + 14
ORDER  BY expiry_date;

-- Users with no login in 90 days
SELECT username, last_login FROM dba_users
WHERE  last_login < SYSDATE - 90 OR last_login IS NULL;

-- Common vs local (CDB)
SELECT username, common, inherited FROM dba_users ORDER BY common, username;
```

## Related Views

- `USER_USERS` — current user's own row.
- `ALL_USERS` — subset with less info.
- `DBA_ROLE_PRIVS`, `DBA_SYS_PRIVS`, `DBA_TAB_PRIVS` — grants.
- `DBA_PROFILES` — profile settings.

## References

- Oracle Database Reference 19c — `DBA_USERS`
- [Users](../../09-user-management/users.md)
