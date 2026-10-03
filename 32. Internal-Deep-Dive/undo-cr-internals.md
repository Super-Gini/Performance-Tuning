# Undo & Consistent Read Internals

## Overview

Undo is what makes rollback, read consistency, and Flashback Query possible. It's stored in **undo segments** in an **undo tablespace**, addressed through the **Transaction Table** in each undo segment header, and referenced from data-block **ITL entries** (see [Block Format & ITL](block-format-itl.md)).

This page walks how a transaction generates undo, how CR blocks are constructed from that undo, how commit cleanout happens (or gets delayed), and why ORA-01555 appears.

## Undo Segment Layout

An **undo segment** is a special segment in `SYSTEM.UNDO$` allocated for transaction workspace. Contains:

- **Segment header block** — the special block with the **Transaction Table** (TX table).
- **Undo blocks** — buffer for undo records.

Every undo segment header has:

- Up to ~48 **TX table slots** — one per active transaction using this segment.
- Each slot: TX ID (XID = `usn.slot.wrap`), state, undo block chain head, commit SCN (if committed).

Number of undo segments = auto-sized by `MMON` based on concurrency, up to ~1024. Each active transaction is assigned one slot in one undo segment. `_undo_autotune=TRUE` (default) manages this.

```sql
SELECT * FROM v$rollname WHERE name LIKE '_SYSSMU%';

SELECT   segment_name, tablespace_name, status,
         initial_extent, next_extent
FROM     dba_rollback_segs;
```

## Transaction ID (XID)

A transaction is identified by:

- **USN** (Undo Segment Number, `xidusn`) — which undo segment.
- **Slot** (`xidslot`) — which TX table slot within.
- **Wrap** (`xidsqn`) — reuse counter; increments when the slot is recycled.

Concat: `USN.Slot.Wrap` — e.g., `12.34.567`. Visible in `V$TRANSACTION`.

```sql
SELECT   addr, xidusn, xidslot, xidsqn,
         status, start_scn, used_ublk, used_urec
FROM     v$transaction;
```

`used_ublk` = undo blocks; `used_urec` = undo records (row-level).

## Undo Generation — Per DML

For an UPDATE of one row:

1. Session acquires TX slot (if new TX).
2. Session locates the target block, obtains an ITL entry (see [Block Format & ITL](block-format-itl.md)).
3. Session pushes the row's **before image** to the current undo block for this TX.
4. Row is modified in place.
5. Redo record captures both changes (undo write + data write) as change vectors.
6. On COMMIT: TX table slot state → committed, commit SCN stamped.

Two orthogonal writes: undo block gets old-row-image, data block gets new-row-image. Both durable via redo.

## Reading Undo — Consistent Read Path

When session S1 reads a block whose SCN > S1's query SCN:

1. Read block from buffer cache (X$BH.SCN > S1.query_scn).
2. **Clone** the buffer — copy body to a new buffer, mark `STATE='cr'`.
3. Walk the block's ITL:
   - For each entry with SCN > S1.query_scn: this transaction modified the block after S1 started.
   - Fetch the undo chain from that TX's undo segment.
   - Apply undo records in reverse: reverse the changes to bring block back to pre-TX state.
4. If the ITL entry's XID is still **active** (uncommitted), fetch the transaction's undo, apply.
5. Repeat until block SCN ≤ S1.query_scn.
6. Read rows from CR block, return.

The CR clone stays in the cache until aged out via LRU. Reused if another session at similar SCN needs the same block.

## Delayed Block Cleanout

When a transaction commits, its commit SCN needs to appear in **every ITL entry** of blocks it modified. But writing that many blocks would be expensive. So Oracle does **fast commit**:

- Only mark commit SCN in the TX table slot immediately.
- Individual block ITL entries retain the old (in-progress) TX state.

Later, when a session reads such a block:

1. Sees TX status "active" in ITL.
2. Looks up XID in TX table.
3. Finds TX is committed with SCN X.
4. **Cleans out** the ITL: replaces "active" marker with commit SCN X.
5. This is a write to the block — generates redo + marks buffer dirty.

Result: **a SELECT can generate redo and dirty buffers** (surprising to newcomers).

Statistics:

```sql
SELECT name, value FROM v$sysstat
WHERE  name IN ('commit cleanouts',                           -- fast-commit ITL entries
                'commit cleanouts successfully completed',    -- succeeded fast-commit
                'cleanouts only - consistent read gets',      -- CR reads with cleanout
                'cleanouts and rollbacks - consistent read gets'); -- CR that needed rollback
```

Forcing cleanout after a big load:

```sql
SELECT /*+ FULL(t) */ COUNT(*) FROM app.big_table t;
```

Reads every block; cleans out along the way. Alternative: gather stats on the segment (does similar full read).

## Snapshot Too Old — ORA-01555

Fires when Oracle needs undo for CR construction but that undo has been **overwritten**.

Anatomy:

1. Session S1 starts long query at SCN X.
2. Concurrent DML by other sessions generates undo, cycles through undo segments.
3. When their transactions commit and their retention expires, their undo blocks become **expired** — reusable.
4. New DML overwrites those expired undo blocks.
5. S1 needs to CR-clone a block whose ITL points to overwritten undo.
6. Oracle can't reconstruct the pre-image → ORA-01555.

Root causes:

- Undo tablespace too small.
- `UNDO_RETENTION` too low.
- `UNDO_RETENTION` not guaranteed (no `RETENTION GUARANTEE`).
- Very long query.
- Fetch across commits (application anti-pattern — session does DML in the middle of its own long open cursor).

Detection:

```sql
-- Historical
SELECT begin_time, ssolderrcnt ora_1555, maxquerylen longest_secs,
       maxquerysqlid, tuned_undoretention tuned_ret_s
FROM   v$undostat
WHERE  ssolderrcnt > 0
ORDER  BY begin_time DESC;
```

## Undo Retention — Auto-Tuning

`UNDO_RETENTION` (seconds) is a **target**, not a hard limit. `MMON` auto-tunes actual retention based on:

- Longest active query.
- Undo tablespace headroom.
- `RETENTION GUARANTEE` on the tablespace.

Current tuned retention:

```sql
SELECT MAX(tuned_undoretention) tuned_seconds,
       MAX(maxquerylen)         longest_query_seconds
FROM   v$undostat
WHERE  begin_time > SYSDATE - 1;
```

Guarantee: if you set `RETENTION GUARANTEE`, undo is never overwritten until retention expires — but this can cause `ORA-30036: unable to extend segment in undo tablespace`. Trade-off between ORA-01555 (reads fail) and ORA-30036 (writes fail).

## Undo States

Every undo block has a **state**:

- **Active** — belongs to an uncommitted transaction. Cannot be overwritten.
- **Unexpired** — committed, but within retention window. Preferred not to overwrite (used for CR).
- **Expired** — committed and retention passed. Reusable at will.

Sizing:

```sql
SELECT   status, ROUND(SUM(bytes)/1024/1024, 2) mb
FROM     dba_undo_extents
GROUP BY status;
```

Healthy: mix of ACTIVE (small), UNEXPIRED (largest), EXPIRED (some headroom).

## Sizing Undo Tablespace

```
required_undo_size = undo_generation_rate * undo_retention_target + headroom
```

Compute:

```sql
SELECT   ROUND(MAX(undoblks)/10*(SELECT block_size FROM dba_tablespaces
                   WHERE tablespace_name = (SELECT value FROM v$parameter WHERE name='undo_tablespace'))
               *3600/1024/1024/1024, 2) required_gb_per_hour_retention,
         MAX(maxquerylen) longest_query_s,
         MAX(tuned_undoretention) tuned_ret_s
FROM     v$undostat
WHERE    begin_time > SYSDATE - 7;
```

For 1-hour retention target: multiply `required_gb_per_hour_retention` × 1 = min size. Add safety factor.

## Fetch Across Commits — The Classic Anti-Pattern

```plsql
-- BAD: cursor stays open across commits
FOR r IN (SELECT rowid FROM big_table) LOOP
   UPDATE big_table SET x = f(x) WHERE ROWID = r.ROWID;
   COMMIT;    -- commits every row
END LOOP;
```

Problem: the outer SELECT captures a snapshot SCN. Each COMMIT advances active SCN. Undo for the old snapshot may be reused after retention. Eventually the outer SELECT tries to CR-construct a block whose undo is gone → ORA-01555.

Fix — batch commits or use `FORALL`:

```plsql
DECLARE
  TYPE t_rowids IS TABLE OF ROWID;
  v_rowids t_rowids;
BEGIN
  SELECT ROWID BULK COLLECT INTO v_rowids FROM big_table;
  FORALL i IN 1..v_rowids.COUNT
    UPDATE big_table SET x = f(x) WHERE ROWID = v_rowids(i);
  COMMIT;
END;
```

## Temp Undo (12c+)

`ALTER SESSION SET TEMP_UNDO_ENABLED = TRUE` — undo for global temporary tables goes to TEMP tablespace instead of UNDO. Reduces redo generation (TEMP doesn't need redo) and undo pressure.

Use for ETL sessions heavily using GTTs.

## Diagnostic Queries

### Transaction footprint

```sql
SELECT   t.addr, s.sid, s.username, s.sql_id,
         t.xidusn, t.xidslot, t.xidsqn,
         t.status, t.used_ublk, t.used_urec,
         ROUND(t.log_io/1024/1024,2) redo_mb,
         t.start_scn
FROM     v$transaction t JOIN v$session s ON s.taddr = t.addr
ORDER BY t.used_urec DESC
FETCH FIRST 20 ROWS ONLY;
```

### Undo pressure right now

```sql
SELECT   status, COUNT(*) extents,
         ROUND(SUM(bytes)/1024/1024, 2) mb
FROM     dba_undo_extents
GROUP BY status;
```

### Sessions holding long-running transactions

```sql
SELECT   s.sid, s.username, s.sql_id, t.start_scn,
         (SYSDATE - LOGON_TIME)*86400 session_age_s,
         t.used_urec undo_recs
FROM     v$transaction t JOIN v$session s ON s.taddr = t.addr
WHERE    t.start_scn < (SELECT current_scn FROM v$database) - 1000000
ORDER BY t.start_scn;
```

### Undo Retention Advisor

```sql
SELECT * FROM V$UNDOSTAT_ADVISOR_HISTORY;   -- 19c
```

## Interview Framing

> "Why can a SELECT generate redo?"

Delayed block cleanout: reader touches a block whose ITL entries were left in "active" state after a fast-commit. Reader walks TX table, finds the TX committed, cleans out the ITL (writes commit SCN), marks buffer dirty, generates a small redo record.

> "What causes ORA-01555 and how do you fix it?"

Long-running query needs pre-image undo that has been overwritten (undo cycled out under pressure). Fix: bigger undo tablespace, higher `UNDO_RETENTION`, `RETENTION GUARANTEE`, fix fetch-across-commits, or shorten the query.

> "What is RETENTION GUARANTEE?"

Tablespace attribute: unexpired undo is not overwritten even under write pressure. Prevents ORA-01555 at the cost of possible ORA-30036 (undo full → new writes fail).

## Related

- [ORA-01555](../26-errors/ora-01555.md).
- [Consistent Read](../05-undo/consistent-read.md).
- [Undo Architecture](../05-undo/undo-architecture.md).
- [Undo Management](../05-undo/undo-management.md).
- [Undo Retention](../05-undo/undo-retention.md).
- [Block Format & ITL](block-format-itl.md).
- [Buffer Cache Internals](buffer-cache-internals.md).
- [Redo Internals](redo-internals.md).
- [SCN](scn.md).
