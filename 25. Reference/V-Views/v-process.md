# V$PROCESS

## Purpose

One row per **OS process** attached to the instance — both foreground (server) and background. Join with `V$SESSION` via `PADDR` (session) → `ADDR` (process).

## Key Columns

| Column                                         | Meaning                                         |
| ---------------------------------------------- | ----------------------------------------------- |
| `ADDR`                                         | Process address (join to `V$SESSION.PADDR`).    |
| `PID`                                          | Oracle PID (small integer).                     |
| `SPID`                                         | OS process ID (`ps -ef` value).                 |
| `USERNAME`                                     | OS user of the process.                         |
| `PROGRAM`                                      | Client program image.                           |
| `TERMINAL`                                     | Terminal / IP.                                  |
| `BACKGROUND`                                   | `1` for background, NULL for user.              |
| `PNAME`                                        | Short name (`PMON`, `SMON`, ...).               |
| `TRACEFILE`                                    | Trace file path for this process — very useful. |
| `PGA_USED_MEM`, `PGA_ALLOC_MEM`, `PGA_MAX_MEM` | PGA sizes.                                      |
| `EXECUTION_TYPE`, `NUMA_ID`                    | RAC / NUMA info.                                |

## Common Queries

```sql
-- SID -> SPID (kill from OS)
SELECT s.sid, s.serial#, p.spid, p.tracefile
FROM   v$session s JOIN v$process p ON p.addr = s.paddr
WHERE  s.sid = &target_sid;

-- Foreground processes with big PGA
SELECT p.spid, s.sid, s.username, s.program,
       ROUND(p.pga_used_mem/1024/1024,1) used_mb,
       ROUND(p.pga_max_mem/1024/1024,1)  max_mb
FROM   v$process p JOIN v$session s ON p.addr = s.paddr
WHERE  p.background IS NULL
ORDER  BY p.pga_max_mem DESC
FETCH  FIRST 20 ROWS ONLY;

-- Background processes
SELECT pname, spid, tracefile
FROM   v$process
WHERE  background = 1
ORDER  BY pname;
```

## Related

- `V$SESSION` — user-visible session.
- `V$BGPROCESS` — background process registry.
- `GV$PROCESS` — cluster-wide.

## References

- Oracle Database Reference 19c — `V$PROCESS`
