# Performance Tuning

This section covers Oracle's diagnostic and tuning infrastructure — the tools you use to answer "why is this slow?" from **hourly aggregates** (AWR) down to **individual sessions in the last few seconds** (ASH). Reading these correctly, in the right order, and with an understanding of the wait interface is what separates DBA guessing from DBA engineering.

## Contents

### Diagnostic Frameworks (EE + Diagnostic/Tuning Pack)

| Page                          | Purpose                                            |
| ----------------------------- | -------------------------------------------------- |
| [AWR](awr.md)                 | Automatic Workload Repository — hourly snapshots   |
| [ASH](ash.md)                 | Active Session History — 1-second session sampling |
| [ADDM](addm.md)               | Automatic Database Diagnostic Monitor              |
| [SQL Monitor](sql-monitor.md) | Real-time long-running SQL                         |

### Analysis Techniques

| Page                                                      | Purpose                                            |
| --------------------------------------------------------- | -------------------------------------------------- |
| [Wait Events](wait-events.md)                             | The wait interface — what sessions are waiting on  |
| [Top SQL](top-sql.md)                                     | Finding the workhorses (and the offenders)         |
| [CPU Analysis](cpu-analysis.md)                           | System CPU vs Oracle CPU                           |
| [IO Analysis](io-analysis.md)                             | Datafile I/O, redo I/O, TEMP I/O                   |
| [Latch Contention](latch-contention.md)                   | `latch: shared pool`, `cache buffers chains`, etc. |
| [Mutex Contention](mutex-contention.md)                   | `cursor: pin S wait on X`, library cache mutexes   |
| [Parallel Execution Tuning](parallel-execution-tuning.md) | PX plans, downgrades, distribution                 |

## Related

- [SQL Optimizer](../11-sql-optimizer/index.md)
- [SQL Tuning](../36-sql-tuning/index.md) — SQL Trace, TKPROF, SQLT/SQLHC.
- [Locking](../13-locking/index.md)
- [Reference / Wait Events](../25-reference/wait-events/user-io.md)

## The Method

Always follow the same order:

1. **Identify pain**: end-user complaint, dashboard, alerting.
2. **Timeframe**: correlate to AWR snapshots and ASH samples.
3. **Wait events**: what class of resource is being waited on?
4. **Top SQL / sessions**: who is consuming that resource?
5. **Root cause**: plan bad? statistics wrong? config wrong? code wrong?
6. **Fix + verify**: change one thing, measure again.

Do not skip step 3. Chasing a "high CPU" alert without knowing which wait events are the problem leads to hours of wrong turns.
