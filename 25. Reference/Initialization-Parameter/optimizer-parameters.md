# Optimizer Parameters

## Overview

Init parameters that shape the **Cost-Based Optimizer's** decisions. Set the wrong one and every query plan flips.

## The Most-Touched Parameters

| Parameter                              | Default 19c           | Purpose                                                                 |
| -------------------------------------- | --------------------- | ----------------------------------------------------------------------- |
| `OPTIMIZER_MODE`                       | `ALL_ROWS`            | `ALL_ROWS`, `FIRST_ROWS_n`. Choose ALL_ROWS almost always.              |
| `OPTIMIZER_FEATURES_ENABLE`            | `19.1.0`              | Enables features for that version. Rollback lever after RU.             |
| `OPTIMIZER_INDEX_COST_ADJ`             | `100`                 | Multiplier on index access cost. 100 = default. Lower = prefer indexes. |
| `OPTIMIZER_INDEX_CACHING`              | `0`                   | Assumed % of index blocks cached. Higher = prefer NL joins.             |
| `OPTIMIZER_ADAPTIVE_PLANS`             | `TRUE`                | 12c+ adaptive join methods.                                             |
| `OPTIMIZER_ADAPTIVE_STATISTICS`        | `FALSE` (19c default) | Dynamic sampling & feedback. Often causes plan flips.                   |
| `OPTIMIZER_DYNAMIC_SAMPLING`           | `2`                   | Level 0–11.                                                             |
| `OPTIMIZER_USE_SQL_PLAN_BASELINES`     | `TRUE`                | Enforce SPB.                                                            |
| `OPTIMIZER_CAPTURE_SQL_PLAN_BASELINES` | `FALSE`               | Automatic capture.                                                      |
| `OPTIMIZER_USE_INVISIBLE_INDEXES`      | `FALSE`               | Include invisible indexes in cost.                                      |
| `CURSOR_SHARING`                       | `EXACT`               | `EXACT`, `FORCE`. Only use FORCE as a bandaid.                          |
| `DB_FILE_MULTIBLOCK_READ_COUNT`        | `8`–`128`             | Full scan MBRC. Usually auto-tuned.                                     |
| `PARALLEL_DEGREE_POLICY`               | `MANUAL`              | `MANUAL`, `LIMITED`, `AUTO`, `ADAPTIVE`.                                |
| `PARALLEL_MIN_TIME_THRESHOLD`          | `AUTO`                | Auto-DOP threshold.                                                     |
| `RESULT_CACHE_MODE`                    | `MANUAL`              | `MANUAL`, `FORCE`.                                                      |
| `STAR_TRANSFORMATION_ENABLED`          | `FALSE`               | Star transform.                                                         |
| `_OPTIMIZER_ADAPTIVE_CURSOR_SHARING`   | (hidden)              | ACS toggle.                                                             |

## Advanced / Hidden (See [Underscore Parameters](../hidden-parameters/underscore-parameters.md))

Sometimes set to work around bugs:

- `_optimizer_use_feedback` — statistics feedback (12c/19c cardinality feedback).
- `_optimizer_null_aware_antijoin` — legacy compat.
- `_fix_control` — enable/disable specific optimizer fixes by bug#.

## Session-Level Tuning

You can override per-session for testing:

```sql
ALTER SESSION SET optimizer_mode = 'FIRST_ROWS_10';
ALTER SESSION SET optimizer_features_enable = '12.2.0.1';
ALTER SESSION SET "_optimizer_use_feedback" = FALSE;
```

## Common Fixes

- **After 19c RU, plans flip** — `ALTER SYSTEM SET optimizer_features_enable = '19.15.0' SCOPE=BOTH;` while investigating.
- **NL vs hash plan flapping** — set `OPTIMIZER_INDEX_CACHING` after measuring `db block gets` vs `physical reads`.
- **Adaptive plans instability** — many sites set `OPTIMIZER_ADAPTIVE_STATISTICS=FALSE` (19c default) and rely on baselines instead.

## Views

```sql
-- Which fixes are enabled?
SELECT bugno, description, value
FROM   v$system_fix_control
WHERE  is_default = 'FALSE';   -- non-default

-- Parameter change history
SELECT * FROM v$parameter_valid_values WHERE name = 'optimizer_mode';
```

## References

- Oracle Database SQL Tuning Guide 19c
- MOS Doc ID 271196.1 — Optimizer parameter changes across versions
- [SQL Plan Management](../../11-sql-optimizer/sql-plan-management.md)
