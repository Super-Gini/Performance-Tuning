# MMAN — Memory Manager

## Purpose

Runs the automatic tuning behind **ASMM** (Automatic Shared Memory Management, `SGA_TARGET`) and **AMM** (Automatic Memory Management, `MEMORY_TARGET`). Resizes SGA components (buffer cache, shared pool, large pool, java pool, streams pool) based on demand and advice.

## Behavior

- Wakes on demand — sizing decisions driven by `V$MEMORY_TARGET_ADVICE` and `V$SGA_TARGET_ADVICE`.
- Only active when `SGA_TARGET > 0` (ASMM) or `MEMORY_TARGET > 0` (AMM).
- If `_MMAN_AUTOMATIC_TUNING=FALSE`, MMAN honors only granule-level rebalances.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(MMAN)%';

-- Current sizing
SHOW PARAMETER memory_target
SHOW PARAMETER sga_target

-- Advice
SELECT * FROM v$sga_target_advice ORDER BY sga_size;
SELECT * FROM v$memory_target_advice ORDER BY memory_size;

-- Resize history
SELECT   component, oper_type, initial_size/1024/1024 initial_mb,
         final_size/1024/1024 final_mb, status, start_time, end_time
FROM     v$memory_resize_ops
ORDER BY start_time DESC
FETCH FIRST 20 ROWS ONLY;
```

## Related Views

- `V$MEMORY_TARGET_ADVICE` — AMM advice.
- `V$SGA_TARGET_ADVICE` — ASMM advice.
- `V$MEMORY_RESIZE_OPS` — history of MMAN actions.
- `V$MEMORY_DYNAMIC_COMPONENTS` — current sizes.

## Common Issues

- **`ORA-04031` even with AMM** — MMAN was too slow / minimum sizes hit. Set explicit floors via `_shared_pool_reserved_min_alloc` or bump `MEMORY_TARGET`.
- **Frequent thrashing between buffer cache and shared pool** — Set `DB_CACHE_SIZE` and `SHARED_POOL_SIZE` as floors.
- **Not using MMAN at all** — `SGA_TARGET=0` and `MEMORY_TARGET=0`. Manual mode.

## Best Practices

1. Use **ASMM** in 19c (`SGA_TARGET` + manual `PGA_AGGREGATE_TARGET`) — not AMM (which is buggy on Linux with HugePages).
2. Set floor sizes for buffer cache and shared pool to prevent thrashing.
3. Monitor `V$MEMORY_RESIZE_OPS` — many resizes/hour = under-sized.

## References

- Oracle Database Administrator's Guide 19c — Automatic Memory Management
- [ASMM](../../03-instance-architecture/memory/asmm.md), [AMM](../../03-instance-architecture/memory/amm.md)
