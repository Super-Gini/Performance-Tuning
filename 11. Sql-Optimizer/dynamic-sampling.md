# Dynamic Sampling

## Overview

**Dynamic Sampling** (aka **Dynamic Statistics** in 12c+) is CBO's fallback when it needs cardinality but table statistics are missing or insufficient. At parse time, Oracle runs a small internal query to sample the table and compute an estimated cardinality on the fly.

Controlled by `optimizer_dynamic_sampling` (0–11). Default: 2. Level 11 is aggressive — used by **Automatic Dynamic Statistics** for complex queries.

## Sampling Levels

| Level       | Trigger                                                       |
| ----------- | ------------------------------------------------------------- |
| 0           | Off                                                           |
| 1           | Table has no stats, and 1+ nonpartitioned tables in query     |
| 2 (default) | Any table without stats                                       |
| 3           | + Any table where standard selectivity estimate can't be made |
| 4           | + Predicates with `OR`/`IN` conditions                        |
| 5–9         | Increasing sample size                                        |
| 10          | Sample all rows (expensive)                                   |
| 11          | Adaptive — Oracle decides based on cost of sampling           |

## When It Fires

- Missing table statistics (`DBA_TAB_STATISTICS.NUM_ROWS IS NULL`).
- Global temporary tables (usually no persistent stats).
- Complex predicates CBO can't handle statically.
- External tables.
- Automatic Dynamic Statistics (12c+): for complex or expensive plans, kicks in even with valid stats.

## Cost

Dynamic sampling reads blocks from disk at parse time. For hot production SQL parsed thousands of times per hour, sampling overhead adds up. Trade-off: better plan quality vs parse-time cost.

## Session vs System

```sql
-- Per-session
ALTER SESSION SET optimizer_dynamic_sampling = 4;

-- System-wide
ALTER SYSTEM SET optimizer_dynamic_sampling = 4;

-- Hint on a single SQL
SELECT /*+ DYNAMIC_SAMPLING(4) */ ... FROM hr.employees WHERE ...;
```

Hint applies to that statement only.

## Automatic Dynamic Statistics (12c+)

Triggered by:

- Complex predicates (multi-column, function-based).
- Complex query blocks (many joins).
- Plans with high cost estimates.

If the resulting statistics are useful, CBO can even **persist** them for reuse:

```sql
-- Enable automatic dynamic statistics
ALTER SYSTEM SET optimizer_adaptive_statistics = TRUE SCOPE=BOTH;
-- 19c default: FALSE (regression risk)
```

19c defaults are `optimizer_adaptive_plans=TRUE` and `optimizer_adaptive_statistics=FALSE`. Consider enabling adaptive stats after testing.

## Directives

**SQL Plan Directives (SPD)** are a 12c+ mechanism where Oracle records "next time you see this pattern, use dynamic sampling" — persistent across parses:

```sql
-- View existing directives
SELECT directive_id, type, state, reason
FROM   dba_sql_plan_dir_objects
FETCH FIRST 20 ROWS ONLY;

-- Force a directive to be used
EXEC DBMS_SPD.ALTER_SQL_PLAN_DIRECTIVE(
  directive_id => 12345,
  attribute => 'ENABLED',
  value => 'YES');
```

Directives complement extended stats — Oracle may recommend an extension based on directives, or you can drop directives after creating stats.

## Diagnostic Queries

```sql
-- Was dynamic sampling used?
-- In DBMS_XPLAN output, look for "Note":
--   - dynamic statistics used: dynamic sampling (level=4)

-- Or query V$SQL for "dynamic_sampling"
SELECT sql_id, plan_hash_value
FROM   v$sql
WHERE  sql_id = '&sql_id';

-- SQL plan directives
SELECT directive_id, type, state, reason, notes
FROM   dba_sql_plan_directives
WHERE  state <> 'SUPERSEDED'
FETCH FIRST 10 ROWS ONLY;

-- Directive objects (which tables/columns)
SELECT directive_id, owner, object_name, subobject_name,
       object_type, notes
FROM   dba_sql_plan_dir_objects
WHERE  owner = 'HR'
ORDER  BY directive_id;
```

## Common Issues

- **Parse-time overhead** — Level 10 on frequent hard parses hurts. Reduce level or ensure stats exist.
- **Random plan flips** — Sampling result varies each parse; different plans. Consider baselines or fixed extended stats.
- **Directives spawning stats gathering** — Extended stats created automatically; may not always be optimal.
- **12c regression** — Adaptive stats defaulted ON; caused instability. 19c defaults it OFF.

## Best Practices

1. **Always keep proper table stats.** Dynamic sampling is a safety net, not a primary strategy.
2. Set `optimizer_dynamic_sampling=2` (default) except for specific tuning.
3. For **global temp tables**, either populate stats manually via `DBMS_STATS.SET_TABLE_STATS` or accept sampling cost.
4. Use hints (`DYNAMIC_SAMPLING`) surgically.
5. Test adaptive statistics before enabling globally in 19c.
6. Review SQL plan directives quarterly; act on their recommendations by creating extended stats.
7. Do not disable dynamic sampling — level 0 causes failures on tables without stats.

## Interview Questions

1. **Q:** What is dynamic sampling?
   **A:** Parse-time sampling to estimate cardinality when table stats are missing or CBO needs help.

2. **Q:** Levels?
   **A:** 0–11. Default 2. Higher = more sampling, better estimates, higher parse cost.

3. **Q:** When does automatic dynamic statistics kick in?
   **A:** Complex plans, out-of-range predicates, multi-column predicates.

4. **Q:** SPD (SQL Plan Directive)?
   **A:** 12c+ mechanism where Oracle remembers "use dynamic sampling for this pattern" persistently.

5. **Q:** Cost?
   **A:** Parse-time block reads. For high-parse workloads, can add measurable latency.

6. **Q:** 19c defaults?
   **A:** Adaptive plans ON, adaptive statistics OFF. Test before enabling adaptive stats.

## References

- Oracle Database SQL Tuning Guide 19c — Dynamic Statistics
- MOS Doc ID 470577.1 — Dynamic Sampling
- MOS Doc ID 2312911.1 — Adaptive Features in 12c/19c
