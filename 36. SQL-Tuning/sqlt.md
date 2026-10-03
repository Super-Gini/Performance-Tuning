# SQLT (SQLTXPLAIN)

## Overview

**SQLT** (SQL Test Case Builder / SQL Tuning) is a Oracle-provided support tool that collects everything about a SQL statement — plan history, statistics, parameters, environment — into an HTML report and self-contained "test case" that Oracle Support can reproduce.

Distributed on MOS (Doc ID 215187.1). Not shipped with the Oracle Home.

## Why Use SQLT

- **Standard support artifact** — Oracle Support asks for SQLT XECUTE output routinely.
- **Complete context** — plans across snapshots, all bind captures, all environment parameters.
- **Reproducible test case** — a self-contained script that recreates the SQL + its stats + its DDL + its data (obfuscated if needed).
- **Comparison** — compare a good run to a bad run.

## Installation

Download `sqlt.zip` from MOS Doc ID 215187.1. Unzip on the DB host:

```bash
cd /home/oracle/sqlt
unzip sqlt.zip
cd install
sqlplus / as sysdba @sqcreate.sql
```

Creates a `SQLTXPLAIN` schema (owned by `SQLTXADMIN`). Prompts for tablespaces to store SQLT data (typically 500 MB).

## Main Executables

Under `sqlt/run/`:

| Script            | Purpose                                             |
| ----------------- | --------------------------------------------------- |
| `sqltxecute.sql`  | Run against a SQL statement — capture full context. |
| `sqltxtract.sql`  | Extract history for a SQL_ID from V$SQL + AWR.      |
| `sqltxtrxec.sql`  | Extract + Execute.                                  |
| `sqltcompare.sql` | Compare two SQLT runs.                              |
| `sqlthc.sql`      | Health check for a SQL_ID (older SQLHC).            |

## SQLTXTRACT — Non-Invasive (Most Common)

For a SQL_ID that's in the shared pool or AWR:

```sql
sqlplus / as sysdba
@sqlt/run/sqltxtract.sql &sql_id &sqlt_password
```

Produces:

- `sqlt_<snapshot>_main_<sql_id>.zip` — the deliverable.
- Contains multiple HTMLs, CSVs, SQLs.

Load into a browser; main HTML has ~40 tabs covering:

- SQL text, parsed schema, execution history.
- All plans found in V$SQL, AWR, cursor cache.
- Column histograms.
- Table & index stats.
- Init parameters.
- Data Guard / RAC state.
- Query blocks & join order enumeration.

## SQLTXECUTE — Runs the SQL

For a fresh test — SQLT actually runs the SQL with tracing:

```sql
sqlplus <parsing_schema>/pw
@sqlt/run/sqltxecute.sql /home/oracle/mysql.sql &sqlt_password
```

Where `mysql.sql` contains one SQL statement. SQLT:

1. Runs the statement with 10046 + 10053 trace.
2. Collects TKPROF output.
3. Rebuilds the plan.
4. Compares to plans in cursor cache / AWR.
5. Emits comprehensive report.

## Test Case (TC) Export

```sql
@sqlt/utl/sqltmetadata.sql
```

Or from a SQLTXECUTE run — the resulting ZIP contains a `TC` folder:

- `create_stmt.sql` — the SQL.
- `metadata/` — DDL for all objects.
- `stats/` — table/column/index stats.
- `plan_control/` — SPB baseline attempts.
- `xpl.sql` — a self-contained script that recreates the setup.

Deliver this to Oracle Support to reproduce your specific plan behavior.

## Comparison

```sql
@sqlt/utl/sqltcompare.sql <good_sqlt_id> <bad_sqlt_id>
```

Side-by-side of two SQLT runs — same SQL, different behavior. Show what changed (parameter, stats, plan).

## SQLT Configuration

Global switches:

```sql
-- Skip trace generation (faster)
EXEC sqltxadmin.sqlt$a.set_param('trc_directory','');

-- Include AWR history
EXEC sqltxadmin.sqlt$a.set_param('sqlt$_include_awr','YES');

-- Cap the AWR history size
EXEC sqltxadmin.sqlt$a.set_param('sqlt$_max_days_awr','30');
```

## When to Use SQLT

- Oracle Support requested it.
- Complex plan regressions where you want everything in one artifact.
- Need to hand off diagnosis to another team.
- Suspect stats/histograms/parameters combined causing behavior.
- Reproduce production behavior in QA.

## When Not

- Simple bad-plan cases — DBMS_XPLAN + AWR is faster.
- Prod DBs already under stress — SQLTXECUTE runs the SQL again (impact).

## SQLT vs SQLHC

- **SQLT** — deep, comprehensive, produces test case.
- **[SQLHC](sqlhc.md)** — lightweight health check, read-only, quick.

Both authored by Carlos Sierra (Oracle Support). SQLHC is a subset for quick triage.

## Cleanup

SQLT stores its captures in `SQLTXPLAIN` schema. Purge periodically:

```sql
-- Delete captures older than 30 days
EXEC sqltxadmin.sqlt$a.purge_repository(30);
```

## Related

- [SQLHC](sqlhc.md).
- [SQLTXPLAIN standalone](sqltxplain.md).
- [SQL Trace](sql-trace.md).
- MOS Doc ID 215187.1.
