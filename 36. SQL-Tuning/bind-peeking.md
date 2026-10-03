# Bind Peeking

## Overview

**Bind peeking** — the optimizer's practice of looking at the actual values of bind variables during the **first hard parse** to choose an execution plan. Introduced in 9i, it enables histograms to influence plans — but comes with side effects when the "peeked" value isn't representative.

## Why It Exists

Without bind peeking:

- Optimizer sees `WHERE status = :b1` and picks a plan for average selectivity.
- Skewed data (99% "ACTIVE", 1% "ARCHIVED") gets a mediocre plan for both.

With bind peeking:

- Optimizer looks at the actual value of `:b1` at first parse.
- Uses histograms to pick an optimal plan for that value.
- Reuses the plan for subsequent executions.

## The Problem

If the first execution has an atypical value (`:b1 = 'ARCHIVED'`), the plan is optimized for that. All subsequent executions with `'ACTIVE'` inherit the wrong plan.

Classic symptom: query is fast one day, slow the next, no code change.

## When Bind Peeking Fires

- **First hard parse** of a SQL statement.
- After cursor invalidation (DDL, `DBMS_STATS`, memory pressure).
- After manual `FLUSH SHARED_POOL`.

## Detecting Bind Peeking Issues

```sql
-- Multiple plans for the same SQL_ID over time
SELECT sql_id, plan_hash_value, COUNT(*) execs,
       ROUND(AVG(elapsed_time_delta/executions_delta/1e6), 3) avg_sec
FROM   dba_hist_sqlstat
WHERE  sql_id = '&sql_id' AND executions_delta > 0
GROUP  BY sql_id, plan_hash_value
ORDER  BY avg_sec;
```

If multiple plans + wildly different average timings, bind peeking is a suspect.

### V$SQL

```sql
SELECT sql_id, child_number, plan_hash_value,
       is_bind_sensitive, is_bind_aware, is_shareable
FROM   v$sql
WHERE  sql_id = '&sql_id'
ORDER  BY child_number;
```

- `IS_BIND_SENSITIVE = Y` — optimizer thinks binds matter (histograms present).
- `IS_BIND_AWARE = Y` — [Adaptive Cursor Sharing](adaptive-cursor-sharing.md) is generating multiple bind-aware children.

## The Init Parameter

```sql
SHOW PARAMETER _optim_peek_user_binds
```

`TRUE` = bind peeking ON (default). Setting to `FALSE` disables peeking — plans use column-level defaults, not peeked bind values.

**Rarely** the right fix; the collateral damage (worse plans for skewed data) is usually worse than the flapping.

## Fixes

### 1. Adaptive Cursor Sharing (12c+)

Let Oracle generate bind-aware children automatically. Nothing to configure — it's on by default.

### 2. SQL Plan Baseline

Load the "good" plan as a baseline; ACS and bind peeking can't override:

```sql
DECLARE
  n NUMBER;
BEGIN
  n := DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE(
         sql_id => '&sql_id',
         plan_hash_value => &plan_hash_value);
END;
/
```

### 3. Hint

For a critical query, force a specific access:

```sql
SELECT /*+ INDEX(o orders_pk) */
       * FROM orders o WHERE order_id = :b1;
```

Or:

```sql
SELECT /*+ USE_HASH(a b) */ ... ;
```

### 4. Rewrite

If the query is really doing two very different things depending on bind, split into two queries:

```sql
-- Instead of
SELECT * FROM orders WHERE status = :b1 AND order_dt > :b2;

-- Two:
SELECT * FROM orders WHERE status = 'ACTIVE' AND order_dt > :b2;
SELECT * FROM orders WHERE status = 'ARCHIVED' AND order_dt > :b2;
```

Application picks which to run based on which case.

### 5. Statistics Feedback / Cardinality Feedback

Newer versions may auto-adjust on second execution. But this is unstable — many sites disable via `_optimizer_use_feedback=FALSE`.

## Detecting a Bad Peek

Check the CBO trace (10053) — it explicitly shows the peeked value:

```
Peeked values of the binds in SQL statement
=======================================================
Bind#0
  oacdty=01 mxl=32(20) mxlc=00 mal=00 scl=00 pre=00
  oacflg=03 fl2=0000
  ...
  value="ARCHIVED"
```

Then check `DBA_TAB_COL_STATISTICS` histogram — is "ARCHIVED" the rare value?

## Best Practices

1. **Enable ACS** (default in 12c+).
2. **Use SPB** for queries where plan stability matters more than optimality.
3. **Don't disable bind peeking globally** — small chance of any query being right.
4. **Design histograms carefully** — bind peeking + histograms = double-edged.
5. Monitor `DBA_HIST_SQLSTAT` for plan variance week over week.

## Interview Framing

> "Query was slow after a shared pool flush, then fast again the next day."

That's a bind peeking story. First hard parse got a bad peek; some later invalidation re-peeked to a better value.

## Related

- [Adaptive Cursor Sharing](adaptive-cursor-sharing.md).
- [SQL Plan Management](../11-sql-optimizer/sql-plan-management.md).
- [Histograms](../11-sql-optimizer/histograms.md).
- [Cursor Internals](../32-internals-deep-dive/cursor-internals.md).
