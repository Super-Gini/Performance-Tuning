# SQLTXPLAIN (Standalone Utility)

## Overview

**SQLTXPLAIN** is the historical name for the SQLT plan-diagnosis tool — specifically the `SQLTXPLAIN.SQL` script that was the ancestor of the full [SQLT](sqlt.md) tool. Still occasionally referenced in older docs.

Modern practice: use SQLT (`sqltxtract`, `sqltxecute`) rather than the standalone SQLTXPLAIN.

## What Legacy SQLTXPLAIN Does

For a given SQL, produces an HTML report covering:

- The SQL text.
- Current execution plan.
- CBO trace (10053) content.
- Statistics for each object.
- Init parameters.
- Environment.

Modern SQLTXECUTE / SQLTXTRACT does all this + more.

## When You Might Still See It

- Legacy runbooks from 11g / 12c era.
- Oracle Support notes older than 2015.
- Custom in-house shell wrappers that call `SQLTXPLAIN.SQL` directly.

## Migration Path

If your team still uses `SQLTXPLAIN.SQL`, migrate to modern SQLT:

```sql
-- Old
@sqltxplain.sql

-- New
@sqlt/run/sqltxtract.sql &sql_id &sqlt_pw
```

The output structure is similar; SQLT is more comprehensive and includes AWR history.

## Related

- [SQLT](sqlt.md) — the modern tool.
- [SQL Trace](sql-trace.md).
- MOS Doc ID 215187.1.
