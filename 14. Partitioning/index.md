# Partitioning

**Partitioning** divides a large table or index into smaller physical pieces (partitions) that appear as one logical object to SQL. It's the single biggest lever for scaling data warehouses and large OLTP tables: partition pruning eliminates I/O, partition-wise joins reduce PX overhead, and partition-level maintenance (drop, exchange, split) turns "hours of ETL" into "seconds of DDL."

Partitioning is a licensed **Enterprise Edition option**. Always confirm licensing before use.

## Contents

| Page                                                | Purpose                                     |
| --------------------------------------------------- | ------------------------------------------- |
| [Range Partitioning](range-partitioning.md)         | By value range — dates most common          |
| [List Partitioning](list-partitioning.md)           | By discrete values — country, region        |
| [Hash Partitioning](hash-partitioning.md)           | By hash — even distribution for concurrency |
| [Composite Partitioning](composite-partitioning.md) | Range+Hash, Range+List, etc.                |

## Related Features

- **Interval partitioning** — Auto-create partitions when new values arrive (Range extension).
- **Reference partitioning** — Child inherits parent's partitioning key by FK.
- **System partitioning** — Application-controlled partition assignment.
- **Virtual column partitioning** — Partition by a computed expression.
- **Hybrid partitioned tables** (19c) — Mix internal and external partitions.

## Related Sections

- [Storage / Segments](../04-storage/segments.md)
- [Performance Tuning / Parallel Execution](../12-performance-tuning/parallel-execution-tuning.md)
- [RMAN](../15-rman/index.md) — partition-level backup/restore.
