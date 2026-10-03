# LGWR — Log Writer

## Purpose

Writes the **redo log buffer** to the current online redo log file. Every committed transaction waits for LGWR (`log file sync`), so LGWR performance is on the critical path of every commit.

## Triggers

- On `COMMIT`.
- Every 3 seconds.
- When redo buffer 1/3 full (or 1 MB filled, whichever first).
- Before DBWn writes a corresponding dirty buffer (log-first rule).

## Multiple LGWR Workers (12.2+)

`_use_single_log_writer=FALSE` (default TRUE in 19c) enables multi-worker LGWR (`LGnn`). Rarely toggled; the single-writer path handles most workloads well.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(LGWR)%';

-- Commit waits and redo throughput
SELECT   name, value FROM v$sysstat
WHERE    name IN ('redo writes','redo size','redo write time',
                  'redo blocks written','log file sync');

-- Wait event
SELECT event, total_waits, time_waited/100 secs, average_wait/100 avg_secs
FROM   v$system_event
WHERE  event IN ('log file sync','log file parallel write');
```

## Related Views

- `V$SESSION_WAIT` for `log file sync`.
- `V$LOG` — group state (CURRENT, ACTIVE, INACTIVE).
- `V$LOG_HISTORY` — switch history.

## Common Issues

- **`log file sync` slow (>10 ms avg)** — Storage or fsync latency. Check disk with `V$IOSTAT_FILE`.
- **`log file parallel write` slow** — Similar; the underlying IO for redo.
- **`log file switch (archiving needed)`** — LGWR wants next log; ARCn hasn't archived it. Bump `LOG_ARCHIVE_MAX_PROCESSES`.
- **`log file switch completion`** — Log switch in progress.
- **LGWR CPU spinning** — Bug or resource contention. Check `_use_single_log_writer`.

## Best Practices

1. Redo on fastest storage — dedicated NVMe or ASM diskgroup.
2. Multiplex to two locations minimum.
3. Size redo logs so switch every 15–20 min under peak load.
4. Never enable `NOLOGGING` widely — LGWR still writes for other redo.
5. On RAC, LGWR-per-instance — measure each independently.

## References

- Oracle Database Concepts 19c — LGWR
- MOS Doc ID 34592.1 — LGWR tuning
- [Redo Tuning](../../06-redo/redo-tuning.md)
