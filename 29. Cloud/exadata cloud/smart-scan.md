# Exadata Smart Scan

## Overview

**Smart Scan** is Exadata's killer feature: **SQL predicate and projection push-down to storage cells**. Instead of the DB reading blocks from cells and filtering, cells run the filter and return only the matching rows.

Works only when the query is doing a **direct-path read** and hitting Exadata storage.

## Example

```sql
SELECT customer_id FROM sales WHERE amount > 100000 AND region = 'US';
```

Without Smart Scan (commodity storage):

- Cells return all blocks.
- Compute reads blocks, filters rows, returns `customer_id`.

With Smart Scan:

- Cells apply `amount > 100000 AND region = 'US'` per block.
- Cells extract only `customer_id`.
- Compute receives filtered rows only.

Result: 10–100× less data over the interconnect, 10–100× less CPU on compute.

## When Smart Scan Activates

Requirements:

1. **Direct-path read** — bypassing buffer cache. Usually triggered by:
   - Table scan on segment > `_small_table_threshold`.
   - Parallel query.
   - `/*+ FULL(t) */` hint.
2. **Segment on Exadata storage** — obvious on ExaCS, always true.
3. **Predicate can be offloaded** — not all can (see below).

## What Can Be Offloaded

**Yes**:

- Filter predicates on scalar types (INT, VARCHAR2, DATE, NUMBER, etc.).
- Bloom filters (join filtering).
- Simple aggregations (SUM, COUNT, MAX, MIN).
- Column projection (only requested columns).
- Function-based expressions on Oracle-native functions.
- HCC decompression.

**No**:

- PL/SQL user-defined functions (unless `PRAGMA UDF` allowed).
- Some XML / geospatial operations.
- LOB access.
- CLOB / BLOB predicates.

## Verifying Smart Scan Happened

```sql
-- Session stats
SELECT   name, value
FROM     v$mystat s JOIN v$statname n ON s.statistic# = n.statistic#
WHERE    name LIKE 'cell%' OR name LIKE '%physical read%'
ORDER BY name;

-- Key stats:
-- 'cell physical IO interconnect bytes' - what came back after Smart Scan
-- 'cell physical IO bytes eligible for predicate offload' - what could've offloaded
-- 'cell physical IO bytes saved by storage index' - Storage Index skips
```

Ratio: `interconnect bytes` / `eligible bytes` — smaller is better. 5% means 95% saved.

## Storage Indexes

Automatic. For each 1 MB storage region per column, cells maintain min/max metadata. On a scan, cells skip regions where the predicate can't match.

Example: `WHERE order_date > SYSDATE - 7` — cells skip 90% of the table if data is time-clustered.

Verify:

```sql
SELECT   name, value FROM v$mystat s JOIN v$statname n ON s.statistic# = n.statistic#
WHERE    name = 'cell physical IO bytes saved by storage index';
```

## HCC + Smart Scan

Cells decompress HCC data on read. Result: HCC-compressed tables scan faster than uncompressed on Exadata (less I/O, decompression on cell not compute).

## Common Anti-Patterns

- **Buffer cache in the way**: table is small, buffer cache reads dominate — no Smart Scan. Force with `/*+ FULL(t) */` and `/*+ PARALLEL(4) */` if needed.
- **Function on indexed column blocks offload**: `WHERE UPPER(name) = 'X'` — offload works but Storage Index doesn't. Consider function-based indexes.
- **Fetch-based cursor**: Cursor loop over lots of rows — smart scan happens on the underlying scan, but overhead dominates.

## Explain Plan Hints

Look for `STORAGE (SMART SCAN)` or `TABLE ACCESS STORAGE FULL FIRST ROWS` in plans:

```sql
EXPLAIN PLAN FOR
SELECT COUNT(*) FROM sales WHERE amount > 100000;

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY(FORMAT => 'TYPICAL +ADVANCED'));
```

## Sizing Implications

Smart Scan means:

- Compute can be smaller than commodity (less CPU spent filtering).
- Interconnect matters less (less traffic).
- Cell CPU matters more.

Exadata's shape sizing accounts for this balance.

## Related

- [Overview](overview.md).
- [Storage Cells](storage-cells.md).
- [Monitoring](monitoring.md).
