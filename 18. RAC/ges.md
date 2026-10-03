# GES — Global Enqueue Services

## Overview

**Global Enqueue Services (GES)** is the RAC subsystem that coordinates **non-block enqueues** across instances: TM locks, TX locks, library cache locks, row cache locks, and dozens of internal enqueue types. Where GCS handles blocks, GES handles everything else — with equivalent 2-way/3-way coordination.

## Enqueue Types Managed

All enqueues visible in `V$LOCK` become global in RAC:

- **TM** — table DML lock.
- **TX** — transaction lock.
- **UL** — user lock.
- **HW** — high water mark.
- **SQ** — sequence cache.
- **LB** — library cache.
- Many more.

## Processes

- **LMON** — Global Enqueue Service Monitor. Recovers dead instances. Handles reconfiguration.
- **LMD** — Lock Manager Daemon. Handles GES requests and messages.
- **LCK0** — Local instance lock.
- **LCK1..N** — Additional lock processes.

## Wait Events

Cluster-class waits related to GES:

- `gc cr grant 2-way` / `3-way` — grant for CR (read) access.
- `gc current grant 2-way` / `3-way` — grant for CURRENT (write) access.
- `gc cr grant congested` / `gc current grant congested` — GES load.

## Diagnostic Queries

```sql
-- GES statistics
SELECT name, value FROM v$sysstat WHERE name LIKE 'ges%' OR name LIKE 'global lock%';

-- GES enqueue stats
SELECT * FROM v$ges_enqueue_stat;

-- GES resource summary
SELECT * FROM v$ges_statistics;

-- Enqueues currently held cluster-wide
SELECT eq_type, req_reason,
       block_state, grant_time_ms
FROM   gv$ges_convert_local
FETCH FIRST 20 ROWS ONLY;

-- Cross-node blocking
SELECT s.inst_id, s.sid, s.username, s.blocking_session,
       s.blocking_instance, s.event, s.seconds_in_wait
FROM   gv$session s
WHERE  s.blocking_session IS NOT NULL
ORDER  BY s.seconds_in_wait DESC;
```

## Common Issues

- **`gc cr grant congested`** — GES overloaded; check LMD activity.
- **Cross-node deadlocks** — GES detects globally; trace file contains cluster-wide deadlock graph.
- **Enqueue-heavy workloads** — Consider service partitioning.

## Best Practices

1. Fast interconnect.
2. Service-based partitioning.
3. Monitor GES statistics.
4. `enqueue_hash_chains` sized appropriately (auto).
5. Alert on cross-node deadlocks.

## Interview Questions

1. **Q:** What is GES?
   **A:** Global Enqueue Services — coordinates non-block enqueues (TM, TX, LB, etc.) across RAC instances.

2. **Q:** GCS vs GES?
   **A:** GCS handles blocks. GES handles enqueues (locks).

3. **Q:** LMON's role?
   **A:** Recovers dead instances, handles cluster reconfiguration.

4. **Q:** Cross-node deadlocks?
   **A:** GES detects and produces cluster-wide deadlock graph in trace file.

## References

- Oracle Real Application Clusters Administration 19c
- MOS Doc ID 730079.1 — Cluster Enqueue Services
