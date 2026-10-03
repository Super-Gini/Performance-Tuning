# TM Locks

## Overview

**TM (Table Modification / DML) locks** are table-level locks acquired for the duration of any DML on the table. They coexist happily with each other in most modes — the point of TM is to protect against **structural changes** (DDL) while DML is in flight, not to serialize DML.

You'll see them in `V$LOCK` (as `TYPE = 'TM'`) and in wait events (`enq: TM - contention`).

## Lock Modes

| LMode | Name                           | Semantics                                         |
| ----- | ------------------------------ | ------------------------------------------------- |
| 2     | RS (Row Share, SS)             | Held by `SELECT ... FOR UPDATE`                   |
| 3     | RX (Row Exclusive, SX)         | Held by `INSERT/UPDATE/DELETE/MERGE`              |
| 4     | S (Share)                      | Held by `LOCK TABLE ... IN SHARE MODE`            |
| 5     | SRX (Share Row Exclusive, SSX) | `LOCK TABLE ... IN SHARE ROW EXCLUSIVE`           |
| 6     | X (Exclusive)                  | `LOCK TABLE ... IN EXCLUSIVE MODE`, or during DDL |

**RX + RX** are compatible — most DML on the same table doesn't collide at the TM level. Row-level collisions manifest as **TX** enqueue.

## Compatibility Matrix

|     | RS(2) | RX(3) | S(4) | SRX(5) | X(6) |
| --- | :---: | :---: | :--: | :----: | :--: |
| RS  |  ✅   |  ✅   |  ✅  |   ✅   |  ❌  |
| RX  |  ✅   |  ✅   |  ❌  |   ❌   |  ❌  |
| S   |  ✅   |  ❌   |  ✅  |   ❌   |  ❌  |
| SRX |  ✅   |  ❌   |  ❌  |   ❌   |  ❌  |
| X   |  ❌   |  ❌   |  ❌  |   ❌   |  ❌  |

## Sources of `enq: TM - contention`

### 1. DDL During DML

Any DDL on the table needs mode 6 (X). Any active DML holds RX. DDL waits.

### 2. Unindexed Foreign Keys — The Classic

Session A: `UPDATE parent SET id = 100 WHERE id = 99` — acquires RX on parent, then tries to lock **the entire child table in SRX** to check FK integrity. If child.parent_id has no index, the lock is table-level. Session B trying to DML the child now waits.

**Fix**: Index every FK column. Almost always a good idea anyway.

### 3. `SELECT ... FOR UPDATE` on very hot rows

Locks rows and holds RS on the table — doesn't collide with normal DML but does with DDL.

### 4. Long DDL

`CREATE INDEX ... ONLINE` still takes brief exclusive locks at start / end. Non-online: full duration.

### 5. Direct-path INSERT

`INSERT /*+ APPEND */` takes a mode 6 X on the table — blocks all other DML.

## Diagnostic Queries

```sql
-- Sessions holding TM locks
SELECT l.sid, l.type, l.lmode, l.request,
       o.owner, o.object_name, s.username, s.machine, s.sql_id
FROM   v$lock l
       JOIN dba_objects o ON o.object_id = l.id1
       JOIN v$session s ON s.sid = l.sid
WHERE  l.type = 'TM'
ORDER  BY l.sid;

-- TM blockers
SELECT holder.sid AS blocker_sid, waiter.sid AS waiter_sid,
       waiter.request AS waiter_lmode,
       o.owner || '.' || o.object_name AS object
FROM   v$lock holder
       JOIN v$lock waiter ON waiter.type = 'TM' AND waiter.id1 = holder.id1
                          AND waiter.lmode = 0
       JOIN dba_objects o ON o.object_id = holder.id1
WHERE  holder.type = 'TM' AND holder.lmode > 0;

-- Unindexed FK check
SELECT c.owner, c.constraint_name, c.table_name,
       LISTAGG(cc.column_name, ',') WITHIN GROUP (ORDER BY cc.position) AS fk_cols
FROM   dba_constraints c
       JOIN dba_cons_columns cc ON cc.constraint_name = c.constraint_name
                               AND cc.owner = c.owner
WHERE  c.constraint_type = 'R'
   AND NOT EXISTS (
     SELECT 1 FROM dba_ind_columns i
     WHERE i.table_owner = c.owner AND i.table_name = c.table_name
       AND i.column_name = cc.column_name AND i.column_position = cc.position)
GROUP  BY c.owner, c.constraint_name, c.table_name;
```

## Common Issues

- **DDL blocks DML** — Move DDL to a maintenance window, or use `ONLINE` variants where available.
- **Foreign key TM waits** — Index the FK columns.
- **Direct-path INSERT blocking OLTP** — Coordinate with app; batches run off-hours.
- **`ORA-00054: resource busy` when acquiring TM in NOWAIT** — Session waiting on a lock; retry later.

## Best Practices

1. **Index every foreign key column** unless you have a well-understood reason not to.
2. Do heavy DDL during **maintenance windows**.
3. Use `ONLINE` DDL where available (`CREATE INDEX ONLINE`, `ALTER TABLE MOVE ONLINE`).
4. Direct-path load: schedule when other DML is quiet; use partitioning to isolate.
5. `LOCK TABLE ... IN EXCLUSIVE MODE` is a hammer — use rarely.

## Interview Questions

1. **Q:** What is a TM lock?
   **A:** Table-level DML lock — protects against structural changes during DML.

2. **Q:** RX vs RX compatible?
   **A:** Yes — most DML on same table is fine at TM level.

3. **Q:** Why do unindexed FKs cause TM contention?
   **A:** Parent DML acquires SRX on child during FK check; if not indexed, this is a table-level lock blocking other DML.

4. **Q:** Direct-path INSERT lock mode?
   **A:** Exclusive (mode 6) — blocks other DML on the table.

5. **Q:** How to see TM locks?
   **A:** `SELECT * FROM v$lock WHERE type = 'TM'`.

## References

- Oracle Database Concepts 19c — Data Concurrency and Consistency
- MOS Doc ID 15476.1 — TM Contention
- MOS Doc ID 32852.1 — FK Locking
