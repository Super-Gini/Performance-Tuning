# V$INSTANCE

## Purpose

One row describing **the current instance** — name, host, startup time, status, edition. Every session sees the same row. Ur-view for identity checks.

## Key Columns

| Column             | Meaning                                          |
| ------------------ | ------------------------------------------------ |
| `INSTANCE_NAME`    | SID / instance name.                             |
| `HOST_NAME`        | OS hostname.                                     |
| `VERSION`          | `19.0.0.0.0`.                                    |
| `VERSION_LEGACY`   | `12.2.0.1.0` style (backward-compat display).    |
| `VERSION_FULL`     | `19.19.0.0.0` including RU.                      |
| `STARTUP_TIME`     | When the instance started.                       |
| `STATUS`           | `STARTED` / `MOUNTED` / `OPEN` / `OPEN MIGRATE`. |
| `PARALLEL`         | `YES` if part of a RAC cluster.                  |
| `INSTANCE_ROLE`    | `PRIMARY_INSTANCE` / `SECONDARY_INSTANCE`.       |
| `ACTIVE_STATE`     | `NORMAL` / `QUIESCING` / `QUIESCED`.             |
| `DATABASE_STATUS`  | `ACTIVE` / `SUSPENDED`.                          |
| `LOGINS`           | `ALLOWED` / `RESTRICTED`.                        |
| `SHUTDOWN_PENDING` | `YES` when shutting down.                        |
| `BLOCKED`          | `YES` if instance is hung.                       |
| `EDITION`          | `EE`, `SE`, `SE2`.                               |
| `CON_ID`           | `0` for non-CDB, `1` for CDB root.               |

## Common Queries

```sql
-- Am I on the right instance?
SELECT instance_name, host_name, version_full, status, startup_time
FROM   v$instance;

-- Uptime
SELECT (SYSDATE - startup_time)*24 hours_up FROM v$instance;

-- Blocked / quiescing?
SELECT instance_name, blocked, active_state, database_status
FROM   v$instance;
```

## Related

- `V$DATABASE` — database-level (not instance).
- `GV$INSTANCE` — same but across all RAC instances.

## References

- Oracle Database Reference 19c — `V$INSTANCE`
