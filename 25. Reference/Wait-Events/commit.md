# Commit Wait Events

## Overview

Wait events in the **Commit** class fire when a session is waiting for `LGWR` to persist redo. These are on the critical path of every transaction's `COMMIT`.

## Events

### `log file sync`

**Meaning**: Session waiting for LGWR to acknowledge that the redo covering its commit has been persisted to disk.

**Parameters**:

- `p1` — buffer# in redo log buffer.
- `p2` — SCN wait for.

**Typical duration**: <5 ms on fast storage; 10–50 ms is workable; >50 ms is a problem.

**Fix path**:

1. Check average wait: > 10 ms suggests LGWR or storage issue.
2. `log file parallel write` — LGWR's underlying I/O. If also high, storage is the bottleneck.
3. Commit rate — batching commits (`commits/second`) reduces total waits.

```sql
-- Average wait
SELECT event, total_waits, ROUND(time_waited_micro/1e6, 2) secs,
       ROUND(time_waited_micro/GREATEST(total_waits,1)/1000, 3) avg_ms
FROM   v$system_event
WHERE  event = 'log file sync';
```

**Common causes**:

- Slow redo storage.
- Too-small redo buffer forcing frequent LGWR wakes.
- Very high commit rate — application should batch.
- LGWR busy — RAC synchronous redo transport can add latency.

### `log file parallel write`

LGWR's own wait for the OS write to complete. Not user-facing — it's a background event — but a leading indicator: `log file sync` can't be faster than `log file parallel write`.

**Parameters**:

- `p1` — file #.
- `p2` — first block.
- `p3` — blocks written.

### `log file switch (checkpoint incomplete)`

Session waiting because LGWR wants to switch to the next redo log, but the checkpoint on that log hasn't finished. Foreground processes are blocked.

**Fix**: Grow redo log files. Rule of thumb: 15–20 minute switch interval.

### `log file switch (archiving needed)`

Same idea but ARCn hasn't archived the log yet. Fix: increase `LOG_ARCHIVE_MAX_PROCESSES`, faster archive dest storage, or add more redo groups.

### `log file switch completion`

Waiting for a normal log switch to complete. Small waits, occasional.

### `log buffer space`

Redo log buffer full — foreground process waiting to insert redo. Rare in 19c (auto-sized). If seen: bump `LOG_BUFFER`.

## Diagnostic Query

```sql
SELECT event, total_waits,
       ROUND(time_waited/100,1) secs,
       ROUND(average_wait,2) cs
FROM   v$system_event
WHERE  event LIKE 'log%'
ORDER  BY time_waited DESC;
```

## References

- Oracle Database Reference 19c — Wait event index
- MOS Doc ID 34592.1 — LGWR waits
- [LGWR](../background-processes/lgwr.md)
