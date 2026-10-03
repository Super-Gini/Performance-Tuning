# Background Process Reference

## Overview

Every Oracle instance starts a fixed set of **background processes** — one per role, with a few (`Jnnn`, `ARCn`, `DWnn`) auto-scaled. This page lists them all with the one-line purpose and where to read more.

## Complete Process List (19c)

| Name   | Full Name                  | Purpose                                            | Deep dive       |
| ------ | -------------------------- | -------------------------------------------------- | --------------- |
| `PMON` | Process Monitor            | Cleans up failed sessions.                         | [PMON](pmon.md) |
| `CLMN` | Cleanup Master             | Newer PMON helper (18c+).                          | –               |
| `CLnn` | Cleanup Slave              | Parallel cleanup work.                             | –               |
| `SMON` | System Monitor             | Instance recovery, coalesces free space.           | [SMON](smon.md) |
| `DBWn` | Database Writer            | Flushes dirty buffers to datafiles.                | [DBWn](dbwn.md) |
| `LGWR` | Log Writer                 | Flushes redo buffer to online redo logs.           | [LGWR](lgwr.md) |
| `LGnn` | Log Writer Worker (12.2+)  | Parallel LGWR with `_use_single_log_writer=FALSE`. | –               |
| `CKPT` | Checkpoint                 | Updates data file headers with checkpoint SCN.     | [CKPT](ckpt.md) |
| `ARCn` | Archiver                   | Copies filled redo logs to archive destinations.   | [ARCn](arcn.md) |
| `RECO` | Recoverer                  | Resolves in-doubt distributed transactions.        | [RECO](reco.md) |
| `MMON` | Manageability Monitor      | AWR snapshot, ADDM run, monitors SGA advice.       | [MMON](mmon.md) |
| `MMNL` | Manageability Monitor Lite | ASH sampling every second.                         | [MMNL](mmnl.md) |
| `MMAN` | Memory Manager             | ASMM/AMM auto-tuning.                              | [MMAN](mman.md) |
| `CJQ0` | Job Coordinator            | Spawns `Jnnn` slaves for Scheduler jobs.           | –               |
| `Jnnn` | Job Slave                  | Runs individual Scheduler jobs.                    | –               |
| `FBDA` | Flashback Data Archiver    | Populates FDA history tables.                      | [FBDA](fbda.md) |
| `VKTM` | Virtual Keeper of Time     | High-resolution clock source.                      | [VKTM](vktm.md) |
| `DIAG` | Diagnosability Coordinator | Trace and diagnostic dumps.                        | –               |
| `DIA0` | Diagnostic Slave           | Hang analysis.                                     | –               |
| `LREG` | Listener Registration      | Registers services with listener.                  | [LREG](lreg.md) |
| `TT00` | Trace Coordinator          | Space transaction manager.                         | –               |
| `PSP0` | Process Spawner            | Spawns other background processes.                 | –               |
| `RVWR` | Recovery Writer            | Flashback log writer.                              | –               |
| `SPnn` | Statistics Coordinator     | Auto stats gathering slaves.                       | –               |
| `Wnnn` | Space Management Slave     | Segment space management.                          | –               |
| `QMnn` | Advanced Queue Master      | AQ propagation.                                    | –               |
| `Qnnn` | Advanced Queue Slave       | AQ dequeue.                                        | –               |
| `Snnn` | Shared Server              | Serves shared server clients.                      | –               |
| `Dnnn` | Dispatcher                 | Routes shared server requests.                     | –               |
| `Onnn` | Objects Compilation        | Recompiles PL/SQL objects.                         | –               |

### RAC-Specific

| Name      | Purpose                                                  |
| --------- | -------------------------------------------------------- |
| `LMS0..N` | Global Cache Service — ships blocks between instances.   |
| `LMD`     | Global Enqueue Service Daemon — remote lock management.  |
| `LMON`    | Global Enqueue Service Monitor — reconfig on join/leave. |
| `LCK0`    | Instance Enqueue Process — non-cache resources.          |
| `RMSn`    | RAC Management Slave.                                    |
| `RSMN`    | Remote Slave Monitor.                                    |
| `ACMS`    | Atomic Controlfile-to-Memory Service.                    |

### Data Guard (Physical Standby)

| Name   | Purpose                                        |
| ------ | ---------------------------------------------- |
| `NSSn` | Network Server Sync — SYNC redo transport.     |
| `NSAn` | Network Server Async — ASYNC redo transport.   |
| `LNS`  | Log Network Server (legacy).                   |
| `MRP0` | Managed Recovery Process — applies redo.       |
| `RFSn` | Remote File Server — receives redo on standby. |

### ASM Instance

| Name   | Purpose                           |
| ------ | --------------------------------- |
| `RBAL` | Rebalance coordinator.            |
| `ARBn` | Rebalance slaves.                 |
| `GMON` | Diskgroup monitor.                |
| `MARK` | Mark stale allocation units.      |
| `PING` | Peer detection for ASM instances. |

## Quick Query

```sql
-- All background processes currently active
SELECT   pname, program, spid, wait_class, state
FROM     v$process p
JOIN     v$bgprocess b ON b.paddr = p.addr
ORDER BY pname;

-- Or including background sessions
SELECT p.pname, s.sid, s.event, s.wait_class, s.state
FROM   v$process p
JOIN   v$session s ON s.paddr = p.addr
WHERE  s.type = 'BACKGROUND'
ORDER  BY p.pname;
```

## Ballpark Counts on a Normal Instance

- 30–50 background processes for a plain single-instance non-RAC 19c DB.
- +10–15 on RAC (LMS, LMD, LCK, LMON, etc.).
- +5–10 on Data Guard primary or standby.
- +5–10 on ASM.

## References

- Oracle Database Reference 19c — Background Processes
- MOS Doc ID 69642.1 — Background process meanings
- MOS Doc ID 396940.1 — RAC-specific processes
