# Wait Event Framework

## Overview

Every time Oracle sleeps waiting on something, it emits a **wait event**. Every wait event has a name, a wait class, three positional parameters (P1/P2/P3), an ID, and a timing bucket. Together these form the primary lens for performance diagnosis — from `V$SESSION` to ASH to AWR to SQL Monitor. Understanding the framework itself (not just the events) is what turns "top event is X, look up X on MOS" into "here's why X is happening".

This page maps the framework: event lifecycle, parameter semantics, timing internals, histograms, and the KSLE layer.

## The KSLE Layer

Wait event tracking is implemented in Oracle's KSLE (Kernel Service Latch/Event) layer:

- **`kslwtbctx`** — Begin wait context. Called when the process starts to sleep.
- **`kslwtectx`** — End wait context. Called on wake-up.
- **`kskthbwt`** — Set post-wait state.

Between `kslwtbctx` and `kslwtectx`, Oracle records the event ID + parameters. On end, elapsed time is bucketed into per-session and per-system counters.

## Event Anatomy

Every event has:

| Attribute        | Source                                                                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `EVENT_ID`       | 32-bit hash of the event name. Stable across DB restarts.                                                                                                      |
| `NAME`           | Human-readable event name.                                                                                                                                     |
| `WAIT_CLASS`     | Group: `Idle`, `User I/O`, `System I/O`, `Concurrency`, `Application`, `Commit`, `Cluster`, `Network`, `Administrative`, `Configuration`, `Queueing`, `Other`. |
| `WAIT_CLASS_ID`  | 32-bit hash of wait class name.                                                                                                                                |
| `P1`, `P2`, `P3` | Event-specific parameters.                                                                                                                                     |
| `PARAMETER1/2/3` | Names of P1/P2/P3 (for interpretation).                                                                                                                        |

Explore:

```sql
SELECT event_id, name, wait_class, wait_class_id,
       parameter1, parameter2, parameter3
FROM   v$event_name
WHERE  name = 'db file sequential read';
```

Result:

```
EVENT_ID  NAME                        WAIT_CLASS  PARAMETER1  PARAMETER2  PARAMETER3
2652584166 db file sequential read    User I/O    file#       block#      blocks
```

## Interpreting P1/P2/P3

Each event's parameters mean something specific. Not universal.

### `db file sequential read`

- `P1` = file#.
- `P2` = block#.
- `P3` = blocks (usually 1).

Recover the object:

```sql
SELECT owner, segment_name, segment_type
FROM   dba_extents
WHERE  file_id = &p1
   AND &p2 BETWEEN block_id AND block_id + blocks - 1;
```

### `enq: TX - row lock contention`

- `P1` = enqueue type + mode (packed): `1415053318` decodes to type `TX`, mode 6 (X).
- `P2` = XID's `xidusn.xidslot` combined.
- `P3` = `xidsqn`.

Find the blocker:

```sql
SELECT t.addr, t.xidusn, t.xidslot, t.xidsqn, s.sid, s.username
FROM   v$transaction t JOIN v$session s ON s.taddr = t.addr
WHERE  t.xidusn = TRUNC(&p2 / POWER(2,16))
   AND t.xidslot = MOD(&p2, POWER(2,16))
   AND t.xidsqn = &p3;
```

### `latch: cache buffers chains`

- `P1RAW` = latch address (join to `V$LATCH_CHILDREN.ADDR`).
- `P2` = latch#.
- `P3` = tries (number of retries).

Find contended buffer:

```sql
SELECT b.file#, b.dbablk, b.class, b.tch, o.owner, o.object_name
FROM   x$bh b LEFT JOIN dba_objects o ON o.data_object_id = b.obj
WHERE  b.hladdr = HEXTORAW('&p1raw')
ORDER  BY b.tch DESC
FETCH FIRST 20 ROWS ONLY;
```

### `library cache: mutex X`

- `P1RAW` = hash / bucket.
- `P2RAW` = KGL object address.
- `P3RAW` = mutex value & operation.

Find contended object:

```sql
SELECT kglnaown, kglnaobj, kglobtyd
FROM   x$kglob
WHERE  kglhdadr = HEXTORAW('&p2raw');
```

## Wait Class Categories

12 wait classes group events by cause:

| Class            | Events (examples)                                                                  | Root causes typically         |
| ---------------- | ---------------------------------------------------------------------------------- | ----------------------------- |
| `User I/O`       | `db file sequential read`, `db file scattered read`, `direct path read`            | Storage; cache size.          |
| `System I/O`     | `db file parallel write`, `log file parallel write`, `control file parallel write` | Background IO.                |
| `Concurrency`    | `latch: *`, `library cache: mutex *`, `cursor: pin *`, `buffer busy waits`         | Shared-structure contention.  |
| `Application`    | `enq: TX row lock contention`, `enq: TM contention`, `SQL*Net break/reset`         | Application locking behavior. |
| `Commit`         | `log file sync`                                                                    | LGWR / redo storage.          |
| `Cluster`        | `gc *` events                                                                      | RAC cache fusion.             |
| `Network`        | `SQL*Net message from client`, `SQL*Net message to client`                         | Client / network.             |
| `Administrative` | `Rman backup`, `wait for a undo record`                                            | Ops.                          |
| `Configuration`  | `undo segment extension`, `log file switch (checkpoint incomplete)`                | Sizing.                       |
| `Queueing`       | `resmgr:cpu quantum`                                                               | Resource Manager.             |
| `Idle`           | `SQL*Net message from client`, `pmon timer`, `rdbms ipc message`                   | Not an issue.                 |
| `Other`          | Miscellaneous.                                                                     | Case by case.                 |

`Idle` events are filtered out of "top events" reports because they represent sessions waiting for external input, not database work.

## Session Wait Recording

`V$SESSION` shows current wait state — updated as sessions enter/exit `kslwtbctx`/`kslwtectx`:

| Column            | Meaning                                                                                                                                                                           |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `STATE`           | `WAITING` (currently sleeping), `WAITED KNOWN TIME` (last wait completed with time known), `WAITED SHORT TIME` (finished before micros could be captured), `WAITED UNKNOWN TIME`. |
| `EVENT`           | Current or last event name.                                                                                                                                                       |
| `SECONDS_IN_WAIT` | Elapsed seconds in current wait (only if STATE=WAITING).                                                                                                                          |
| `WAIT_TIME`       | Duration of last completed wait in centiseconds.                                                                                                                                  |
| `WAIT_TIME_MICRO` | Same in microseconds.                                                                                                                                                             |
| `P1/P2/P3`        | Parameter values.                                                                                                                                                                 |

## System-Wide Counters

`V$SYSTEM_EVENT` accumulates event totals since instance start:

```sql
SELECT event, wait_class, total_waits,
       ROUND(time_waited/100, 1) secs,
       ROUND(average_wait, 3) cs
FROM   v$system_event
WHERE  wait_class <> 'Idle'
ORDER  BY time_waited DESC
FETCH  FIRST 20 ROWS ONLY;
```

`TIME_WAITED` in centiseconds; `TIME_WAITED_MICRO` in microseconds for precision.

## Histograms

`V$EVENT_HISTOGRAM` and `V$EVENT_HISTOGRAM_MICRO` bucket wait durations:

```sql
SELECT   event, wait_time_milli, wait_count
FROM     v$event_histogram
WHERE    event = 'db file sequential read'
ORDER BY wait_time_milli;
```

Buckets (ms): `1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024, 2048, 4096, 8192, 16384, 32768, 65536, ...`.

Micro histogram (12c+):

```sql
SELECT   event, wait_time_micro, wait_count
FROM     v$event_histogram_micro
WHERE    event = 'log file sync'
ORDER BY wait_time_micro;
```

Buckets (µs): 1, 2, 4, ... doubling.

Read as: for `log file sync`, how many waits fell in [1024, 2048) µs, [2048, 4096) µs, etc. Skew right = long tail = investigate outliers.

## Timing Sources & Precision

Wait timing uses `V$LATCH.NAME='timer service'` under the hood — hardware clock reads on entry and exit of the wait context. Precision:

- Microsecond resolution on modern platforms.
- `TIMED_STATISTICS = TRUE` (default) enables collection.
- `STATISTICS_LEVEL = TYPICAL` (default) includes waits + timing.

`STATISTICS_LEVEL = BASIC` disables all — never in production.

## ASH — The Sampler

MMNL samples `V$SESSION` every second (`_ash_sampling_interval=1s`) and stores rows in the in-memory ring (`V$ACTIVE_SESSION_HISTORY`). Only **active** sessions (`state <> 'IDLE'`) are sampled. Every 10th sample is persisted to `DBA_HIST_ACTIVE_SESS_HISTORY`.

ASH captures:

- `EVENT`, `WAIT_CLASS`, `SESSION_STATE`.
- `P1, P2, P3`.
- `SQL_ID`, `SQL_PLAN_HASH_VALUE`, `SQL_PLAN_LINE_ID`.
- `BLOCKING_SESSION`.
- `CURRENT_OBJ#, CURRENT_FILE#, CURRENT_BLOCK#`.
- `PGA_ALLOCATED, TEMP_SPACE_ALLOCATED`.

Because of 1-second sampling, ASH represents time — 1 sample ≈ 1 second of DB time.

## Enabling / Disabling Instrumentation

Never disable at production. If troubleshooting:

```sql
-- Global disable (bad idea)
ALTER SYSTEM SET timed_statistics = FALSE SCOPE=BOTH;

-- Trace an event to see it inline
ALTER SYSTEM SET EVENTS '10046 trace name context forever, level 12';
```

## Wait Event → SQL Path

The key move in performance diagnosis: **from a wait event to the offending SQL**.

Live sessions:

```sql
SELECT s.sid, s.event, s.sql_id, s.p1, s.p2, s.p3, s.seconds_in_wait
FROM   v$session s
WHERE  s.event = '&event_name'
   AND s.status = 'ACTIVE';
```

Historical (ASH):

```sql
SELECT event, sql_id, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time > SYSDATE - 15/1440
   AND event = '&event_name'
GROUP  BY event, sql_id
ORDER  BY 3 DESC
FETCH  FIRST 10 ROWS ONLY;
```

## Custom Wait Event Analysis

Group event by class + object for a period:

```sql
SELECT   wait_class, event, current_obj#,
         COUNT(*) samples,
         ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) pct
FROM     v$active_session_history
WHERE    sample_time BETWEEN &t1 AND &t2
   AND   session_state = 'WAITING'
GROUP BY wait_class, event, current_obj#
ORDER BY 4 DESC
FETCH FIRST 20 ROWS ONLY;
```

## Wait Event Names — Reserved Vocabulary

Well over 1500 events in 19c. Look up any name's parameters:

```sql
SELECT name, wait_class, parameter1, parameter2, parameter3
FROM   v$event_name
WHERE  name LIKE '%mutex%'
ORDER  BY name;
```

Deprecated / renamed events:

- Pre-11g `enqueue` → 11g+ `enq: TX - row lock contention`, `enq: TM - contention`, etc.
- Pre-11g `library cache pin` → 11g+ `library cache: mutex X` for most cursor-pin operations.

## Interview Framing

> "Where does `V$SESSION.EVENT` come from?"

The KSLE layer instruments every internal `sleep` call in Oracle with wait context enter/exit. When a session enters the wait, the event name and parameters are stored on the session; on wake-up, the elapsed time is bucketed into `V$SESSION_EVENT`, `V$SYSTEM_EVENT`, and histograms.

> "You see a top wait event — what next?"

Look up P1/P2/P3 in `V$EVENT_NAME.PARAMETER1/2/3`. Use them to identify the specific object/block/latch/cursor. Then investigate why that object is hot.

> "Difference between `WAIT_TIME` and `TIME_WAITED`?"

`WAIT_TIME` (V$SESSION) = duration of the last completed wait, per session. `TIME_WAITED` (V$SYSTEM_EVENT) = cumulative sum across all sessions since startup.

## Related

- [Wait Events chapter](../25-reference/wait-events/commit.md), [Concurrency](../25-reference/wait-events/concurrency.md), [Network](../25-reference/wait-events/network.md), [System I/O](../25-reference/wait-events/system-io.md), [User I/O](../25-reference/wait-events/user-io.md).
- [ASH](../12-performance-tuning/ash.md).
- [AWR](../12-performance-tuning/awr.md).
- [V$SESSION](../25-reference/v-views/v-session.md).
- [V$SESSION_WAIT](../25-reference/v-views/v-session-wait.md).
- [V$SYSTEM_EVENT](../25-reference/v-views/v-system-event.md).
- [Fixed Tables (X$)](fixed-tables-x-dollar.md).
- [Latches vs Mutexes](latches-vs-mutexes.md).
