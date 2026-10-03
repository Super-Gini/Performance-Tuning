# MMNL — Manageability Monitor Lite

## Purpose

Samples **active sessions** every 1 second and writes to the ASH circular buffer in memory. Every 10th sample is written to `DBA_HIST_ACTIVE_SESS_HISTORY` (persisted ASH).

## Behavior

- Loops continuously on 1 s tick.
- Very lightweight — reads `V$SESSION` and its wait state, writes a row.
- Flushes to disk in bulk every 60 seconds or when the in-memory buffer wraps.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(MMNL)%';

-- Current in-memory ASH count
SELECT COUNT(*) FROM v$active_session_history;

-- Persisted ASH growth
SELECT   snap_id, COUNT(*) samples
FROM     dba_hist_active_sess_history
WHERE    sample_time > SYSDATE - 1
GROUP BY snap_id
ORDER BY snap_id DESC
FETCH FIRST 10 ROWS ONLY;
```

## Related Views

- `V$ACTIVE_SESSION_HISTORY` — in-memory buffer (~1 hour).
- `DBA_HIST_ACTIVE_SESS_HISTORY` — persisted (weeks).
- `V$SESSION` — MMNL's sampling source.

## Common Issues

- **ASH shows huge time skips** — MMNL slept because CPU starvation. Rare.
- **MMNL spinning** — Bug; check for open SR against version.
- **Persisted ASH not growing** — MMON side (not MMNL) writes the persisted rows. See [MMON](mmon.md).

## References

- Oracle Database Performance Tuning Guide 19c — ASH
- [ASH](../../12-performance-tuning/ash.md)
