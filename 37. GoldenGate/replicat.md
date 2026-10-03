# GoldenGate Replicat

## Overview

**Replicat** is the target-side process that reads the remote trail file, generates SQL DML/DDL, and applies to the target database. Like Extract, comes in Classic and Integrated modes.

## Prerequisites (Target DB)

```sql
-- Enable GG
ALTER SYSTEM SET enable_goldengate_replication = TRUE SCOPE=BOTH;

-- Target user
CREATE USER ggadmin IDENTIFIED BY <pw>;
GRANT DBA TO ggadmin;
GRANT EXECUTE ON DBMS_GOLDENGATE_AUTH TO ggadmin;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('GGADMIN');
```

## Checkpoint Table

Replicat records its position in a **checkpoint table** for durability:

```
GGSCI> DBLOGIN USERID ggadmin@target_db PASSWORD <pw>
GGSCI> ADD CHECKPOINTTABLE ggadmin.chkpt
```

## Replicat Config — Classic

```
REPLICAT r_prd
USERID ggadmin@target_db, PASSWORD <pw>
DISCARDFILE ./dirrpt/r_prd.dsc, PURGE
ASSUMETARGETDEFS
DDL INCLUDE MAPPED
MAP APP.*, TARGET APP.*;
```

`ASSUMETARGETDEFS` = don't validate source vs target table structure. Faster.

## Replicat Config — Integrated (Recommended for 12c+)

```
REPLICAT r_prd
USERID ggadmin@target_db, PASSWORD <pw>
DBOPTIONS INTEGRATEDPARAMS(parallelism 8)
ASSUMETARGETDEFS
DDL INCLUDE MAPPED

-- Handle-collision to be forgiving on missing/duplicate rows
HANDLECOLLISIONS

MAP APP.ORDERS, TARGET APP.ORDERS;
MAP APP.CUSTOMERS, TARGET APP.CUSTOMERS;
MAP APP.PRODUCTS, TARGET APP.PRODUCTS;
```

Integrated Replicat parallelism 8 = 8 apply servers, dependency-preserving.

## Coordinated Replicat (12.2+)

For advanced parallelism where you want fine control:

```
REPLICAT r_prd COORDINATED
MAXTHREADS 8
USERID ggadmin@target_db, PASSWORD <pw>
...
MAP APP.ORDERS, TARGET APP.ORDERS THREADRANGE(1-4);
MAP APP.CUSTOMERS, TARGET APP.CUSTOMERS THREADRANGE(5-8);
```

Tables split across thread ranges. Threads within a range preserve serial order.

## Add Replicat to GoldenGate

```
GGSCI> ADD REPLICAT r_prd, INTEGRATED, EXTTRAIL ./dirdat/ba, CHECKPOINTTABLE ggadmin.chkpt
GGSCI> START REPLICAT r_prd
```

## Monitoring

```
GGSCI> info all
GGSCI> info replicat r_prd, showch
GGSCI> stats replicat r_prd, latest
GGSCI> lag replicat r_prd
GGSCI> view report r_prd
```

Key numbers:

- **Lag at Chkpt** — how far behind Replicat is (seconds).
- **Applied TPS** — transactions committed per second.
- **Skipped** — rows GG skipped (usually 0; if not, investigate).
- **Discarded** — errors written to discard file.

Healthy: Lag < 10 s, no discards.

## Handling Errors

By default, Replicat abends on the first error. Better to have policies:

```
-- Handle unique constraint violations by ignoring the duplicate
REPERROR (1, IGNORE)      -- ORA-00001 unique constraint
REPERROR (26, DISCARD)    -- generic - move to discard
REPERROR (1403, IGNORE)   -- ORA-01403 no data found

-- Or route to error handler:
REPERROR (DEFAULT, EXCEPTION)
```

Exception mapping — a special table for problem rows:

```
INSERTALLRECORDS
MAP APP.*, TARGET APP.*, EXCEPTIONSONLY;
MAP APP.*, TARGET GG_EXCEPTIONS.*;
```

## Handle Collisions

For initial-load scenarios where target already has some rows:

```
HANDLECOLLISIONS
```

Instructions:

- INSERT that hits unique constraint → convert to UPDATE.
- UPDATE with no target row → convert to INSERT.
- DELETE with no target row → ignore.

Remove `HANDLECOLLISIONS` once caught up.

## Filter and Transform

Column mapping and expressions:

```
MAP APP.ORDERS, TARGET APP.ORDERS_ARCHIVE,
    COLMAP (USEDEFAULTS,
            LOAD_TIME = @DATENOW(),
            NEW_ID = @COMPUTE(ID + 1000000));
```

`WHERE` filter:

```
MAP APP.ORDERS, TARGET APP.ORDERS,
    WHERE (status = 'ACTIVE');
```

## Bidirectional Replication

Avoid loopback with:

```
TRANLOGOPTIONS EXCLUDEUSER ggadmin
```

Extract on the target side excludes changes made by `ggadmin` (which are Replicat's applied changes).

## Common Issues

- **`OGG-00519` (ORA-00001 unique constraint)** — Duplicate row insertion. Use `HANDLECOLLISIONS` during startup; investigate later.
- **`OGG-00867` (table not found)** — Missing DDL on target. Enable DDL replication in Extract + Replicat.
- **Lag growing** — Target IO bound; increase parallelism; check target CPU.
- **Replicat abends on complex column type** — Some LOBs / spatial / XML types need Integrated Replicat.
- **Character set mismatch** — Set `SOURCECHARSET` in Replicat param.

## Related

- [Architecture](architecture.md).
- [Extract](extract.md).
- [Troubleshooting](troubleshooting.md).
