# Hash Partitioning

## Overview

**Hash partitioning** distributes rows across partitions using a hash function on the partition key. It's used when data doesn't have a natural range/list boundary but you want to:

- Spread I/O and contention across many partitions.
- Enable partition-wise joins between two tables partitioned on the same key.
- Reduce hot-block contention (right-most index leaf, sequence-heavy tables).

Unlike range/list, hash partitions have no "meaning" — they're just N buckets. Queries with `WHERE key = X` prune to the single partition holding X's hash.

## Syntax

```sql
CREATE TABLE hr.orders (
  order_id    NUMBER,
  customer_id NUMBER NOT NULL,
  order_date  DATE,
  amount      NUMBER
)
PARTITION BY HASH (customer_id)
PARTITIONS 16
STORE IN (ts_orders_1, ts_orders_2);
```

- **PARTITIONS N** — Oracle picks names (`SYS_P####`). Use a power of 2 for uniform distribution.
- **STORE IN** — round-robin tablespaces for partitions.

Explicit partition names:

```sql
CREATE TABLE hr.orders (...)
PARTITION BY HASH (customer_id)
(
  PARTITION p_h01 TABLESPACE ts_orders_1,
  PARTITION p_h02 TABLESPACE ts_orders_2,
  ...
);
```

## Partition Count

Always a **power of 2** (2, 4, 8, 16, ...). Ensures uniform hash distribution. Non-power values lead to skew.

## Pruning

Works only for **equality**:

```sql
-- Pruned to one partition
SELECT * FROM hr.orders WHERE customer_id = 42;
-- PARTITION HASH SINGLE

-- No pruning
SELECT * FROM hr.orders WHERE customer_id BETWEEN 1 AND 100;
-- PARTITION HASH ALL
```

Hash partitioning is not for range queries — use range or composite.

## Partition-Wise Joins

If two tables are hash-partitioned on the join key with the **same partition count**, Oracle can join partition-to-partition in parallel — dramatically reducing PX message overhead:

```sql
CREATE TABLE hr.orders (...) PARTITION BY HASH (customer_id) PARTITIONS 16;
CREATE TABLE hr.customers (...) PARTITION BY HASH (id) PARTITIONS 16;

-- Full partition-wise join
SELECT /*+ PARALLEL(8) */ *
FROM   hr.orders o JOIN hr.customers c ON o.customer_id = c.id;
```

## Common Operations

- **Add partition** — must re-hash existing data:
  ```sql
  ALTER TABLE hr.orders ADD PARTITION p_new;
  -- Data movement happens; slow on large tables
  ```
- **Coalesce partition** — combines two:
  ```sql
  ALTER TABLE hr.orders COALESCE PARTITION;
  ```

Not commonly used at runtime — pick your partition count carefully at creation.

## Diagnostic Queries

```sql
-- Row distribution across partitions
SELECT partition_name, num_rows
FROM   user_tab_partitions
WHERE  table_name = 'ORDERS'
ORDER  BY partition_position;

-- Check for skew
SELECT MIN(num_rows), MAX(num_rows),
       AVG(num_rows), STDDEV(num_rows)
FROM   user_tab_partitions
WHERE  table_name = 'ORDERS';
```

## Common Issues

- **Skew** — Partitioning key has low NDV (e.g., 100 customers, 16 partitions → some partitions hold many customers).
- **Non-power-of-2 partition count** — Uneven distribution.
- **Range queries scan all partitions** — Add local index on the range column.
- **Adding partitions expensive** — Redistribution.

## Best Practices

1. **Power of 2 partition count.**
2. Pick partition key with **high cardinality**.
3. Use for **equality-only** access patterns.
4. Enable **partition-wise joins** by hash-partitioning both sides of hot joins on the same key with the same partition count.
5. For OLTP with hot inserts, hash partition can spread sequence-driven inserts if you hash on the sequence-generated PK.
6. Combine with range in **composite (range-hash)** for time-series with join-key spread.

## Interview Questions

1. **Q:** When hash vs range?
   **A:** Hash: no natural range; want even distribution / concurrency spread. Range: time-series or ordered data.

2. **Q:** Why power of 2?
   **A:** Uniform hash distribution across partitions.

3. **Q:** Partition-wise join?
   **A:** Both tables hash-partitioned on the join key with same partition count → PX joins partition-to-partition, no data movement.

4. **Q:** Pruning for hash?
   **A:** Equality only.

5. **Q:** Adding a partition later?
   **A:** Requires row redistribution — costly.

## References

- Oracle Database VLDB and Partitioning Guide 19c
- MOS Doc ID 1078666.1 — Partition-Wise Joins
