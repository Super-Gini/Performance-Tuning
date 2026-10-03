# Wait Events

## Overview

Oracle's **wait interface** is the DBA's most powerful diagnostic tool. Every time an Oracle process blocks on something (I/O, lock, network, latch), it records the reason as a **wait event** with duration, parameters, and context. Sessions accumulate this data in `V$SESSION_WAIT`, `V$SESSION_EVENT`, `V$SYSTEM_EVENT`, and — crucially — ASH.

Every performance problem manifests as a wait event or CPU consumption. Reading the top waits correctly is the first step in any tuning exercise.

## Wait Classes

Oracle groups events by class:

| Class             | Examples                                                                |
| ----------------- | ----------------------------------------------------------------------- |
| **User I/O**      | `db file sequential read`, `db file scattered read`, `direct path read` |
| **System I/O**    | `log file parallel write`, `control file parallel write`                |
| **Concurrency**   | `latch: shared pool`, `library cache mutex X`, `enq: HW - contention`   |
| **Application**   | `enq: TX - row lock contention`, `SQL*Net message from client` (idle)   |
| **Commit**        | `log file sync`                                                         |
| **Configuration** | `log file switch (checkpoint incomplete)`, `undo segment tx slot`       |
| **Network**       | `SQL*Net more data to client`                                           |
| **Cluster**       | `gc buffer busy`, `gc cr block busy`, `gc current grant congested`      |
| **Scheduler**     | `resmgr:cpu quantum`                                                    |
| **Idle**          | `SQL*Net message from client`, `rdbms ipc message`, `pmon timer`        |

**Idle** events mean the session is not doing work — usually filtered out of top waits.

## Top Wait Events to Know

### User I/O

- **`db file sequential read`** — Single-block read (index / by-rowid). Number-one wait in OLTP. Cause: too many logical reads or slow storage.
- **`db file scattered read`** — Multi-block read (full scans). Cause: full scans (bad plan / missing index / big DW query).
- **`direct path read`** — Bypass buffer cache; large full scan or parallel query.
- **`direct path write temp` / `direct path read temp`** — Sort/hash spill to TEMP.

### Commit

- **`log file sync`** — Commit waiting for LGWR write. See [Commit Processing](../06-redo/commit-processing.md).
- **`log file parallel write`** — LGWR's own I/O wait.

### Concurrency

- **`enq: TX - row lock contention`** — Session waits on another session's uncommitted row.
- **`enq: TX - index contention`** — Right-most index leaf hot; sequences/timestamps.
- **`enq: TX - allocate ITL entry`** — Not enough ITL slots. Increase `INITRANS`.
- **`enq: HW - contention`** — High Water Mark contention (parallel inserts).
- **`buffer busy waits`** — Two sessions want same buffer with incompatible modes.
- **`latch: shared pool`** — Hard parse storm.
- **`latch: cache buffers chains`** — Hot block.
- **`library cache: mutex X`** — Cursor contention.
- **`cursor: pin S wait on X`** — Cursor being modified while others read.

### Configuration

- **`log file switch (checkpoint incomplete)`** — DBWn behind. Enlarge log groups or add DBWn.
- **`log file switch (archiving needed)`** — ARCn behind. Fix FRA or destinations.
- **`free buffer waits`** — Foregrounds can't find free buffer; DBWn behind or cache small.
- **`write complete waits`** — Waiting for specific block flush.

### Cluster (RAC)

- **`gc buffer busy acquire`** — Block requested from another instance, held by someone.
- **`gc cr block busy`** — Waiting for CR construction on remote instance.
- **`gc current grant congested`** — Global cache pressured.
- **`gc cr grant 2-way / 3-way`** — Normal cache fusion — high count is expected in RAC.

## Diagnostic Queries

```sql
-- Top waits since instance start
SELECT event, wait_class, total_waits,
       ROUND(time_waited_micro/1e6, 1) AS total_sec,
       ROUND(time_waited_micro/DECODE(total_waits,0,1,total_waits)/1000, 2) AS avg_ms
FROM   v$system_event
WHERE  wait_class <> 'Idle'
ORDER  BY total_sec DESC
FETCH FIRST 15 ROWS ONLY;

-- Currently waiting sessions
SELECT sid, serial#, username, machine, sql_id, event, wait_class,
       seconds_in_wait, state, p1text, p1, p2text, p2, p3text, p3
FROM   v$session
WHERE  status = 'ACTIVE' AND type = 'USER'
   AND wait_class <> 'Idle'
ORDER  BY seconds_in_wait DESC;

-- Waits over the last hour from ASH
SELECT event, wait_class, COUNT(*) AS samples
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 1/24
   AND event IS NOT NULL
GROUP  BY event, wait_class
ORDER  BY samples DESC
FETCH FIRST 15 ROWS ONLY;

-- Wait event histogram (distribution)
SELECT event, wait_time_milli, wait_count
FROM   v$event_histogram
WHERE  event = 'db file sequential read'
ORDER  BY wait_time_milli;
```

## Wait Event Parameters (p1/p2/p3)

Each event has up to 3 numeric parameters that identify what's being waited on. Meanings differ per event:

| Event                     | p1        | p2     | p3             |
| ------------------------- | --------- | ------ | -------------- |
| `db file sequential read` | file#     | block# | blocks (1)     |
| `db file scattered read`  | file#     | block# | blocks (multi) |
| `enq: TX - row lock`      | type/mode | id1    | id2            |
| `buffer busy waits`       | file#     | block# | class          |
| `latch: X`                | latch#    | number | tries          |

```sql
-- Which block is being waited on?
SELECT sid, event, p1 AS file#, p2 AS block#, p3
FROM   v$session
WHERE  event = 'buffer busy waits';

-- Which segment does that block belong to?
SELECT owner, segment_name, segment_type
FROM   dba_extents
WHERE  file_id = &file AND &block BETWEEN block_id AND block_id + blocks - 1;
```

## Common Patterns

- **CPU dominates, waits are noise** — Focus on SQL efficiency.
- **`db file sequential read` dominates** — I/O bound. Buffer cache small, storage slow, or bad plans.
- **`log file sync` high** — Commit frequency or LGWR bottleneck.
- **`enq:` events** — Concurrency / locking. See [Locking](../13-locking/index.md).
- **`latch:` events** — Internal serialization. Bind vars, cursor sharing.
- **`gc` events** — RAC cache fusion; check interconnect.

## Best Practices

1. Look at top waits **first**, top SQL second.
2. Use ASH for time-based sampling — better than session-level counters.
3. Ignore idle waits.
4. Compare `avg_ms` to expected latency — 20 ms avg on `log file sync` = LGWR bottleneck.
5. p1/p2/p3 tell you exactly what's being waited on — don't ignore them.
6. Wait histograms (`V$EVENT_HISTOGRAM`) show distribution, not just averages — outliers matter.
7. Correlate wait events with wall-clock time — problem duration, not just count.

## Interview Questions

1. **Q:** What is the wait interface?
   **A:** Oracle's mechanism for recording what sessions are waiting on — event name, parameters, duration.

2. **Q:** Top 3 wait events in OLTP?
   **A:** `db file sequential read`, `log file sync`, `latch: cache buffers chains` (varies).

3. **Q:** How do you find the current wait event for a session?
   **A:** `SELECT event, p1, p2, p3, seconds_in_wait FROM v$session WHERE sid = X;`.

4. **Q:** Idle vs non-idle?
   **A:** Idle: session not working (e.g., waiting for client). Non-idle: session blocked on real work.

5. **Q:** What does p1/p2/p3 mean for `db file sequential read`?
   **A:** file#, block#, block count (1). Points to exact block being read.

6. **Q:** Difference `log file sync` and `log file parallel write`?
   **A:** Sync = foreground total wait for LGWR. Parallel write = LGWR's own I/O time.

7. **Q:** `enq: TX - row lock contention` — cause?
   **A:** Row is locked by another uncommitted transaction. Investigate blocker.

## References

- Oracle Database Reference 19c — Wait Events
- Cary Millsap, _Optimizing Oracle Performance_
- MOS Doc ID 61998.1 — Wait Event Analysis
- MOS Doc ID 34592.1 — Wait Interface
