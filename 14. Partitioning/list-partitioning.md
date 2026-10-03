# List Partitioning

## Overview

**List partitioning** groups rows by explicit discrete values of the partition key. Ideal when the partition key has a small, well-defined set of values — country codes, region names, product categories.

## Syntax

```sql
CREATE TABLE hr.customers (
  customer_id NUMBER,
  name        VARCHAR2(100),
  country     VARCHAR2(2) NOT NULL,
  region      VARCHAR2(20)
)
PARTITION BY LIST (country)
(
  PARTITION p_americas VALUES ('US','CA','MX','BR','AR') TABLESPACE ts_amer,
  PARTITION p_emea     VALUES ('GB','DE','FR','IT','ES','NL') TABLESPACE ts_emea,
  PARTITION p_apac     VALUES ('JP','CN','KR','IN','AU','SG') TABLESPACE ts_apac,
  PARTITION p_other    VALUES (DEFAULT) TABLESPACE ts_other
);
```

`DEFAULT` is the catch-all — mandatory unless you enforce a check constraint.

## Automatic List Partitioning (12.2+)

Like interval partitioning for LIST — new partitions materialize when new distinct values arrive:

```sql
CREATE TABLE hr.customers (
  customer_id NUMBER, name VARCHAR2(100), country VARCHAR2(2) NOT NULL
)
PARTITION BY LIST (country) AUTOMATIC
(
  PARTITION p_us VALUES ('US')
);
-- INSERT with country='DE' auto-creates a new partition
```

## Multi-Column List (11g+)

```sql
CREATE TABLE hr.sales (
  region VARCHAR2(20), sub_region VARCHAR2(20), amount NUMBER
)
PARTITION BY LIST (region, sub_region)
(
  PARTITION p_us_east VALUES (('US','EAST')),
  PARTITION p_us_west VALUES (('US','WEST')),
  PARTITION p_eu_north VALUES (('EU','NORTH')),
  PARTITION p_eu_south VALUES (('EU','SOUTH'))
);
```

## Partition Operations

Same DDL vocabulary as range: `ADD`, `DROP`, `MERGE`, `SPLIT`, `EXCHANGE`, `TRUNCATE`, `MOVE`, `RENAME`.

Key differences:

- **Add** — specify list values, not ranges.
- **Split** — split at specific values from the list.

```sql
-- Add a new list partition
ALTER TABLE hr.customers ADD PARTITION p_middle_east
  VALUES ('AE','SA','IL','TR') TABLESPACE ts_me;

-- Move country from p_other into new partition
ALTER TABLE hr.customers SPLIT PARTITION p_other
  VALUES ('CN') INTO
  (PARTITION p_china, PARTITION p_other);
```

## Diagnostic Queries

```sql
SELECT partition_name, high_value, tablespace_name, num_rows
FROM   user_tab_partitions
WHERE  table_name = 'CUSTOMERS';

-- View list values per partition
SELECT partition_name, high_value
FROM   user_tab_partitions
WHERE  table_name = 'CUSTOMERS';
```

## Common Issues

- **Falls into DEFAULT partition** — DEFAULT grows large; consider adding explicit partitions for high-volume values.
- **Cannot add value already in DEFAULT** — Split DEFAULT first.
- **Automatic list without explicit values** — Partition names are system-generated.

## Best Practices

1. Use list for **enumerable, low-cardinality** partition keys.
2. Include a **DEFAULT** partition (or use AUTOMATIC).
3. Watch skew — one region dominates all others? Consider composite (list-hash) partitioning.
4. Match partitioning to query filter patterns.
5. `UPDATE INDEXES` on partition operations.

## Interview Questions

1. **Q:** When list vs range?
   **A:** List: discrete enumerable values. Range: value ranges (dates, IDs).

2. **Q:** What is DEFAULT?
   **A:** Catch-all partition for values not matching any explicit list.

3. **Q:** Automatic list?
   **A:** 12.2+ feature: new distinct values trigger new partition creation.

4. **Q:** Can you list-partition on multiple columns?
   **A:** Yes (11g+) — VALUES are tuples.

## References

- Oracle Database VLDB and Partitioning Guide 19c
- MOS Doc ID 2091823.1 — Automatic List Partitioning
