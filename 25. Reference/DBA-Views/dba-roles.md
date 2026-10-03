# DBA_ROLES

## Purpose

All roles defined in the database.

## Key Columns

| Column                | Meaning                                         |
| --------------------- | ----------------------------------------------- |
| `ROLE`                | Role name.                                      |
| `PASSWORD_REQUIRED`   | `YES`/`NO` — role activation requires password. |
| `AUTHENTICATION_TYPE` | `NONE`, `PASSWORD`, `EXTERNAL`, `GLOBAL`.       |
| `COMMON`              | `YES` = common role in CDB.                     |
| `ORACLE_MAINTAINED`   | `Y` = Oracle-owned.                             |
| `INHERITED`           | For PDBs — inherited from CDB.                  |
| `IMPLICIT`            | Oracle-internal roles.                          |

## Common Queries

```sql
-- Non-Oracle roles
SELECT role, password_required, authentication_type, common
FROM   dba_roles
WHERE  oracle_maintained = 'N'
ORDER  BY role;

-- Who has a specific role
SELECT grantee, granted_role, admin_option, default_role
FROM   dba_role_privs
WHERE  granted_role = 'DBA';

-- Role hierarchy (roles granted to other roles)
SELECT grantee, granted_role
FROM   dba_role_privs
WHERE  grantee IN (SELECT role FROM dba_roles);
```

## Related Views

- `DBA_ROLE_PRIVS` — role-to-grantee mapping.
- `ROLE_SYS_PRIVS` — system privs granted to a role.
- `ROLE_TAB_PRIVS` — object privs granted to a role.
- `SESSION_ROLES` — enabled roles for current session.

## References

- Oracle Database Reference 19c — `DBA_ROLES`
- [Roles](../../09-user-management/roles.md)
