# SMON — System Monitor

## Purpose

- Performs **instance recovery** (crash recovery) at startup — rolls forward from redo, rolls back uncommitted transactions.
- Coalesces free space in dictionary-managed tablespaces (legacy).
- Cleans up temporary segments in `TEMP` tablespaces.
- Drops OBJ$ metadata for dropped objects that couldn't be dropped online.
- Reclaims space from expired transactions in undo tablespaces.
- Maintains `SMON_SCN_TIME` (scn↔timestamp map) — 5-minute buckets.

## When It Wakes

- ~5 minutes idle interval.
- Instance startup (recovery phase).
- On demand for temp cleanup / undo cleanup.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(SMON)%';

-- Temp segments SMON will eventually clean
SELECT tablespace_name, extents, blocks
FROM   dba_segments
WHERE  segment_type = 'TEMPORARY';

-- SCN-time map SMON maintains
SELECT * FROM smon_scn_time ORDER BY time_dp DESC FETCH FIRST 5 ROWS ONLY;
```

## Related Views

- `V$FAST_START_TRANSACTIONS` — SMON's rollback queue after recovery.
- `V$RECOVERY_STATUS` — recovery in progress.
- `SMON_SCN_TIME` — SCN↔time mappings.

## Common Issues

- **`SMON: enable cache recovery`** at startup takes long — Massive uncommitted transactions to roll back. `_fast_start_parallel_rollback` to speed up.
- **Temp not shrinking** — SMON only reclaims when no session holds. Check `V$SORT_USAGE`.
- **`SMON_SCN_TIME` gaps** — instance was down or clock skewed.

## References

- Oracle Database Concepts 19c — SMON
- MOS Doc ID 386385.1 — SMON temp cleanup
