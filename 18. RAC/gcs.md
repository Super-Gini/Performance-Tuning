# GCS — Global Cache Services

## Overview

**Global Cache Services (GCS)** is the RAC subsystem that coordinates data-block access across instances. Every buffer cache block in the cluster has a **master** instance — the authoritative record of who owns the block, in which mode. When any instance wants to read or modify a block, it consults the master (usually via the LMD/LMS processes) and coordinates via [Cache Fusion](cache-fusion.md).

## Resource Types

GCS manages a specific enqueue: the **BL (Block) enqueue**. Combined with GES enqueues for other resources, this forms **Global Resource Directory (GRD)**.

## Modes

- **NULL (N)** — no interest.
- **SHARED (S)** — reader.
- **EXCLUSIVE (X)** — writer.

Transitions coordinated by GCS. Multiple readers OK simultaneously; single writer excludes others.

## Master Selection

Block mastering is hashed: given a `DBA`, the same instance always masters it (avoiding lookup churn). Some blocks may be re-mastered based on affinity heuristics — a segment accessed almost exclusively by one instance may have its blocks re-mastered to that instance.

## Diagnostic Queries

```sql
-- GCS statistics
SELECT name, value FROM v$sysstat WHERE name LIKE 'gcs%';

-- GC event summary
SELECT event, total_waits,
       ROUND(time_waited_micro/1e6, 1) AS total_sec,
       ROUND(time_waited_micro/DECODE(total_waits,0,1,total_waits)/1000, 2) AS avg_ms
FROM   v$system_event
WHERE  wait_class = 'Cluster'
ORDER  BY total_sec DESC;

-- Interconnect metrics
SELECT if_name, tx_bytes, rx_bytes
FROM   v$cluster_interconnects;

-- Block servers (LMS)
SELECT * FROM v$gcs_server_stats;
```

## Common Issues

- **`gc cr block 2-way` avg > 5 ms** — Interconnect latency; upgrade NICs or reduce load.
- **`gc buffer busy acquire`** — Hot block; multiple instances contending.
- **`gc current grant congested`** — GCS overloaded; check LMS activity.
- **Re-mastering thrashing** — Random access from all nodes; consider service partitioning.

## Best Practices

1. Fast interconnect (10G+, low-latency switches).
2. Service-based partitioning — cluster affinity.
3. Adequate `gcs_server_processes`.
4. Monitor `gc` events in AWR.
5. Avoid application patterns causing cross-node hot blocks.

## Interview Questions

1. **Q:** What is GCS?
   **A:** Global Cache Services — coordinates block access across RAC instances.

2. **Q:** BL enqueue?
   **A:** Block enqueue used by GCS to serialize block access.

3. **Q:** Master?
   **A:** The instance authoritative for coordinating a specific block's state.

4. **Q:** Re-mastering?
   **A:** Moving block mastering to the instance most frequently accessing it — affinity optimization.

## References

- Oracle Real Application Clusters Administration 19c
- MOS Doc ID 730079.1 — Cache Fusion / GCS
