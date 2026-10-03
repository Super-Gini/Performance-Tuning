# Cardinality

## Overview

**Cardinality** is CBO's estimate of the number of rows a step in the plan will produce. Every join method, access path, and cost calculation depends on it. If cardinality is right, the plan is usually right. If it's wrong, everything else is downstream noise.

This page describes the ways CBO estimates cardinality, the common failure modes, and how to detect and fix them.

## Basic Cardinality Formula

For a single predicate:

- **`col = literal`** — `NumRows × Density` (or histogram bucket if histogram exists).
- **`col > literal`** — Uses low/high value range: `NumRows × (High - literal) / (High - Low)`.
- **`col LIKE 'X%'`** — Roughly 5% of NumRows if no histogram.
- **`col IS NULL`** — `NumNulls`.
- **`col IN (v1,v2,v3)`** — Sum of individual estimates.

Multiple predicates are multiplied (assuming independence).

## Where CBO Goes Wrong

### 1. Correlated Predicates

`WHERE state = 'CA' AND city = 'Los Angeles'`. City is highly correlated with state — everyone in Los Angeles is in California. CBO treats them independently: `P(state=CA) × P(city=LA)` — huge under-estimate.

**Fix**: Column group extended stats:

```sql
DECLARE cg VARCHAR2(30);
BEGIN
  cg := DBMS_STATS.CREATE_EXTENDED_STATS('HR','CUSTOMERS','(STATE,CITY)');
END;
/

EXEC DBMS_STATS.GATHER_TABLE_STATS('HR','CUSTOMERS',
  method_opt=>'FOR ALL COLUMNS SIZE AUTO FOR COLUMNS (STATE,CITY) SIZE AUTO');
```

### 2. Function on Column

`WHERE UPPER(name) = 'JOHN'`. CBO cannot use column statistics on `UPPER(name)` — treats as 1% selectivity by default.

**Fix**: expression statistics or a **function-based index**:

```sql
CREATE INDEX cust_upper_name ON customers (UPPER(name));

-- Or expression stats without index
BEGIN
  DBMS_STATS.CREATE_EXTENDED_STATS('HR','CUSTOMERS','(UPPER(NAME))');
END;
/
```

### 3. Out-of-Range Predicates

`WHERE order_date >= DATE '2027-01-01'` on a table where `HIGH_VALUE = DATE '2026-08-01'`. CBO returns near-0 cardinality; the query returns rows that arrive after stats gathered.

**Fix**: **Real-Time Statistics** (12c+) captures HIGH_VALUE inline, or gather stats more frequently on hot tables. `DBMS_STATS.SET_TABLE_PREFS('METHOD_OPT', ...)` for the strategy.

### 4. Skewed Data Without Histogram

Covered in [Histograms](histograms.md).

### 5. Bind Peeking

CBO peeks bind values at first parse and estimates cardinality for those values. If subsequent binds are extreme, plan is wrong. See [Adaptive Cursor Sharing](../36-sql-tuning/adaptive-cursor-sharing.md).

### 6. Join Cardinality

For an equi-join `A.k = B.k`:

`Card(A ⋈ B) = Card(A) × Card(B) / MAX(NDV(A.k), NDV(B.k))`

Wrong NDVs, wrong join cardinality. Fix: gather column stats.

### 7. Complex OR / IN Lists

Very long IN lists (thousands of values) confuse CBO. Consider using GLOBAL TEMP TABLE + join.

## Diagnostic Approach

### Compare Estimated vs Actual

```sql
ALTER SESSION SET statistics_level = 'ALL';
SELECT /*+ GATHER_PLAN_STATISTICS */ ... ;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR(NULL, NULL, 'ALLSTATS LAST'));
```

Compare **E-Rows** (estimated) and **A-Rows** (actual). Any operation off by 10× or more is a cardinality miss.

### Predicate Selectivity Test

Query the actual selectivity vs what CBO would estimate:

```sql
-- Actual
SELECT COUNT(*) FROM hr.customers WHERE state = 'CA' AND city = 'Los Angeles';
-- Say 500 out of 1,000,000 = 0.05%

-- CBO estimate (assume 100M rows, NDV state=50, NDV city=10000)
-- Density(state) = 1/50 = 0.02
-- Density(city)  = 1/10000 = 0.0001
-- Product        = 0.02 × 0.0001 = 0.000002
-- Estimated rows = 100M × 0.000002 = 200

-- Actual = 500, estimated = 200 — under by 2.5x. If actual were 500K, estimate would be 200 — 2500x off.
```

Column groups fix this.

## Diagnostic Queries

```sql
-- Column statistics for cardinality inputs
SELECT column_name, num_distinct, density, num_nulls,
       low_value, high_value, histogram
FROM   dba_tab_col_statistics
WHERE  owner = 'HR' AND table_name = 'CUSTOMERS';

-- Extended stats present
SELECT extension_name, extension, creator
FROM   dba_stat_extensions
WHERE  owner = 'HR' AND table_name = 'CUSTOMERS';

-- Was cardinality misestimated? (per operation)
-- Use DBMS_XPLAN.DISPLAY_CURSOR with ALLSTATS LAST format
```

## Common Fixes Summary

| Symptom                           | Fix                                        |
| --------------------------------- | ------------------------------------------ |
| Correlated predicates under-count | Extended stats (column group)              |
| Function on column returns 1%     | Expression stats or FBI                    |
| Out-of-range predicate returns 0  | Real-Time Stats or more frequent gathering |
| Skewed column no histogram        | `method_opt` with `SIZE 254`               |
| Join cardinality wrong            | Gather join column stats                   |
| Long IN list                      | Use GTT                                    |

## Best Practices

1. **Trust `AUTO_SAMPLE_SIZE`** for accuracy.
2. Column groups are cheap — add them for correlated predicates.
3. Expression stats for functions.
4. Real-Time Stats catches out-of-range gaps.
5. Monitor **E-Rows vs A-Rows** on production queries via SQL Monitor.
6. Use SQL Tuning Advisor recommendations for extended stats.
7. For hot lookup tables, gather stats **weekly** even without ETL.
8. Don't work around cardinality bugs with hints — fix stats.
9. `DBMS_STATS.SET_TABLE_PREFS` for per-table strategy.
10. For volatile transient tables, consider **NO_INVALIDATE=TRUE** to keep child cursor plans + `DBMS_STATS.SET_TABLE_PREFS('STALE_PERCENT', X)`.

## Interview Questions

1. **Q:** What is cardinality?
   **A:** CBO's estimate of rows produced by each plan step.

2. **Q:** Why does correlated predicate mislead CBO?
   **A:** CBO assumes independence; correlated columns share a lot of rows.

3. **Q:** Column group?
   **A:** Extended statistic capturing NDV of a combination of columns — fixes correlated predicate misestimates.

4. **Q:** Expression statistics?
   **A:** Extended statistic on `UPPER(name)` or similar — allows CBO to correctly estimate `WHERE UPPER(name)='X'`.

5. **Q:** Out-of-range predicate estimation?
   **A:** Returns near-zero if predicate exceeds HIGH_VALUE. Real-Time Stats or frequent gathering mitigates.

6. **Q:** How to detect cardinality miss?
   **A:** Compare E-Rows vs A-Rows via `DBMS_XPLAN.DISPLAY_CURSOR` with `ALLSTATS LAST`.

## References

- Oracle Database SQL Tuning Guide 19c — Optimizer Estimator
- MOS Doc ID 149560.1 — DBMS_STATS
- MOS Doc ID 754639.1 — Cardinality Feedback
- Jonathan Lewis, _Cost-Based Oracle Fundamentals_
