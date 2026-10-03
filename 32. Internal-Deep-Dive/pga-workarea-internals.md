# PGA Workarea Internals

## Overview

The **PGA workarea** is a session-private region used for sort, hash join, bitmap, and hash-group operations that can't fit in a small in-memory buffer. Oracle's workarea manager makes real-time decisions about whether each operation runs **optimal** (fully in memory), **one-pass** (spills once to TEMP), or **multi-pass** (multiple TEMP round trips). This decision — driven by the `PGA_AGGREGATE_TARGET` policy — determines whether a query completes in seconds or hours.

This page covers PGA memory management, workarea sizing math, one-pass vs multi-pass modes, and diagnostics.

## PGA vs SGA

- **SGA** — shared, allocated at startup, one per instance.
- **PGA** — per-server-process, allocated on demand, sum across all sessions bounded by `PGA_AGGREGATE_TARGET`.

Each dedicated server process has:

- **UGA** (User Global Area) — session-specific: variables, session state, package state.
- **Sort area / hash area / bitmap area** — workareas, dynamically sized.
- **Cursor state** — active cursor storage.

## Workarea Sizing — Automatic PGA

Introduced 9i. Instead of manual `SORT_AREA_SIZE` per session, DBA sets `PGA_AGGREGATE_TARGET` (soft target) and (12c+) `PGA_AGGREGATE_LIMIT` (hard cap).

Oracle's workarea policy (`WORKAREA_SIZE_POLICY = AUTO`):

- Track total PGA usage cluster-wide.
- Assign each workarea a size based on demand:
  - Big enough to run **optimal** if room permits.
  - Downgrade to **one-pass** if too many big workareas concurrent.
  - Downgrade to **multi-pass** under severe pressure.

Manual (`WORKAREA_SIZE_POLICY = MANUAL`) — legacy — uses per-session `SORT_AREA_SIZE`, `HASH_AREA_SIZE`. Rarely used.

## Workarea Modes

For a hash join with N-row build:

| Mode           | Behavior                                                                                         | Performance       |
| -------------- | ------------------------------------------------------------------------------------------------ | ----------------- |
| **Optimal**    | Build hash table fits entirely in PGA. Probe streams through.                                    | Fastest. No TEMP. |
| **One-pass**   | Build hash table too big → partition & spill to TEMP. Probe streams. Merge partitions from TEMP. | 2× reads.         |
| **Multi-pass** | Even the partitions too big → recursive spill. Many TEMP round trips.                            | Very slow.        |

For a sort:

- Optimal — entire sort in memory.
- One-pass — sort runs of memory-sized batches to TEMP, then merge.
- Multi-pass — merge runs are themselves too big; iterate.

## Sizing Math (Simplified)

For each workarea:

- **Expected size** = row count × row width × 1.2 (safety).
- **Optimal target** = size that avoids TEMP.
- **One-pass target** ≈ `sqrt(expected_size)` × row width.

Oracle picks the largest of {allowed by policy, workarea's target}.

Example — hash join with 100M rows × 200 bytes = 20 GB build:

- Optimal → needs 20 GB PGA workarea.
- One-pass → needs `sqrt(20 GB × 200B) = sqrt(4 × 10^12)` ≈ 2 MB. Very small workarea, but many partitions each spilling to TEMP.
- Multi-pass → happens when policy squeezes below one-pass minimum.

## `V$SQL_WORKAREA` — Historical Workareas

Every workarea execution records:

```sql
SELECT   sql_id, operation_type, policy, operation_id,
         estimated_optimal_size/1024/1024 optimal_mb,
         estimated_onepass_size/1024/1024 onepass_mb,
         last_memory_used/1024/1024 last_used_mb,
         last_execution, last_tempseg_size/1024/1024 last_temp_mb
FROM     v$sql_workarea
WHERE    sql_id = '&sql_id'
ORDER BY operation_id;
```

Fields:

- `LAST_EXECUTION` = `OPTIMAL`, `ONE PASS`, `MULTI-PASS`.
- `LAST_TEMPSEG_SIZE` = bytes spilled to TEMP.

## `V$SQL_WORKAREA_ACTIVE` — Live Workareas

```sql
SELECT   sid, operation_type,
         work_area_size/1024/1024 alloc_mb,
         expected_size/1024/1024 expected_mb,
         actual_mem_used/1024/1024 used_mb,
         max_mem_used/1024/1024 max_used_mb,
         tempseg_size/1024/1024 temp_mb,
         number_passes
FROM     v$sql_workarea_active;
```

`NUMBER_PASSES = 0` — optimal. `= 1` — one-pass. `> 1` — multi-pass (bad).

## PGA Advisor

Simulates PGA sizing effects:

```sql
SELECT   pga_target_for_estimate/1024/1024/1024 target_gb,
         pga_target_factor,
         estd_extra_bytes_rw/1024/1024/1024 estd_extra_io_gb,
         ROUND(estd_pga_cache_hit_percentage,2) cache_hit_pct,
         estd_overalloc_count
FROM     v$pga_target_advice
ORDER BY pga_target_for_estimate;
```

`cache_hit_pct` should be > 95%. If < 90%, PGA is too small — more workareas going one-pass / multi-pass.

## `PGA_AGGREGATE_LIMIT` (12c+)

Hard cap on total PGA usage. Introduced because runaway workloads could allocate unbounded PGA under `PGA_AGGREGATE_TARGET` policy — target is soft.

When breached:

- New sessions get `ORA-04036: PGA memory used by the instance exceeds PGA_AGGREGATE_LIMIT`.
- Or sessions killed.

Recommended: `PGA_AGGREGATE_LIMIT = 2 × PGA_AGGREGATE_TARGET` or system RAM allowance.

```sql
SHOW PARAMETER pga_aggregate_limit
```

## Session PGA Statistics

```sql
SELECT   name, value/1024/1024 mb FROM v$pgastat
WHERE    unit = 'bytes'
ORDER BY value DESC;
```

Key metrics:

- **`aggregate PGA target parameter`** — the setting.
- **`aggregate PGA auto target`** — auto workarea share (target minus other PGA uses).
- **`total PGA allocated`** — current sum across all sessions.
- **`total PGA inuse`** — actively-used portion.
- **`total freeable PGA memory`** — allocated but unused.
- **`over allocation count`** — how many times target was exceeded.
- **`extra bytes read/written`** — TEMP IO from workareas that didn't fit.

## Top PGA Consumers

```sql
SELECT   s.sid, s.serial#, s.username, s.program,
         ROUND(p.pga_used_mem/1024/1024, 2) used_mb,
         ROUND(p.pga_alloc_mem/1024/1024, 2) alloc_mb,
         ROUND(p.pga_max_mem/1024/1024, 2) max_mb
FROM     v$process p JOIN v$session s ON s.paddr = p.addr
WHERE    s.type = 'USER'
ORDER BY p.pga_max_mem DESC
FETCH FIRST 20 ROWS ONLY;
```

## Sort vs Hash vs Bitmap

Workarea operation types:

| Type               | When used                                             | Size correlated with            |
| ------------------ | ----------------------------------------------------- | ------------------------------- |
| `SORT`             | ORDER BY, GROUP BY without hash aggregation, DISTINCT | Sort key + row width.           |
| `HASH-JOIN`        | Hash join build side.                                 | Smaller of the two join inputs. |
| `HASH-GROUP-BY`    | GROUP BY / DISTINCT via hash.                         | Number of distinct groups.      |
| `BUFFER`           | Buffered sort.                                        | Small.                          |
| `CONNECT-BY`       | Hierarchical query recursion.                         | Depth × row width.              |
| `BITMAP-CREATE`    | BITMAP index build.                                   | Row count.                      |
| `BITMAP-MERGE`     | Merging bitmap results.                               | Number of bitmaps.              |
| `BITMAP-CONSTRUCT` | Bitmap conversion.                                    | Row count.                      |

## Query-Level Workarea Tuning

Force manual mode per session for a specific query:

```sql
ALTER SESSION SET WORKAREA_SIZE_POLICY = MANUAL;
ALTER SESSION SET SORT_AREA_SIZE = 1024*1024*512;   -- 512 MB
ALTER SESSION SET HASH_AREA_SIZE = 1024*1024*1024;  -- 1 GB
-- Run the query.
ALTER SESSION SET WORKAREA_SIZE_POLICY = AUTO;
```

Rarely necessary in 19c — the auto policy is competent.

## TEMP Interaction

Workareas spilling to TEMP write **direct path writes** to a TEMP file. Later reads are **direct path read temp**.

Monitor:

```sql
SELECT   sid, event,
         p1 file#, p2 block#, p3 blocks,
         seconds_in_wait
FROM     v$session
WHERE    event IN ('direct path write temp','direct path read temp')
   AND   state = 'WAITING';
```

Excess `direct path write temp` = workareas spilling. If a specific SQL is the culprit, its plan often shows large `TEMP` in `A-Rows` / `A-Time` from `DBMS_XPLAN.DISPLAY_CURSOR(..., 'ALLSTATS LAST')`.

## Interview Framing

> "What's the difference between optimal, one-pass, and multi-pass?"

Workarea execution modes. Optimal = fully in memory. One-pass = data spills to TEMP once (build partitioned, probe streams, merge). Multi-pass = TEMP round trips because even partitions don't fit → very slow.

> "You see a query with high `direct path write temp`. What do you do?"

Check `V$SQL_WORKAREA` for that SQL — likely one-pass or multi-pass on a hash join or sort. Options: bigger PGA target, rewrite to reduce intermediate size (better predicate pushdown), or force manual with bigger session workarea for that one execution.

> "How do you diagnose PGA sizing?"

`V$PGA_TARGET_ADVICE` cache_hit_pct. `V$PGASTAT.over allocation count`. `V$SQL_WORKAREA.last_execution` distribution across sessions.

## Related

- [PGA](../03-instance-architecture/memory/pga.md).
- [Memory Parameters](../25-reference/initialization-parameters/memory-parameters.md).
- [Tempfiles](../04-storage/tempfiles.md).
- [User I/O Wait Events](../25-reference/wait-events/user-io.md).
- [ORA-01652](../26-errors/ora-01652.md).
- [Wait Event Framework](wait-event-framework.md).
