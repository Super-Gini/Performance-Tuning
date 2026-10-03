# User I/O Wait Events

## Overview

**User I/O** — the foreground process waiting for data. `db file %` events dominate this class. When SQL is slow and CPU isn't the issue, you're almost always looking at User I/O.

## Events

### `db file sequential read`

Single-block read — usually **index access** or **rowid lookup**.

**Parameters**:

- `p1` — file#.
- `p2` — block#.
- `p3` — always 1 (single block).

**Fix**:

- Better index (avoid unnecessary lookups).
- Increase buffer cache.
- Faster storage.

### `db file scattered read`

Multi-block read — full table scan or fast full index scan.

**Parameters**:

- `p1` — file#.
- `p2` — first block#.
- `p3` — number of blocks (up to `DB_FILE_MULTIBLOCK_READ_COUNT`).

**Fix**:

- Index instead of full scan (only if filtering enough).
- Adjust `DB_FILE_MULTIBLOCK_READ_COUNT`.
- Storage tier.

### `direct path read`

Reads bypass buffer cache — big scans (11g+ auto for large tables), parallel queries, LOB reads.

**Parameters**: Same shape.

**Interpretation**: The optimizer picked this method for a full scan of a "big" table. Not necessarily wrong, but memory hits are cheaper. Turn off adaptive direct read for critical repeated scans via `_serial_direct_read=NEVER`.

### `direct path read temp`

Same but reading from a TEMP tablespace — hash join spill, sort spill.

**Fix**:

- Bigger PGA (raises the workarea sizes).
- Better plan (avoid huge intermediate results).
- Faster TEMP storage.

### `direct path write`

Writing back to temp (sort spill) or datafile (direct path insert).

### `direct path write temp`

Sort/hash spill to TEMP.

### `read by other session`

Waiting for another session to finish loading the block into buffer cache.

**Fix**: Cache thrashing — grow buffer cache or reduce concurrent full scans.

### `Disk file operations I/O`

File open/close/create/resize/delete — usually on ADR / temp file.

### `local write wait`

Waiting on a local (buffered) write. Rare.

## Diagnostic Query

```sql
SELECT   event, wait_class, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(time_waited/GREATEST(total_waits,1)/100,3) avg_secs
FROM     v$system_event
WHERE    wait_class = 'User I/O' AND time_waited > 0
ORDER BY time_waited DESC;
```

## `V$IOSTAT_FILE`

Per-file, per-op-type I/O:

```sql
SELECT   file_no, filetype_name,
         small_read_reqs, small_read_megabytes,
         small_write_reqs, small_write_megabytes,
         large_read_reqs, large_read_megabytes,
         large_write_reqs, large_write_megabytes,
         ROUND(retries_on_error,0) retry_errors
FROM     v$iostat_file
ORDER BY small_read_reqs + small_write_reqs DESC
FETCH FIRST 20 ROWS ONLY;
```

## References

- Oracle Database Performance Tuning Guide 19c
- MOS Doc ID 34405.1
- [I/O Analysis](../../12-performance-tuning/io-analysis.md)
