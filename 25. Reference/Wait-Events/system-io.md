# System I/O Wait Events

## Overview

The **System I/O** wait class covers I/O performed by background processes — DBWn writes, LGWR writes, ARCn reads/writes, control file operations. Foreground I/O lives in [User I/O](user-io.md).

## Events

### `db file parallel write` (DBWn)

DBWn writing dirty buffers in parallel batches.

**Parameters**:

- `p1` — files (count of files being written).
- `p2` — blocks.

**Interpretation**: Average > 20 ms suggests storage saturated. Correlates with buffer cache hit ratio, IOPS.

### `log file parallel write` (LGWR)

LGWR writing redo. Same interpretation as `log file sync` from the LGWR side.

### `control file parallel write` (CKPT)

Writing control file — at every log switch, tablespace add, backup metadata update.

**Long waits**: control file is on slow storage — move it.

### `control file sequential read`

Reading control file. Common during recovery / backup metadata queries.

### `db file async I/O submit`

DBWn asynchronously submitting I/O.

### `Log archive I/O`

ARCn reading from redo, writing to archive dest.

### `RMAN backup & recovery I/O`

RMAN's IO — read from datafile, write to backup piece.

### `Standby redo I/O`

Standby side — RFS writing to standby redo logs.

## Diagnostic Query

```sql
SELECT   event, wait_class, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(average_wait,3) cs
FROM     v$system_event
WHERE    wait_class = 'System I/O' AND time_waited > 0
ORDER BY time_waited DESC;
```

## Underlying I/O Stats

```sql
-- Per file I/O (foreground + background)
SELECT   file#, phyrds, phywrts, ROUND(avgwait,3) avgwait_cs,
         ROUND(readtim/GREATEST(phyrds,1),3) avg_read_cs,
         ROUND(writetim/GREATEST(phywrts,1),3) avg_write_cs
FROM     v$filestat
ORDER BY phyrds+phywrts DESC
FETCH FIRST 20 ROWS ONLY;
```

## References

- Oracle Database Reference 19c — Wait event index
- MOS Doc ID 34405.1 — System I/O troubleshooting
