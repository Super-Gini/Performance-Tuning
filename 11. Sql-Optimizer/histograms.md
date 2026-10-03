# Histograms

## Overview

A **histogram** describes the **distribution** of values in a column. Without a histogram, CBO assumes uniform distribution — every value equally common — and computes cardinality as `NumRows / NumDistinctValues`. For **skewed** columns, this is badly wrong. Histograms fix this by recording how many rows have each value (or value range).

Skew comes in patterns like: 90% of orders are `STATUS = 'COMPLETE'`, 5% `PENDING`, 5% `SHIPPED`. A query `WHERE status = 'PENDING'` should estimate ~5% rows; without a histogram, it estimates 33%.

## Types

Oracle 12c+ has four histogram types:

| Type                | When Used                                                                              |
| ------------------- | -------------------------------------------------------------------------------------- |
| **Frequency**       | NDV ≤ 254; one bucket per distinct value                                               |
| **Top-Frequency**   | Highly skewed with a few dominant values (12c+)                                        |
| **Height-Balanced** | Legacy — NDV > 254 with older algorithm                                                |
| **Hybrid**          | 12c+ replacement for height-balanced; combines features of frequency + height-balanced |

Modern default (`AUTO_SAMPLE_SIZE` + `SIZE AUTO`) picks the right type automatically.

## Creation

### Auto (recommended)

```sql
EXEC DBMS_STATS.GATHER_TABLE_STATS('HR','ORDERS',
  method_opt=>'FOR ALL COLUMNS SIZE AUTO');
```

`SIZE AUTO` means: create histogram if the column is used in a WHERE / GROUP BY clause **and** has skew (based on column usage tracking in `SYS.COL_USAGE$`).

### Manual for specific column

```sql
EXEC DBMS_STATS.GATHER_TABLE_STATS('HR','ORDERS',
  method_opt=>'FOR COLUMNS STATUS SIZE 254');
```

`SIZE N` where N = max bucket count. 254 is the max for frequency-like histograms.

### Delete a histogram (if it's causing plan flip / bind peeking issues)

```sql
EXEC DBMS_STATS.DELETE_COLUMN_STATS(
  ownname=>'HR', tabname=>'ORDERS', colname=>'STATUS',
  col_stat_type=>'HISTOGRAM');
```

## Bind Peeking Interaction

CBO peeks at bind values on the **first parse** and generates a plan optimized for those values. If subsequent executions use different bind values that map to very different histogram buckets, the cached plan is bad — this is the **bind peek problem**.

**Adaptive Cursor Sharing (11g+)** partially mitigates: CBO notices execution stats varying, marks the cursor as bind-sensitive, and produces child cursors per bind pattern. See [Adaptive Cursor Sharing](../36-sql-tuning/adaptive-cursor-sharing.md).

## Choosing Histogram Size

- **Frequency**: use if NDV ≤ 254 and skew matters.
- **Top-Frequency**: skewed with a small number of popular values; captures those + one "other" bucket.
- **Hybrid**: high NDV columns with tail distribution.

`SIZE 254` covers most cases. Small sizes (`SIZE 10`) exist for legacy compatibility — rarely needed.

## Diagnostic Queries

```sql
-- Histograms present
SELECT table_name, column_name, histogram, num_buckets, sample_size,
       last_analyzed
FROM   dba_tab_col_statistics
WHERE  owner = 'HR' AND histogram <> 'NONE';

-- Histogram bucket detail
SELECT endpoint_number, endpoint_value, endpoint_actual_value
FROM   dba_tab_histograms
WHERE  owner = 'HR' AND table_name = 'ORDERS' AND column_name = 'STATUS'
ORDER  BY endpoint_number;

-- Column usage tracking (which columns SIZE AUTO decides to histogram)
SELECT o.owner, o.object_name, c.name AS column_name,
       cu.equality_preds, cu.equijoin_preds, cu.nonequijoin_preds,
       cu.range_preds, cu.like_preds, cu.null_preds
FROM   sys.col_usage$ cu
JOIN   dba_objects o ON o.object_id = cu.obj#
JOIN   sys.col$ c ON c.obj# = cu.obj# AND c.col# = cu.intcol#
WHERE  o.owner = 'HR' AND o.object_name = 'ORDERS';
```

## Common Issues

- **Bind peek flip** — Same SQL, wildly different runtime depending on first bind. Enable ACS (default 11g+).
- **Histogram missing** — `SIZE AUTO` didn't detect column usage. Force with explicit `SIZE 254`.
- **Wrong histogram type** — Height-balanced on 12c+ suggests you're using ancient stats. Regather.
- **Histogram invalidating literal SQL** — Extreme case: histogram + literals cause thousands of child cursors. Fix with bind variables.

## Best Practices

1. Let **SIZE AUTO** do its job most of the time.
2. If skew matters and query performance depends on it, gather histogram explicitly for that column.
3. Use bind variables — histograms + literals is a bad combination.
4. Combine histograms with **adaptive cursor sharing** for bind-sensitive plans.
5. Delete histograms rarely — usually you want them.
6. Monitor `V$SQL_SHARED_CURSOR.BIND_SENSITIVE` and `BIND_AWARE`.

## Interview Questions

1. **Q:** Why a histogram?
   **A:** For skewed data, CBO's uniform-distribution assumption is wrong. Histograms give the true distribution.

2. **Q:** Histogram types?
   **A:** Frequency, top-frequency, height-balanced (legacy), hybrid.

3. **Q:** When is a frequency histogram used?
   **A:** NDV ≤ 254 and `SIZE ≥ NDV`.

4. **Q:** Bind peek issue?
   **A:** CBO plans for first bind values; wildly different subsequent binds hit the wrong plan.

5. **Q:** Fix?
   **A:** Adaptive Cursor Sharing (auto in 11g+) + reasonable histogram sizes.

## References

- Oracle Database SQL Tuning Guide 19c — Histograms
- MOS Doc ID 62151.1 — Histograms Overview
- MOS Doc ID 1088918.1 — Hybrid Histograms
