# TX Locks

## Overview

**TX (Transaction) locks** are the row-level locks Oracle uses to serialize concurrent modifications. Every DML transaction acquires **one TX enqueue** in exclusive mode. Sessions modifying the _same_ rows queue on that TX enqueue.

Key insight: TX is **not** a per-row lock. Oracle records row locks inline in the block header (ITL slot) — the TX enqueue exists to represent "this transaction is alive" so that other waiters can queue on it and be notified on commit/rollback.

## The ITL and TX Interaction

Every block modified by a transaction has an **ITL (Interested Transaction List)** slot pointing to the transaction's undo. The slot has a **lock byte** per row in the block, marking rows this txn has locked.

When session B tries to update a row session A has locked:

1. B reads the block, sees A's ITL slot with the row lock.
2. B enqueues on A's TX in mode X.
3. B waits with `enq: TX - row lock contention`.
4. When A commits/rolls back, B is posted and proceeds.

Row locks are **unlimited** because they cost nothing in memory — just an ITL slot marker.

## TX Wait Sub-types

`V$LOCK` shows TX with different `REQUEST` values indicating what's being waited for:

- **`enq: TX - row lock contention`** — waiting on a row locked by another transaction.
- **`enq: TX - index contention`** — waiting on an index leaf block (right-most on ascending PK).
- **`enq: TX - allocate ITL entry`** — the block has no free ITL slot; waiting for one to free.
- **`enq: TX - contention`** — generic; typically PK/UK violations resolving.

## Diagnostic Queries

### Who's blocking whom?

```sql
SELECT s.sid, s.serial#, s.username, s.status,
       s.blocking_session AS blocker_sid,
       s.event, s.seconds_in_wait, s.sql_id
FROM   v$session s
WHERE  s.blocking_session IS NOT NULL
ORDER  BY s.seconds_in_wait DESC;
```

### Full blocking chain

```sql
SELECT LPAD(' ', LEVEL*2) || s.sid AS chain,
       s.username, s.status, s.event, s.seconds_in_wait, s.sql_id
FROM   v$session s
START WITH s.blocking_session IS NULL
       AND s.sid IN (SELECT blocking_session FROM v$session WHERE blocking_session IS NOT NULL)
CONNECT BY PRIOR s.sid = s.blocking_session;
```

### TX enqueue detail

```sql
SELECT l.sid, l.type, l.lmode, l.request, l.id1, l.id2,
       s.username, s.sql_id, s.event
FROM   v$lock l JOIN v$session s ON l.sid = s.sid
WHERE  l.type = 'TX'
ORDER  BY l.sid;
```

- `lmode=6` and `request=0` — holder.
- `lmode=0` and `request=6` — waiter.

### Which row (via ROWID from p1/p2/p3)

```sql
-- For enq: TX - row lock contention, p3 encodes the block+row
-- Cleaner: find object being waited on via ASH
SELECT ash.event, ash.blocking_session,
       o.owner || '.' || o.object_name AS object
FROM   v$active_session_history ash
       JOIN dba_objects o ON o.object_id = ash.current_obj#
WHERE  ash.event = 'enq: TX - row lock contention'
   AND ash.sample_time > SYSDATE - 1/24
ORDER  BY ash.sample_time DESC
FETCH FIRST 10 ROWS ONLY;
```

## `enq: TX - allocate ITL entry`

Cause: block header has no free ITL slot AND no free block space to allocate one. Every existing ITL slot represents an active transaction on this block.

Fix:

```sql
-- Increase INITRANS for the segment
ALTER TABLE hr.orders MOVE INITRANS 10;

-- Rebuild indexes similarly
ALTER INDEX hr.orders_pk REBUILD INITRANS 10;
```

`INITRANS` = initial ITL slots per block. Default 1 for tables, 2 for indexes — too low for hot tables. Set to 10+ for tables with heavy concurrent DML.

## `enq: TX - index contention`

Hot index leaf — every INSERT with a sequence-generated key hits the same right-most leaf. Multiple sessions inserting concurrently collide on that leaf's ITL / structural lock.

Fixes:

- **Sequence CACHE 1000+ NOORDER** — reduces coordination.
- **Reverse-key index** — spreads inserts across index (loses range scans).
- **Hash-partitioned index** — spreads leaves across partitions (partitioning license).

## Common Issues

- **Session forever `enq: TX - row lock contention`** — Blocker session forgot to commit; likely stuck in application think-time. Kill blocker or resolve app-side.
- **Repeating deadlocks in same code path** — See [Deadlocks](deadlocks.md).
- **TX contention on tiny table** — Check for `SELECT ... FOR UPDATE` grabbing whole result set.

## Best Practices

1. **Commit or rollback promptly.** No "open transactions across user think-time."
2. Set `INITRANS 10+` on high-concurrency tables.
3. `enq: TX - index contention` → sequence CACHE + reverse-key or hash partitioning.
4. Use `SELECT ... FOR UPDATE NOWAIT` or `SKIP LOCKED` in queue-processing code.
5. Session timeout (Resource Manager `SWITCH_TIME`) to reap idle-in-transaction sessions.
6. Monitor `V$SESSION.BLOCKING_SESSION` — alert if seconds_in_wait > 60 for critical schemas.

## Interview Questions

1. **Q:** Where are Oracle row locks stored?
   **A:** In the block header's ITL slot — inline with the block, not in a memory table.

2. **Q:** Why are row locks unlimited?
   **A:** Cost nothing — just a mark on the ITL slot in the block.

3. **Q:** `enq: TX - row lock contention`?
   **A:** Session wants a row locked by another transaction; waits until commit/rollback.

4. **Q:** `enq: TX - allocate ITL entry`?
   **A:** Block header full of ITL slots. Increase `INITRANS`.

5. **Q:** Fix hot right-most index leaf?
   **A:** Increase sequence CACHE, reverse-key or hash-partitioned index.

6. **Q:** Difference between TM and TX?
   **A:** TM = table DML lock. TX = transaction / row lock enqueue.

## References

- Oracle Database Concepts 19c — Data Concurrency
- MOS Doc ID 62354.1 — TX Enqueue Deep Dive
- MOS Doc ID 15476.1 — TX/TM Diagnosis
