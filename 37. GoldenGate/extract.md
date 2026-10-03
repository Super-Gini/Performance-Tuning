# GoldenGate Extract

## Overview

**Extract** is the source-side process that reads redo, filters for tables/schemas of interest, transforms to trail records, and writes to a local trail file. Two flavors: **Classic** (legacy) and **Integrated** (recommended).

## Prerequisites (Source DB)

```sql
-- Enable GG on the DB
ALTER SYSTEM SET enable_goldengate_replication = TRUE SCOPE=BOTH;

-- Force logging
ALTER DATABASE FORCE LOGGING;

-- Supplemental logging at DB level
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;

-- User account for Extract
CREATE USER ggadmin IDENTIFIED BY <pw>;
GRANT DBA TO ggadmin;
GRANT EXECUTE ON DBMS_GOLDENGATE_AUTH TO ggadmin;
EXEC DBMS_GOLDENGATE_AUTH.GRANT_ADMIN_PRIVILEGE('GGADMIN');
```

## Per-Table Supplemental Logging

Extract needs enough redo to reconstruct rows. Add schema-level `SCHEMATRANDATA` for full flexibility:

```
GGSCI> DBLOGIN USERID ggadmin PASSWORD <pw>
GGSCI> ADD SCHEMATRANDATA APP
GGSCI> INFO SCHEMATRANDATA APP
```

Or per table:

```
GGSCI> ADD TRANDATA APP.ORDERS
```

## Extract Config — Classic

```
EXTRACT e_prd
USERID ggadmin@prd_source, PASSWORD <pw>
DISCARDFILE ./dirrpt/e_prd.dsc, PURGE
DDL INCLUDE MAPPED
TABLE APP.*;
```

Save as `dirprm/e_prd.prm`.

## Extract Config — Integrated (Recommended)

```
EXTRACT e_prd
USERID ggadmin@prd_source, PASSWORD <pw>

-- Enable integrated mode
LOGALLSUPCOLS

-- Register with DB for logical change capture
INTEGRATED

-- Trail location
EXTTRAIL ./dirdat/aa

-- DDL replication
DDL INCLUDE MAPPED

-- Tables to capture
TABLE APP.ORDERS;
TABLE APP.CUSTOMERS;
TABLE APP.PRODUCTS;
```

## Add Extract to GoldenGate

```
GGSCI> ADD EXTRACT e_prd, INTEGRATED TRANLOG, BEGIN NOW
GGSCI> REGISTER EXTRACT e_prd, DATABASE
GGSCI> ADD EXTTRAIL ./dirdat/aa, EXTRACT e_prd, MEGABYTES 512
GGSCI> START EXTRACT e_prd
```

`BEGIN NOW` = start capturing from current SCN. Alternatives:

- `BEGIN 2026-08-06 00:00:00` — a wall clock time.
- `BEGIN SCN 12345678` — from a specific SCN.

## Data Pump (Distribution) Config

Separate process shipping local trail to remote target:

```
EXTRACT p_prd
USERID ggadmin@prd_source, PASSWORD <pw>
RMTHOST target-host.example.com, MGRPORT 7809
RMTTRAIL ./dirdat/ba
PASSTHRU
TABLE APP.*;
```

`PASSTHRU` = no transformation, just ship. Fast path.

Add:

```
GGSCI> ADD EXTRACT p_prd, EXTTRAILSOURCE ./dirdat/aa
GGSCI> ADD RMTTRAIL ./dirdat/ba, EXTRACT p_prd, MEGABYTES 512
GGSCI> START EXTRACT p_prd
```

## Monitoring

```
GGSCI> info all
GGSCI> info extract e_prd, showch
GGSCI> stats extract e_prd, latest
GGSCI> lag extract e_prd
GGSCI> view report e_prd
GGSCI> view ggsevt         -- event log
```

Key numbers:

- **Lag at Chkpt** — how far behind Extract is (seconds).
- **Time Since Chkpt** — when did Extract last flush its checkpoint.
- **Committed Trans (throughput)** — transactions per second.

Healthy: Lag < 10 s, Time Since Chkpt < 30 s.

## Common Issues

- **`OGG-01031` (log not found)** — Archived redo purged before Extract read it. Increase archive retention or reduce Extract lag.
- **`OGG-02083` (LOB too large)** — LOB won't fit trail buffer. Bump `EXTFILELIMIT`, `MAXTRANSMEMSIZE`.
- **DDL not captured** — Missing `DDL INCLUDE MAPPED` or trigger install (Classic mode).
- **Extract abends on complex data types** — Integrated mode handles these better than Classic.
- **`OGG-00446` (fetch operation failed)** — Row deleted before Extract could fetch full image. Use `FETCHOPTIONS SUPPRESSDUPS`.

## Extract Roles vs Data Pump Roles

Extract writes to local trail; Data Pump ships to remote. Some sites collapse them into a single Extract with `RMTHOST` + `RMTTRAIL` — but two-step is more robust (Data Pump can restart independently, buffer during network outages).

## Related

- [Architecture](architecture.md).
- [Replicat](replicat.md).
- [Troubleshooting](troubleshooting.md).
