# VKTM — Virtual Keeper of Time

## Purpose

Maintains a high-resolution wall-clock time source for the database — used by wait event timings, ASH sample times, SCN generation heuristics. Runs at very high priority (near-realtime).

## Behavior

- Wakes very frequently (every ~1 ms in "high-res" mode, ~20 ms in "low-res").
- Publishes microsecond-resolution time into SGA for all other processes to read.
- OS thread priority is elevated at startup.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(VKTM)%';

-- Resolution mode (from alert log at startup)
!grep VKTM $ORACLE_BASE/diag/rdbms/PRD/PRD1/trace/alert_PRD1.log | tail
```

Look for:

```
VKTM started with pid=6, OS id=1234 at elevated priority
VKTM running at (100ms) precision
```

## Common Issues

- **`Time drift detected from CKPT`** in alert log — OS clock jumped (NTP correction). Usually cosmetic; check `date -u` vs peers.
- **VKTM high CPU** — Kernel scheduler starvation. On virtualized environments, set VM's CPU affinity so VKTM isn't preempted.
- **`VKTM not receiving CPU quantum`** — VM host oversubscribed; the DB host isn't getting scheduled fast enough. Fix the VM host.

## References

- MOS Doc ID 1385320.1 — VKTM CPU
- MOS Doc ID 1553838.1 — Time drift
