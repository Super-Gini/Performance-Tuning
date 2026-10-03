# SQL Tuning

Beyond [SQL Optimizer](../11-sql-optimizer/index.md) — the tools and techniques for diagnosing and fixing specific slow SQL. Focus: tracing, plan analysis, targeted interventions.

## Contents

| Page                                                  | Purpose                           |
| ----------------------------------------------------- | --------------------------------- |
| [SQL Trace](sql-trace.md)                             | Legacy + 10046 tracing            |
| [TKPROF](tkprof.md)                                   | Reading trace files into reports  |
| [SQLT / SQLTXPLAIN](sqlt.md)                          | Oracle's SQL Test Case Builder    |
| [SQLTXPLAIN](sqltxplain.md)                           | The XPLAIN standalone             |
| [SQLHC](sqlhc.md)                                     | SQL Health Check                  |
| [Bind Peeking](bind-peeking.md)                       | The bind-first-value optimization |
| [Adaptive Cursor Sharing](adaptive-cursor-sharing.md) | ACS behavior + tuning             |

## Approach

Modern SQL tuning is:

1. **Identify** the SQL — `V$SQL`, ASH, AWR.
2. **Understand** — read plan (DBMS_XPLAN), check stats, look at actual runtime (SQL Monitor).
3. **Diagnose** — where does time go? (User I/O, CPU, wait, join method).
4. **Intervene** — index, hint, SPB, statistics, rewrite.
5. **Verify** — same plan, better time, stable across binds.

## Related

- [SQL Optimizer](../11-sql-optimizer/index.md).
- [Performance Tuning](../12-performance-tuning/index.md).
- [Top SQL](../12-performance-tuning/top-sql.md).
- [SQL Monitor](../12-performance-tuning/sql-monitor.md).
