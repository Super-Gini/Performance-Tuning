# Active Data Guard

## Overview

**Active Data Guard (ADG)** allows a physical standby to be **open READ ONLY** while simultaneously applying redo. Reporting workloads can offload to the standby without impacting primary or falling behind on apply. 19c adds **DML Redirection** — occasional DML on the standby is automatically shipped to the primary.

ADG is a licensed option beyond base Data Guard. Also enables Automatic Block Repair (primary fetches good block from standby on corruption).

## Enabling

```sql
-- On standby, cancel apply
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;

-- Open READ ONLY
ALTER DATABASE OPEN READ ONLY;

-- Restart apply — now open and applying concurrently
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT;
```

Verify:

```sql
SELECT open_mode, database_role FROM v$database;
-- Expect: READ ONLY WITH APPLY, PHYSICAL STANDBY
```

## What Works on ADG

- SELECT queries (all).
- Recursive SQL and PL/SQL blocks that don't modify data.
- Global temporary tables (with `TEMP_UNDO_ENABLED=TRUE` on primary).
- Reporting workloads.

## What Doesn't (Base ADG)

- INSERT, UPDATE, DELETE, MERGE on user tables.
- DDL.
- Sequences.

## DML Redirection (19c)

Session-level or system-level flag redirects occasional DML from standby to primary:

```sql
ALTER SESSION ENABLE ADG_REDIRECT_DML;
-- or system-wide
ALTER SYSTEM SET adg_redirect_dml = TRUE;

-- Now DML on the ADG standby works transparently
INSERT INTO hr.audit_log VALUES (...);   -- redirected to primary
```

Latency cost: each redirected DML is a network round-trip to primary. Fine for occasional DML; not for high-volume writes.

## Sequences on ADG

Standby's sequences can be Session-Private or `GLOBAL`. Set on primary:

```sql
CREATE SEQUENCE my_seq
  START WITH 1 INCREMENT BY 1 CACHE 20
  SESSION;
```

`SESSION` sequences give each ADG session a private range without primary round-trip.

## Automatic Block Repair

If a session on primary hits `ORA-01578` (block corruption), Oracle transparently fetches a good copy from the ADG standby. Zero user-visible impact.

Enabled automatically when ADG is up and healthy.

## Query-Only Load Balancing

Use different services for OLTP (goes to primary) vs reports (goes to ADG standby):

```
srvctl add service -db prod -service prod_oltp -preferred prod -available prod_dr
srvctl add service -db prod -service prod_reports -preferred prod_dr -available prod
```

Application connects via appropriate service.

## Diagnostic Queries

```sql
-- On standby
SELECT open_mode, database_role, protection_mode, protection_level
FROM   v$database;

-- Sessions on standby
SELECT type, username, sql_id, event
FROM   v$session
WHERE  status = 'ACTIVE' AND type = 'USER';

-- ADG DML redirect stats
SELECT name, value FROM v$sysstat WHERE name LIKE 'ADG%';

-- Feature usage (verify license)
SELECT * FROM dba_feature_usage_statistics
WHERE  name IN ('Active Data Guard','Automatic Block Repair');
```

## Common Issues

- **`ORA-01031: insufficient privileges` when using DML redirect** — Session needs the redirected DML privilege on primary too.
- **Long-running query on ADG blocking apply cleanup** — Rare; apply may pause on old-SCN cleanup while a long query holds the SCN.
- **Redirected DML latency dominates** — Wrong tool; move DML to primary directly.
- **License scare** — Confirm Active Data Guard option is licensed before enabling.

## Best Practices

1. **Use ADG for reporting offload** — huge value for read-heavy workloads.
2. **Sequences: `SESSION`** for ADG-heavy patterns.
3. **DML redirect: sparingly.** Design apps around it.
4. Use services + client-side routing.
5. TEMP_UNDO_ENABLED=TRUE for GTT DML on ADG.
6. Match SRL / storage / CPU on standby to reporting workload — it's a real database.
7. Alert on ADG apply lag > 30s.
8. Consider Real Application Testing to measure ADG report performance.
9. Confirm licensing.

## Interview Questions

1. **Q:** What is ADG?
   **A:** Active Data Guard — open R/O standby with concurrent redo apply.

2. **Q:** DML on ADG?
   **A:** Base ADG: no. 19c: with `adg_redirect_dml`, occasional DML redirected to primary.

3. **Q:** Automatic Block Repair?
   **A:** Primary fetches good block from ADG standby when corruption detected. Auto with ADG.

4. **Q:** Sequences?
   **A:** `SESSION` sequences give private ranges to ADG sessions without primary round-trip.

5. **Q:** License?
   **A:** Active Data Guard option (beyond base DG).

## References

- Oracle Data Guard Concepts and Administration 19c — Active Data Guard
- MOS Doc ID 219344.1 — ADG Overview
- MOS Doc ID 2542017.1 — ADG DML Redirection 19c
