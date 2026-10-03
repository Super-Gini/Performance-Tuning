# Recycle Bin

## Overview

The **Recycle Bin** is a per-user logical container that stores dropped tables (and their dependencies) until the space is truly needed. It's Oracle's "trash can" — a `DROP TABLE` doesn't actually delete anything; it renames the objects to system-generated names and marks them logically dropped. A subsequent `FLASHBACK TABLE ... TO BEFORE DROP` restores them.

Enabled by default (`recyclebin=ON`).

## How It Works

```sql
DROP TABLE hr.employees;
```

Oracle:

1. Renames the table to `BIN$xxxxxxxx==$0`.
2. Renames indexes similarly.
3. Records the mapping in the recycle bin metadata.
4. Space stays allocated; blocks reusable when the tablespace needs them.

```sql
-- List
SHOW RECYCLEBIN
-- or
SELECT * FROM user_recyclebin;
```

## Restore

```sql
FLASHBACK TABLE hr.employees TO BEFORE DROP;

-- Rename during restore
FLASHBACK TABLE hr.employees TO BEFORE DROP RENAME TO employees_v1;
```

Restores table, indexes, triggers, constraints (with system-generated names for referential constraints unless you drop and recreate).

## When Recycle Bin Doesn't Help

- `DROP TABLE ... PURGE` — bypasses recycle bin entirely.
- `TRUNCATE TABLE` — not tracked.
- `DROP TABLESPACE` — recycles everything in that tablespace.
- SYS-owned objects — no recycle bin.
- Tables in SYSTEM tablespace.

## Purging

Manual purge for space:

```sql
-- Specific object
PURGE TABLE hr.employees;
PURGE INDEX employees_pk;

-- Everything for current user
PURGE RECYCLEBIN;

-- Everything (all users) — SYS only
PURGE DBA_RECYCLEBIN;

-- Everything in a tablespace
PURGE TABLESPACE users;
```

Automatic: Oracle purges recycle bin entries when a tablespace needs space to satisfy new allocations.

## Space Considerations

Recycle bin objects consume tablespace quota. If a schema is at quota limits, DROP still succeeds but the recycle bin object counts against the quota — subsequent allocations may fail with `ORA-01536: space quota exceeded`.

## Multitenant

Per-PDB. Each PDB has its own recycle bin. Common users have a per-PDB view when switching containers.

## Disabling (Rarely)

```sql
ALTER SYSTEM SET recyclebin = OFF SCOPE=SPFILE;
-- Restart
```

Session-level:

```sql
ALTER SESSION SET recyclebin = OFF;
```

Do not disable in production unless you have another safety net (RMAN + Flashback Database).

## Diagnostic Queries

```sql
-- Current user's bin
SELECT object_name, original_name, type, ts_name,
       droptime, space
FROM   user_recyclebin;

-- All users
SELECT owner, object_name, original_name, type, ts_name,
       droptime, ROUND(space*8/1024, 1) AS mb  -- assuming 8K blocks
FROM   dba_recyclebin
ORDER  BY droptime DESC;

-- Space held by recycle bin
SELECT ts_name, ROUND(SUM(space)*8/1024, 1) AS mb
FROM   dba_recyclebin
GROUP  BY ts_name
ORDER  BY 2 DESC;
```

## Common Issues

- **Object with same name recycled** — Recycle bin keeps ordinals; newest wins on `FLASHBACK TO BEFORE DROP` unless you name it.
- **Referential constraints renamed** — After restore, FK constraint names differ; rename manually if standards required.
- **Recycle bin fills tablespace quota** — Purge or enlarge quota.
- **`DROP ... PURGE` habit** — Loses recycle bin safety net.

## Best Practices

1. **Leave recycle bin ON** in production.
2. Educate developers: use `DROP TABLE x` not `DROP TABLE x PURGE`.
3. Set alert on `dba_recyclebin` space usage per tablespace.
4. Purge intentionally when reclaiming space, not by default.
5. When restoring, use `RENAME TO` for clarity.
6. Keep recycle bin as first safety net; Flashback Query / Table / DB behind it.
7. Regularly review recycle bin content — identify unintended drops.

## Interview Questions

1. **Q:** What does DROP TABLE do with recycle bin?
   **A:** Renames the table to `BIN$...` and records the mapping. Space stays until purged or reclaimed.

2. **Q:** How to restore?
   **A:** `FLASHBACK TABLE <name> TO BEFORE DROP;` — optionally `RENAME TO`.

3. **Q:** DROP TABLE PURGE?
   **A:** Bypasses recycle bin — permanent drop.

4. **Q:** SYS objects in recycle bin?
   **A:** No — SYS drops are permanent.

5. **Q:** How is space reclaimed automatically?
   **A:** Oracle purges recycle bin when tablespace needs space for new allocations.

6. **Q:** In multitenant?
   **A:** Per-PDB recycle bin.

## References

- Oracle Database Administrator's Guide 19c — Recycle Bin
- MOS Doc ID 258670.1 — Flashback Concepts
- MOS Doc ID 219563.1 — Flashback Table
