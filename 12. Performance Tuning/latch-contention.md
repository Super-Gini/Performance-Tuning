# Latch Contention

## Overview

**Latches** are low-level SGA serialization primitives. They protect shared memory data structures (buffer cache hash chains, redo log buffer, library cache metadata) for microseconds at a time. Contention on latches almost always means the application is doing something inefficient — hot blocks, hard parses, cursor churn.

## Common Latches and Root Causes

| Latch                               | Symptom                             | Root Cause                          | Fix                                                    |
| ----------------------------------- | ----------------------------------- | ----------------------------------- | ------------------------------------------------------ |
| **cache buffers chains**            | Hot block accessed by many sessions | Hot single block or hot hash bucket | Partition hot table; sequence cache; reverse-key index |
| **shared pool**                     | Shared pool free-list contention    | Hard-parse storm, literal SQL       | Bind variables; enlarge shared pool                    |
| **library cache** (10g)             | Cursor lookups                      | Same as above                       | Same                                                   |
| **redo allocation** / **redo copy** | Redo buffer allocation              | Very high redo generation           | Scalable LGWR; large private strands                   |
| **row cache objects**               | Dictionary cache                    | DDL/user creation storm             | Reduce DDL; investigate                                |
| **session allocation**              | UGA / session slot                  | Very high connect rate              | Connection pooling                                     |
| **enqueue hash chains**             | Enqueue table lookup                | Very high lock rate                 | Reduce transaction rate; reduce lock type variety      |

## Detection

### Top latches by sleeps

```sql
SELECT name,
       gets, misses, sleeps,
       ROUND(sleeps/DECODE(misses,0,1,misses)*100, 2) AS sleep_pct_of_miss,
       ROUND(misses/DECODE(gets,0,1,gets)*100, 4) AS miss_pct
FROM   v$latch
ORDER  BY sleeps DESC
FETCH FIRST 15 ROWS ONLY;
```

`sleeps > 0` = actual contention (as opposed to just misses, which retry via spin).

### Wait events

```sql
SELECT event, total_waits,
       ROUND(time_waited_micro/1e6, 1) AS total_sec,
       ROUND(time_waited_micro/DECODE(total_waits,0,1,total_waits)/1000, 2) AS avg_ms
FROM   v$system_event
WHERE  event LIKE 'latch:%'
ORDER  BY total_sec DESC;
```

Modern versions expose specific latches as `latch: <name>` wait events.

### Child latch analysis (for `cache buffers chains`)

```sql
SELECT addr, gets, misses, sleeps, immediate_gets, immediate_misses
FROM   v$latch_children
WHERE  name = 'cache buffers chains'
ORDER  BY sleeps DESC
FETCH FIRST 20 ROWS ONLY;
```

Convert `addr` to block address to find the hot block.

## `cache buffers chains` — Hot Block

Common hot-block sources:

- **Sequence-driven index leaf** — every insert hits the right-most leaf.
- **Right-hand root branches** — pk index on monotonically increasing keys.
- **Single-row lookup tables** — configuration tables read by every session.
- **Small tables in busy joins** — accessed by every worker.

### Fix Strategies

- **Reverse-key index** — for pure integer keys (loses range scans).
- **Hash-partitioned index** — spreads leaf blocks (partitioning license).
- **Sequence CACHE 1000+ NOORDER** — reduces coordination.
- **KEEP pool** — pin the block so it can't be evicted (still hot on latch, but caches consistent).
- **Refactor query** — avoid frequent lookups (application-level cache).

## `shared pool` — Hard-Parse Storm

Signs:

- High `latch: shared pool` and `library cache: mutex X`.
- High `V$SYSSTAT` "parse count (hard)" per second.
- `V$SQL.EXECUTIONS = 1` for millions of rows.

Root cause: literal SQL. Fix: `cursor_sharing=FORCE` (temporary; Oracle rewrites literals as binds) then fix application.

## `redo allocation` / `redo copy`

Very high redo generation on multi-CPU systems. Modern versions use **private redo strands** — each foreground has its own strand. Configured by `_log_private_strand_size` etc.; usually auto-tuned. Enable scalable LGWR.

## Diagnostic Queries

```sql
-- Where in code did the miss happen?
SELECT parent_name, location, gets, misses, sleeps
FROM   v$latch_misses
WHERE  sleeps > 0
ORDER  BY sleeps DESC
FETCH FIRST 20 ROWS ONLY;

-- Who is holding a latch right now?
SELECT h.pid, h.sid, l.name AS latch_name
FROM   v$latchholder h JOIN v$latch l ON l.addr = h.laddr;

-- Sessions currently waiting on a latch
SELECT sid, event, p1raw AS latch_addr, seconds_in_wait
FROM   v$session_wait
WHERE  event LIKE 'latch:%';
```

## Common Issues

- **`cache buffers chains` dominant in AWR** — Hot block.
- **`shared pool` latch waits** — Bind variable problem.
- **RAC — global cache buffer chains** — cross-instance hot block.
- **After add/drop DDL** — Row cache latches climb; investigate DDL storm.

## Best Practices

1. **Bind variables** — eliminates most `shared pool` contention.
2. **Sequence CACHE 1000+** — for high-throughput sequences.
3. **Hash-partitioned tables/indexes** for hot data.
4. **Application-level caching** for reference lookups.
5. Alert on any `latch:` event in AWR top 5.
6. `session_cached_cursors=200` reduces parse pressure.
7. Do not tune `_spin_count` without measured evidence.
8. In RAC, prefer service-based partitioning to keep hot data on one node.

## Interview Questions

1. **Q:** What is a latch?
   **A:** Lightweight SGA serialization primitive; microseconds hold; no queue.

2. **Q:** `latch: cache buffers chains` — root cause?
   **A:** Hot block — multiple sessions accessing same block through the same hash bucket.

3. **Q:** `latch: shared pool` — root cause?
   **A:** Hard-parse storm from literal SQL.

4. **Q:** How to fix hot right-most index leaf?
   **A:** Reverse-key index, hash-partitioned index, sequence cache increase.

5. **Q:** How to find hot child latch?
   **A:** `V$LATCH_CHILDREN` ORDER BY sleeps DESC.

## References

- Oracle Database Performance Tuning Guide 19c — Latch and Mutex Contention
- MOS Doc ID 62143.1 — Diagnosing Latch Contention
- MOS Doc ID 163424.1 — Cache Buffers Chains Latch
- Tanel Poder — Latch internals
