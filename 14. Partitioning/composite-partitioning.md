# Composite Partitioning

## Overview

**Composite partitioning** combines two partitioning schemes: a **primary partitioning** (range, list, or hash) plus a **sub-partitioning** (range, list, or hash). Each partition is itself divided into sub-partitions.

The classic pattern: **range by date** (for ILM) **sub-hash by customer_id** (for parallel-friendly access).

## Combinations

Since 11g, Oracle supports all nine combinations:

- Range-Hash
- Range-List
- Range-Range
- List-Range
- List-Hash
- List-List
- Hash-Hash
- Hash-Range
- Hash-List

Practical use is usually **Range-Hash** (time + spread) or **Range-List** (time + geography).

## Range-Hash Example

```sql
CREATE TABLE hr.orders (
  order_id    NUMBER,
  customer_id NUMBER NOT NULL,
  order_date  DATE NOT NULL,
  amount      NUMBER
)
PARTITION BY RANGE (order_date)
SUBPARTITION BY HASH (customer_id)
SUBPARTITIONS 8
(
  PARTITION p_2024 VALUES LESS THAN (DATE '2025-01-01') TABLESPACE ts_2024,
  PARTITION p_2025 VALUES LESS THAN (DATE '2026-01-01') TABLESPACE ts_2025,
  PARTITION p_max  VALUES LESS THAN (MAXVALUE) TABLESPACE ts_default
);
```

Each year (partition) has 8 hash sub-partitions on `customer_id`. Total: 3 partitions × 8 sub-partitions = 24 segments.

Queries:

- `WHERE order_date >= DATE '2025-01-01'` — prunes to p_2025 + p_max (all 16 sub-partitions).
- `WHERE order_date >= DATE '2025-01-01' AND customer_id = 42` — prunes to 2 sub-partitions.

## Range-List Example

```sql
CREATE TABLE hr.sales (
  sale_date DATE, country VARCHAR2(2), amount NUMBER
)
PARTITION BY RANGE (sale_date)
SUBPARTITION BY LIST (country)
SUBPARTITION TEMPLATE (
  SUBPARTITION sp_us   VALUES ('US','CA','MX'),
  SUBPARTITION sp_emea VALUES ('GB','DE','FR'),
  SUBPARTITION sp_apac VALUES ('JP','CN','IN'),
  SUBPARTITION sp_other VALUES (DEFAULT)
)
(
  PARTITION p_2024 VALUES LESS THAN (DATE '2025-01-01'),
  PARTITION p_2025 VALUES LESS THAN (DATE '2026-01-01')
);
```

**SUBPARTITION TEMPLATE** avoids repeating sub-partition definitions per partition.

## Interval-Hash (Composite with Interval)

```sql
CREATE TABLE hr.events (
  event_time TIMESTAMP, user_id NUMBER, payload CLOB
)
PARTITION BY RANGE (event_time)
INTERVAL (NUMTOYMINTERVAL(1, 'MONTH'))
SUBPARTITION BY HASH (user_id)
SUBPARTITIONS 16
(
  PARTITION p_initial VALUES LESS THAN (TIMESTAMP '2024-01-01 00:00:00')
);
```

New monthly partitions auto-created, each with 16 hash sub-partitions.

## Sub-partition Operations

Operate on sub-partitions individually:

```sql
-- Drop a sub-partition
ALTER TABLE hr.orders DROP SUBPARTITION p_2024_sp_us;

-- Move
ALTER TABLE hr.orders MOVE SUBPARTITION p_2025_sp_apac TABLESPACE ts_apac;

-- Truncate
ALTER TABLE hr.orders TRUNCATE SUBPARTITION p_2024_sp_other;
```

## Partition-Wise Joins on Composite

If two tables share the same range-hash structure, partition-wise joins work at the sub-partition level.

## Diagnostic Queries

```sql
-- Partition + sub-partition tree
SELECT partition_name, subpartition_name, tablespace_name, num_rows
FROM   user_tab_subpartitions
WHERE  table_name = 'ORDERS'
ORDER  BY partition_name, subpartition_position;

-- Sub-partition sizes
SELECT partition_name, subpartition_name,
       ROUND(bytes/1024/1024, 1) AS mb
FROM   user_segments
WHERE  segment_name = 'ORDERS' AND segment_type = 'TABLE SUBPARTITION'
ORDER  BY partition_name, subpartition_name;
```

## Common Issues

- **Sub-partition proliferation** — Interval + 16 sub-partitions = 192/year. Manage carefully.
- **Skew across sub-partitions** — Same rules as pure hash.
- **DDL complexity** — Sub-partition operations are more verbose.

## Best Practices

1. **Range-Hash** for time-series + join key spread.
2. **Range-List** for time-series + geographic ILM.
3. Use **SUBPARTITION TEMPLATE** for consistency.
4. Power-of-2 hash sub-partition count.
5. Match sub-partition scheme to biggest join / access pattern.
6. Monitor segment count — hundreds of segments can slow catalog operations.
7. Local indexes should follow the composite structure.

## Interview Questions

1. **Q:** What is composite partitioning?
   **A:** Primary partitioning (range/list/hash) plus sub-partitioning of each primary partition.

2. **Q:** Most common combo?
   **A:** Range-Hash — range by date for ILM, hash by ID for spread.

3. **Q:** SUBPARTITION TEMPLATE?
   **A:** Defines sub-partition structure once; applied to every parent partition.

4. **Q:** Can you drop a single sub-partition?
   **A:** Yes: `ALTER TABLE ... DROP SUBPARTITION <name>`.

5. **Q:** Partition-wise join at composite level?
   **A:** Both tables share the same composite structure — PX joins matching sub-partitions in parallel.

## References

- Oracle Database VLDB and Partitioning Guide 19c
- MOS Doc ID 268373.1 — Partitioning Overview
- MOS Doc ID 1078666.1 — Partition-Wise Joins
