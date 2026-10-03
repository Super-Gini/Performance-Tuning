# User Management

User management in Oracle covers **users** (accounts), **profiles** (password + resource policy), **roles** (privilege bundles), the **Resource Manager** (workload prioritization), and the **password file** (SYSDBA authentication). This section is about identity and authorization, not application-level access control (see [Security](../10-security/index.md)).

## Contents

| Page                                    | Purpose                                                         |
| --------------------------------------- | --------------------------------------------------------------- |
| [Users](users.md)                       | CREATE USER, quotas, authentication types, schema-only accounts |
| [Roles](roles.md)                       | System roles, application roles, secure roles                   |
| [Profiles](profiles.md)                 | Password policy, resource limits                                |
| [Resource Manager](resource-manager.md) | Consumer groups, plans, per-service QoS                         |
| [Password File](password-file.md)       | SYSDBA/SYSOPER remote auth (`orapw<sid>`)                       |

## Related

- [Security](../10-security/index.md) — auditing, authentication methods, TDE.
- [Multitenant](../08-multitenant/index.md) — common vs local users.
