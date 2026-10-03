# Memory Parameters

## Overview

Initialization parameters that control Oracle memory allocation. Two management models:

- **AMM** — `MEMORY_TARGET`, `MEMORY_MAX_TARGET`. Auto-manages SGA + PGA together. Not recommended on Linux with HugePages.
- **ASMM** — `SGA_TARGET`, `SGA_MAX_SIZE` + explicit `PGA_AGGREGATE_TARGET`. Recommended in 19c.
- **Manual** — each pool set individually (`DB_CACHE_SIZE`, `SHARED_POOL_SIZE`, etc.). Rare; only for edge cases.

## Top-Level Parameters

| Parameter              | Purpose                                          |
| ---------------------- | ------------------------------------------------ |
| `MEMORY_TARGET`        | Total (SGA+PGA) target. AMM.                     |
| `MEMORY_MAX_TARGET`    | Upper bound for MEMORY_TARGET.                   |
| `SGA_TARGET`           | Total SGA. ASMM.                                 |
| `SGA_MAX_SIZE`         | Upper bound for SGA.                             |
| `PGA_AGGREGATE_TARGET` | Soft PGA target.                                 |
| `PGA_AGGREGATE_LIMIT`  | Hard PGA cap. 12c+.                              |
| `USE_LARGE_PAGES`      | `TRUE`, `FALSE`, `ONLY`, `AUTO_ONLY`. HugePages. |

## SGA Component Parameters (Floors under ASMM)

Set these to prevent MMAN from shrinking below sensible sizes:

| Parameter                   | Purpose                                          |
| --------------------------- | ------------------------------------------------ |
| `DB_CACHE_SIZE`             | Buffer cache floor.                              |
| `DB_KEEP_CACHE_SIZE`        | KEEP pool.                                       |
| `DB_RECYCLE_CACHE_SIZE`     | RECYCLE pool.                                    |
| `DB_nK_CACHE_SIZE`          | Non-default block size pools (2K/4K/8K/16K/32K). |
| `SHARED_POOL_SIZE`          | Shared pool floor.                               |
| `SHARED_POOL_RESERVED_SIZE` | Reserved portion for large allocs.               |
| `LARGE_POOL_SIZE`           | Large pool (parallel, RMAN, shared server).      |
| `JAVA_POOL_SIZE`            | Java pool.                                       |
| `STREAMS_POOL_SIZE`         | Streams/AQ/GoldenGate.                           |
| `RESULT_CACHE_MAX_SIZE`     | Result cache upper limit.                        |
| `DB_FLASH_CACHE_SIZE`       | Flash cache (SSD extension).                     |

## Redo Log Buffer

| Parameter    | Purpose                                           |
| ------------ | ------------------------------------------------- |
| `LOG_BUFFER` | Redo log buffer size. Default 5–14 MB auto-sized. |

## Recommended Approach (19c)

```sql
-- Disable AMM, enable ASMM
ALTER SYSTEM RESET MEMORY_TARGET SCOPE=SPFILE;
ALTER SYSTEM RESET MEMORY_MAX_TARGET SCOPE=SPFILE;

ALTER SYSTEM SET SGA_MAX_SIZE = 32G SCOPE=SPFILE;
ALTER SYSTEM SET SGA_TARGET   = 32G SCOPE=SPFILE;

ALTER SYSTEM SET DB_CACHE_SIZE     = 20G SCOPE=BOTH;
ALTER SYSTEM SET SHARED_POOL_SIZE  =  6G SCOPE=BOTH;
ALTER SYSTEM SET LARGE_POOL_SIZE   =  1G SCOPE=BOTH;

ALTER SYSTEM SET PGA_AGGREGATE_TARGET = 10G SCOPE=BOTH;
ALTER SYSTEM SET PGA_AGGREGATE_LIMIT  = 20G SCOPE=BOTH;

ALTER SYSTEM SET USE_LARGE_PAGES = 'ONLY' SCOPE=SPFILE;
```

Bounce required for `SGA_MAX_SIZE` and `USE_LARGE_PAGES`.

## Query Current Values

```sql
COLUMN name FORMAT A35
SHOW PARAMETER memory
SHOW PARAMETER sga
SHOW PARAMETER pga

-- Current dynamic sizes
SELECT component, current_size/1024/1024 mb, min_size/1024/1024 min_mb,
       max_size/1024/1024 max_mb
FROM   v$sga_dynamic_components
ORDER  BY current_size DESC;

-- Advice
SELECT * FROM v$sga_target_advice ORDER BY sga_size;
SELECT * FROM v$pga_target_advice ORDER BY pga_target_for_estimate;
```

## References

- Oracle Database Reference 19c — Init parameters
- [SGA](../../03-instance-architecture/memory/sga.md), [ASMM](../../03-instance-architecture/memory/asmm.md), [HugePages](../../03-instance-architecture/memory/hugepages.md)
