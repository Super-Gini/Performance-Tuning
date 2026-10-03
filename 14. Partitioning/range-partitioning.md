# Range Partitioning

## Overview

**Range partitioning** divides a table by ranges of values on a **partition key**. Date-based partitioning is the canonical case: one partition per month or day. Range partitioning enables **partition pruning** (queries with `WHERE order_date >= ...` scan only relevant partitions) and **partition drop / exchange** for fast ILM operations.

## Syntax

```sql
CREATE TABLE hr.orders (
  order_id       NUMBER,
  order_date     DATE NOT NULL,
  customer_id    NUMBER,
  amount         NUMBER,
  status         VARCHAR2(20)
)
PARTITION BY RANGE (order_date)
(
  PARTITION p_2024_q1 VALUES LESS THAN (DATE '2024-04-01') TABLESPACE ts_2024,
  PARTITION p_2024_q2 VALUES LESS THAN (DATE '2024-07-01') TABLESPACE ts_2024,
  PARTITION p_2024_q3 VALUES LESS THAN (DATE '2024-10-01') TABLESPACE ts_2024,
  PARTITION p_2024_q4 VALUES LESS THAN (DATE '2025-01-01') TABLESPACE ts_2024,
  PARTITION p_2025_q1 VALUES LESS THAN (DATE '2025-04-01') TABLESPACE ts_2025,
  PARTITION p_max     VALUES LESS THAN (MAXVALUE) TABLESPACE ts_default
);
```

`MAXVALUE` catches everything above the last explicit range.

## Interval Partitioning — Auto-Create

For time-series data, avoid managing every partition manually:

```sql
CREATE TABLE hr.events (
  event_id NUMBER, event_time TIMESTAMP, payload CLOB
)
PARTITION BY RANGE (event_time)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
(
  PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00')
);
```

New partitions materialize automatically when data outside existing ranges arrives. Names are system-generated (`SYS_P####`). Rename after creation:

```sql
BEGIN
  FOR p IN (SELECT partition_name, high_value
            FROM user_tab_partitions
            WHERE table_name = 'EVENTS' AND partition_name LIKE 'SYS_P%') LOOP
    EXECUTE IMMEDIATE 'ALTER TABLE hr.events RENAME PARTITION ' ||
      p.partition_name || ' TO p_' || TO_CHAR(SYSDATE, 'YYYYMM');
  END LOOP;
END;
/
```

## Partition Pruning

CBO detects predicates on the partition key and eliminates unqualified partitions:

```sql
EXPLAIN PLAN FOR
SELECT SUM(amount) FROM hr.orders
WHERE  order_date >= DATE '2025-01-01';

-- Plan shows PARTITION RANGE ITERATOR — only 2025+ partitions accessed
```

Static pruning (literals): CBO knows at parse time.
Dynamic pruning (binds): decided at execution.

## Partition Operations

### Drop old partition — instant

```sql
ALTER TABLE hr.orders DROP PARTITION p_2020_q1 UPDATE INDEXES;
```

Compare to `DELETE FROM orders WHERE order_date < DATE '2020-04-01'` — hours of I/O.

### Add partition

```sql
ALTER TABLE hr.orders ADD PARTITION p_2026_q1
  VALUES LESS THAN (DATE '2026-04-01') TABLESPACE ts_2026;
```

### Split MAXVALUE partition

```sql
ALTER TABLE hr.orders SPLIT PARTITION p_max
  AT (DATE '2026-04-01') INTO
  (PARTITION p_2026_q1 TABLESPACE ts_2026,
   PARTITION p_max TABLESPACE ts_default) UPDATE INDEXES;
```

### Exchange partition — fast bulk load

Load into a plain table, then swap:

```sql
CREATE TABLE orders_stage AS SELECT * FROM hr.orders WHERE 1=0;

-- Load data into stage (bulk INSERT, direct-path)
INSERT /*+ APPEND */ INTO orders_stage SELECT * FROM external_data;

-- Swap
ALTER TABLE hr.orders EXCHANGE PARTITION p_2025_q4
  WITH TABLE orders_stage INCLUDING INDEXES WITHOUT VALIDATION;
```

`WITHOUT VALIDATION` skips FK/PK checks — safe only when you know data is clean.

### Truncate partition

```sql
ALTER TABLE hr.orders TRUNCATE PARTITION p_2020_q1 UPDATE INDEXES;
```

## Local vs Global Indexes

- **Local index**: one index partition per table partition. Preferred — drop/exchange partition doesn't invalidate.
- **Global index**: single index across all partitions. Faster for range scans that span partitions but breaks on partition operations unless `UPDATE INDEXES` clause used.

```sql
CREATE INDEX idx_orders_customer ON hr.orders (customer_id) LOCAL;
CREATE INDEX idx_orders_status ON hr.orders (status) GLOBAL;
```

## Diagnostic Queries

```sql
-- Partition list
SELECT partition_name, high_value, tablespace_name, num_rows
FROM   user_tab_partitions
WHERE  table_name = 'ORDERS'
ORDER  BY partition_position;

-- Partition sizes
SELECT partition_name, ROUND(bytes/1024/1024/1024, 2) AS gb
FROM   user_segments
WHERE  segment_name = 'ORDERS' AND segment_type = 'TABLE PARTITION'
ORDER  BY partition_name;

-- Was partition pruning used?
-- Look for PARTITION RANGE ITERATOR / SINGLE in DBMS_XPLAN
```

## Common Issues

- **No pruning** — Predicate uses function on partition key (`WHERE TRUNC(order_date) = ...`). Fix: predicate should be `order_date >= X AND < Y`.
- **Skewed partitions** — Interval by month, but data 90% in current month. Consider daily interval.
- **Global index invalidation** — Missing `UPDATE INDEXES` on partition ops. Now index is UNUSABLE.
- **Interval partition names auto-generated** — Rename script needed for readability.

## Best Practices

1. **Interval partitioning** for time-series data.
2. Match partition interval to query patterns (queries filter by day → daily partitions).
3. Local indexes for OLTP; global indexes only where necessary.
4. `UPDATE INDEXES` on all partition operations.
5. Partition-align tablespaces for ILM (old partitions → read-only tablespace on cheap storage).
6. Use exchange partition for bulk loads.
7. Never `DELETE` — always `DROP PARTITION` or `TRUNCATE PARTITION`.
8. Test partition pruning after query changes.
9. Include partition key in query predicates.
10. Monitor `user_tab_partitions.num_rows` distribution — skew indicates bad interval choice.

## Interview Questions

1. **Q:** What is range partitioning?
   **A:** Table split by value ranges on a partition key.

2. **Q:** Interval vs manual range?
   **A:** Interval auto-creates partitions when data outside existing ranges arrives.

3. **Q:** Partition pruning?
   **A:** CBO uses predicates on partition key to skip irrelevant partitions.

4. **Q:** Local vs global index?
   **A:** Local: one per partition, aligned. Global: spans all partitions. Partition ops break global unless `UPDATE INDEXES`.

5. **Q:** Exchange partition?
   **A:** Swap a stage table with a partition — near-instant data load.

6. **Q:** Why prefer `DROP PARTITION` over `DELETE`?
   **A:** DROP is a DDL — nearly instant. DELETE is full-scan DML — hours on large tables.

## References

- Oracle Database VLDB and Partitioning Guide 19c
- MOS Doc ID 268373.1 — Partitioning Overview
- MOS Doc ID 1462003.1 — Interval Partitioning
