# Concurrency Wait Events

## Overview

The **Concurrency** wait class covers waits caused by contention on shared in-memory structures — latches, mutexes, library cache pins/locks, cursor cache pins, buffer busy waits. High concurrency waits usually mean application design or shared-pool sizing problem, not storage.

## Key Events

### `latch: shared pool`

Contention on the shared pool latch — usually parsing hard.

**Fix**:

- Fix bind variable use (`cursor_sharing=EXACT` + bind).
- Grow shared pool.
- Investigate soft-parse rate; often driven by non-shared SQL.

### `latch: cache buffers chains` (CBC)

Contention on the buffer cache hash chain latches. Hot block.

**Fix**:

- Identify the hot block via `V$LATCH_MISSES.parent_name` and `V$BH`.
- Application: hash-partition the hot table, add sequences with `NOORDER CACHE`, batch DML.

```sql
-- Hot block identification
SELECT   obj, dbarfil, dbablk, COUNT(*)
FROM     v$bh
WHERE    tch > 10
GROUP BY obj, dbarfil, dbablk
ORDER BY 4 DESC
FETCH FIRST 10 ROWS ONLY;
```

### `library cache: mutex X`

Mutex contention on library cache objects — very common in 11g+. Same-SQL executed on many sessions concurrently.

**Fix**:

- Use bind variables.
- `_kgl_hot_object_copies` (MOS Doc ID 9040969.8).
- Redesign hot cursor.

### `library cache lock` / `library cache pin`

DDL vs DML contention. Someone is dropping/altering an object while another session is compiling against it.

**Fix**:

- Schedule DDL in maintenance windows.
- Kill blocking DDL session.

### `cursor: pin S wait on X`

Session S wants shared, X holds exclusive. Same-SQL parse contention.

**Fix**: Same as `library cache: mutex X`.

### `buffer busy waits`

Waiting for a buffer another session is loading. Segment header contention.

**Parameters**:

- `p1` — file #.
- `p2` — block #.
- `p3` — reason class.

**Reason codes** (`p3`):

- `1xx` — segment header.
- `2xx` — data block.
- `3xx` — undo header.

**Fix**:

- ASSM tablespace (usually already yes).
- Larger `INITRANS` on hot tables.
- Freelist groups on segments (legacy MSSM).

### `enq: TX - row lock contention`

Session waiting on another session's row lock. Application concurrency.

**Fix**: Application redesign (shorter transactions, ordered updates).

### `enq: TM - contention`

Table-level lock. Missing FK indexes are the classic cause.

```sql
-- Missing FK indexes
SELECT * FROM DBA_CONSTRAINTS c
WHERE  c.constraint_type = 'R'
   AND NOT EXISTS (SELECT 1 FROM DBA_IND_COLUMNS ic
                   WHERE ic.table_owner = c.owner
                     AND ic.table_name = c.table_name
                     AND ic.column_position = 1
                     AND ic.column_name = (SELECT column_name FROM DBA_CONS_COLUMNS
                                           WHERE owner = c.owner AND constraint_name = c.constraint_name
                                             AND ROWNUM = 1));
```

## Diagnostic Query

```sql
SELECT   event, wait_class, total_waits,
         ROUND(time_waited/100,1) secs,
         ROUND(average_wait,3) cs
FROM     v$system_event
WHERE    wait_class = 'Concurrency'
   AND   time_waited > 0
ORDER BY time_waited DESC
FETCH FIRST 15 ROWS ONLY;
```

## References

- Oracle Database Performance Tuning Guide 19c
- MOS Doc ID 62354.1 — Enqueue reference
- [Latch Contention](../../12-performance-tuning/latch-contention.md)
- [Mutex Contention](../../12-performance-tuning/mutex-contention.md)
