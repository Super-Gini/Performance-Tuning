# CKPT — Checkpoint

## Purpose

Signals DBWn to write dirty buffers up to a specific SCN, then updates the **control file** and **datafile headers** with the checkpoint SCN. Does not write data blocks itself — that's DBWn.

## Types of Checkpoints

- **Full checkpoint** — all dirty buffers written. Triggered by `SHUTDOWN NORMAL/IMMEDIATE`, `ALTER SYSTEM CHECKPOINT`, tablespace offline.
- **Incremental checkpoint** — continuous background trickle. Controlled by `FAST_START_MTTR_TARGET`.
- **Log switch checkpoint** — at every redo log switch.
- **Object-level** — e.g., tablespace begin backup.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(CKPT)%';

-- Checkpoint progress info
SELECT   name, value
FROM     v$sysstat
WHERE    name IN ('DBWR checkpoint buffers written','DBWR checkpoints',
                  'redo log space requests');

-- Estimated MTTR
SELECT   estimated_mttr, target_mttr, log_file_size_redo_blks,
         log_chkpt_timeout_redo_blks
FROM     v$instance_recovery;
```

## Related Views

- `V$DATABASE.CHECKPOINT_CHANGE#` — last full checkpoint SCN.
- `V$DATAFILE.CHECKPOINT_CHANGE#` — per-datafile.
- `V$INSTANCE_RECOVERY` — MTTR targets and reality.
- `V$SYSSTAT` — checkpoint metrics.

## Common Issues

- **`log file switch (checkpoint incomplete)`** — Redo cycling faster than DBW can finish the checkpoint. Grow redo logs or reduce `FAST_START_MTTR_TARGET`.
- **CKPT CPU high** — Rare; usually huge SGA + tight MTTR.
- **`Checkpoint not complete`** in alert log — Same as above.

## Best Practices

1. Set `FAST_START_MTTR_TARGET=300` seconds — 5-minute recovery target.
2. Size redo logs to switch every 15–20 min.
3. Never disable checkpointing — data-loss risk.

## References

- Oracle Database Concepts 19c — Checkpoints
- MOS Doc ID 465191.1 — Checkpoint tuning
- [Checkpoints deep dive](../../06-redo/checkpoints.md)
