# PMON — Process Monitor

## Purpose

Cleans up after failed user or server processes: rolls back their transactions, releases their locks and resources, frees their PGA. Also registers the instance with the listener (delegated to `LREG` in 12.1+).

## Behavior

- Wakes up every ~3 s (or on demand) to look for dead processes.
- Rolls back uncommitted transactions of failed sessions.
- Releases enqueues, KGL locks, cursors, latches held by dead processes.
- Restarts failed shared server dispatchers and shared servers.
- In pre-12.1, registered service names with listeners (now `LREG`).

## Check It's Alive

```sql
SELECT program, spid, pid, addr, sosid
FROM   v$process
WHERE  program LIKE '%(PMON)%';

SELECT pname, description
FROM   v$bgprocess
WHERE  pname = 'PMON';
```

OS side:

```bash
ps -ef | grep "ora_pmon_$ORACLE_SID"
```

If PMON dies, the instance crashes immediately.

## Related Views

- `V$PROCESS` — OS-level process state.
- `V$BGPROCESS` — background process registry.
- `V$FAST_START_TRANSACTIONS` — TX being rolled back after crash recovery.

## Common Issues

- **PMON busy after crash** — many uncommitted transactions being rolled back. Watch `V$FAST_START_TRANSACTIONS`.
- **PMON stuck at 100% CPU** — usually cleanup on hundreds of thousands of leaked cursors. Bug — check for open SR.
- **Instance won't start; PMON dies immediately** — bad `init.ora` parameter or SGA/PGA config problem.

## References

- Oracle Database Concepts 19c — Background Processes
- MOS Doc ID 69642.1
