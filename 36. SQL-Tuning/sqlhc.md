# SQLHC — SQL Health Check

## Overview

**SQLHC** is a lightweight, read-only SQL analysis tool from Oracle Support. Given a `SQL_ID`, it produces an HTML report on the query's health: plan, stats, bind info, and common issues. Faster than SQLT and safe to run in production.

Distributed on MOS (Doc ID 1366133.1).

## Get It

Download from MOS. Unzip to `/home/oracle/sqlhc/`.

## Run It

```bash
cd /home/oracle/sqlhc

sqlplus / as sysdba
SQL> @sqlhc.sql T &sql_id
```

Modes:

- `T` — Typical (default).
- `X` — Extended (more detail).
- `M` — Metadata only (fast).

Output: `sqlhc_<datetime>_<sql_id>_1_health_check.html` and companions.

## Report Sections

- **Observations** — automated findings (missing stats, skewed data, mistaken index).
- **Environment** — DB, parameters, host.
- **SQL Text**.
- **Execution Statistics** — from V$SQL, AWR.
- **Execution Plans** — every plan the CBO has considered.
- **Object Statistics** — tables, indexes, columns, histograms.
- **Bind Information** — captured binds.
- **SQL Profiles / Baselines / Patches**.
- **Statistics for related tables**.
- **Recommendations**.

## When to Use SQLHC

- Quick "health check" of a specific SQL before diving deeper.
- Non-Oracle-Support cases where you want the same diagnostic depth as SQLT without running the SQL.
- Comparing two SQLs.

## When Not

- Oracle Support requested SQLT specifically — use [SQLT](sqlt.md).
- Need a reproducible test case — use SQLT XECUTE.

## Automated Observations Examples

SQLHC's Observations section catches things like:

- **Statistics missing on a column** used in a predicate.
- **Cardinality mismatch** — actual vs estimate off by 10×+.
- **Bind mismatch** — many child cursors due to bind types.
- **Adaptive plan flipped**.
- **Table stats stale** — last analyzed > 30 days.
- **Index has skew** — high LEAF_BLOCKS to distinct keys ratio.

## SQLHC vs SQLT

| Feature              | SQLHC         | SQLT XTRACT / XECUTE |
| -------------------- | ------------- | -------------------- |
| Ships as             | Loose scripts | Installed package    |
| Modifies DB          | No            | Installs schema      |
| Runs the SQL         | No            | XECUTE runs it       |
| Report depth         | Deep          | Deeper               |
| Test case generation | No            | Yes                  |
| Speed                | Fast          | Slower               |

Rule of thumb: SQLHC for triage, SQLT for handoff to Oracle Support.

## Related

- [SQLT](sqlt.md).
- MOS Doc ID 1366133.1 — SQLHC download.
