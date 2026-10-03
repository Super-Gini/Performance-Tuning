# SQL Optimizer

The **Cost-Based Optimizer (CBO)** is the component that turns your SQL text into an execution plan. It considers table cardinalities, column selectivity, index availability, join methods, parallel options, and system statistics, then picks the plan it estimates will cost the least I/O + CPU. Every performance problem is, at some level, an optimizer decision — either the optimizer chose a bad plan, or it was fed bad statistics.

## Contents

| Page                                            | Purpose                                               |
| ----------------------------------------------- | ----------------------------------------------------- |
| [Cost Based Optimizer](cost-based-optimizer.md) | CBO fundamentals — how it computes cost               |
| [Execution Plans](execution-plans.md)           | Reading `EXPLAIN PLAN`, `DBMS_XPLAN`, real-time plans |
| [Statistics](statistics.md)                     | `DBMS_STATS`, table/column/index stats, system stats  |
| [Histograms](histograms.md)                     | Frequency, top-frequency, hybrid, height-balanced     |
| [Cardinality](cardinality.md)                   | How CBO estimates row counts and where it goes wrong  |
| [Dynamic Sampling](dynamic-sampling.md)         | Runtime sampling when stats are missing or stale      |
| [Adaptive Plans](adaptive-plans.md)             | 12c+ plan switching at runtime                        |
| [SQL Plan Management](sql-plan-management.md)   | SPM concepts — baselines, capture, evolution          |
| [SQL Plan Baselines](sql-plan-baselines.md)     | Baseline lifecycle                                    |
| [SQL Profiles](sql-profiles.md)                 | SQL Tuning Advisor-generated hints                    |

## Related

- [Performance Tuning](../12-performance-tuning/index.md) — AWR, ASH, ADDM.
- [SQL Tuning](../36-sql-tuning/index.md) — SQL Trace, TKPROF, SQLT.
