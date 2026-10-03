# Block Format & ITL

## Overview

Every Oracle data block is an 8 KB (default) unit with a fixed on-disk layout: block header, transaction (ITL) list, table/row directories, free space, and row data. Understanding this layout explains `INITRANS`, `MAXTRANS`, `PCTFREE`, ITL waits, row chaining, row migration, and how MVCC actually references old row images.

## The Data Block Layout

```
+-------------------------------------------------+  offset 0
| Common and Variable Header                       |  ~57 bytes
+-------------------------------------------------+
| Interested Transaction List (ITL)                |  N × 24 bytes
+-------------------------------------------------+
| Table Directory                                  |  var
+-------------------------------------------------+
| Row Directory                                    |  N × 2 bytes
+-------------------------------------------------+
|                                                 |
| Free Space                                       |  (PCTFREE)
|                                                 |
+-------------------------------------------------+
| Row Data (grows upward)                          |
+-------------------------------------------------+  offset 8191
```

Row data grows **upward** from the bottom; metadata grows **downward** from the top. Free space is sandwiched in the middle.

## Common Header

Every block, regardless of type (data / index / undo / etc.):

- **Block type** (data, index, undo, segment header, ...).
- **DBA** — File# + Block#, self-reference.
- **SCN of last change**.
- **CSC** (Cleanout SCN) — used for delayed block cleanout.
- **Checksum** (`DB_BLOCK_CHECKSUM=TYPICAL/FULL`).
- **Flags**.

Dump it (dev/lab only):

```sql
ALTER SESSION SET TRACEFILE_IDENTIFIER = 'BLOCKDUMP';
ALTER SYSTEM DUMP DATAFILE 17 BLOCK 1234;
```

Trace file appears in ADR; contents include header + ITL + rows.

## Interested Transaction List (ITL)

The **ITL** is a per-block table of active/recent transactions that touched the block. Each ITL entry = **24 bytes** — one per concurrent modifying transaction.

Structure of one entry:

- **XID** (Transaction ID: USN + Slot + Wrap) — points into a specific undo segment TX table.
- **UBA** (Undo Byte Address) — pointer to the undo record chain for this TX's changes on this block.
- **Flag** — state (uncommitted, cleaned-out, delete lock, etc.).
- **Lock count** — number of rows locked by this TX in this block.
- **SCN/Fsc** — Commit SCN (once cleaned out) or "Fast Commit" flag.

Initial number of ITL slots = **`INITRANS`** (default: 1 for tables, 2 for indexes).

Max slots: **`MAXTRANS`** — historically bounded, in 10g+ effectively bounded only by `PCTFREE` (uses free space to grow ITL).

## ITL Growth

When a new transaction needs an ITL entry:

1. Check ITL for a free slot (committed with expired retention, or unused).
2. If free slot: use it.
3. Else: try to grow ITL by consuming block's free space (if `MAXTRANS` allows and free space available).
4. Else: wait — **`enq: TX - allocate ITL entry`**.

`enq: TX - allocate ITL entry` = ITL contention. High-concurrency tables with `INITRANS=1` will exhibit this.

Fix: raise `INITRANS` (requires table rebuild — new blocks inherit).

## Row Format

Each row starts with a **row header**:

- **Flag byte** (row type: normal, chained head, chained cont, deleted, migrated).
- **Lock byte** (index into ITL — which TX has this row locked).
- **Column count**.

Then per column:

- **Length byte(s)**.
- **Data bytes**.

NULL columns at the end of the row don't consume space (Oracle stops). NULLs in the middle consume 1 byte (length = 0xFF marker).

## Row Directory

Fixed-position array near block top: each entry (2 bytes) is a **pointer** to a row's start offset within the block.

Row's ROWID is `<DBA>.<row_directory_index>`. Row index doesn't change when rows are inserted/deleted within the block — that's why `ROWID` remains stable until table rebuild.

## Free Space & PCTFREE

`PCTFREE` (default 10%) reserves that percentage of block space for **updates to existing rows** — grow rows in place without migrating.

`PCTUSED` (10g and later ASSM: obsolete) — historically used with freelists (MSSM).

Under ASSM (default now), free space is tracked by **bitmap blocks** at segment level, not per-block PCTUSED.

## Row Chaining

When a row is longer than one block (e.g., >8k in a table with LONG or many columns), Oracle **chains**:

- Row starts in one block (head).
- Continues in one or more subsequent blocks (chain pieces).
- Each piece has a pointer to the next.
- Reading requires multiple block reads.

Chaining detection:

```sql
SELECT   name, value FROM v$sysstat WHERE name = 'table fetch continued row';
```

`table fetch continued row` > 0 = chaining occurring. High counts = performance loss.

Fix: bigger block size for that tablespace, or eliminate very wide rows.

## Row Migration

When an UPDATE grows a row and there's no room in its home block:

1. Original block keeps the row header + a **forwarding pointer** to the new location.
2. New block gets the row.
3. Row's ROWID doesn't change — points to the original block.

Reads that use the ROWID hit the original block, follow the pointer, read new block. Two-hop access.

Detection: same `table fetch continued row` statistic; distinguishing chaining from migration requires `ANALYZE TABLE ... LIST CHAINED ROWS`.

Fix: rebuild table (`ALTER TABLE ... MOVE`), raise PCTFREE for tables with growing rows.

## Direct-Path Load & HWM

Segments have a **High Water Mark (HWM)** — the boundary between "blocks ever used" and "blocks below the segment allocation but never touched". Full scans read up to HWM.

Direct-path insert (APPEND hint) writes above HWM, then bumps HWM at commit. Conventional insert may reuse blocks below HWM.

DELETE doesn't lower HWM. Full scan still reads all HWM blocks even if empty. Fix: `ALTER TABLE t SHRINK SPACE` or MOVE.

## Segment Space Management (ASSM)

Under ASSM (default all modern tablespaces):

- Free space tracked via **bitmap blocks** (L1, L2, L3 levels).
- L1 blocks track free space for a range of data blocks.
- L2 blocks summarize multiple L1s.
- L3 in the segment header summarizes L2s.

Inserts find a suitable block via bitmap lookup — much faster than freelist scans in high-concurrency inserts.

`segment header contention` (buffer busy waits with P3=1xx) = bitmap contention. Fix rarely needed with ASSM; if it happens, more freelist groups may help.

## Block Corruption

Every block has a checksum (`DB_BLOCK_CHECKSUM=TYPICAL` default). On read, Oracle verifies. Mismatch → `ORA-01578: data block corrupted`.

Types:

- **Physical** — bits changed on disk (bad LUN, memory corruption during write). Different SCN on backup restore may fix.
- **Logical** — block contents don't make sense (impossible ITL state, ROWID pointing to wrong block). Deeper problem.

Detection tools:

- `DBVERIFY` (external): `dbv file=... blocksize=8192`.
- `RMAN VALIDATE`: `RMAN> VALIDATE DATABASE`.
- `DBMS_HM.RUN_CHECK('Data Block Integrity Check')`.

Repair:

- `RMAN BLOCKRECOVER` (Enterprise Edition).
- `DBMS_REPAIR`.
- Segment move / rebuild.

`V$DATABASE_BLOCK_CORRUPTION` lists known bad blocks.

## Diagnostic Queries

### ITL contention

```sql
SELECT event, total_waits, ROUND(time_waited/100, 1) secs
FROM   v$system_event
WHERE  event = 'enq: TX - allocate ITL entry';

-- Per session
SELECT s.sid, s.username, s.event, s.p1raw, s.p2, s.p3
FROM   v$session s
WHERE  s.event = 'enq: TX - allocate ITL entry';
```

### High INITRANS candidates (hot tables)

```sql
SELECT owner, table_name, initrans, ini_trans, num_rows,
       (SELECT COUNT(*) FROM v$transaction) current_txs
FROM   dba_tables
WHERE  initrans < 10 AND num_rows > 10000000
ORDER  BY num_rows DESC;
```

### Row chaining check

```sql
SELECT owner, table_name, chain_cnt
FROM   dba_tables
WHERE  chain_cnt > 0
ORDER  BY chain_cnt DESC;
```

Update stats to refresh:

```sql
EXEC DBMS_STATS.GATHER_TABLE_STATS('APP','ORDERS');
```

### Segment structure

```sql
SELECT   owner, segment_name, tablespace_name,
         bytes/1024/1024 mb, extents, blocks,
         segment_type
FROM     dba_segments
WHERE    segment_name = '&object_name';
```

### Block dump for study (dev)

```sql
-- Grab a block DBA
SELECT DBMS_ROWID.ROWID_RELATIVE_FNO(ROWID) file#,
       DBMS_ROWID.ROWID_BLOCK_NUMBER(ROWID) block#
FROM   app.orders WHERE id = 1;

-- Dump it
ALTER SYSTEM DUMP DATAFILE 17 BLOCK 1234;
```

Read trace file. ITL section looks like:

```
Itl           Xid                  Uba         Flag  Lck  Scn/Fsc
0x01   0x0009.02c.00003c1a  0x00c012b0.0a83.02  --U-    1  fsc 0x0000.0234ab0f
0x02   0x000a.01b.00003c1b  0x00c00fd0.09a1.15  ----    0  fsc 0x0000.0234abc4
```

Two ITL entries. XID → transaction. UBA → undo. Flag `U` = committed with fast-commit SCN. Lock count and SCN follow.

## Interview Framing

> "What is INITRANS?"

Initial number of ITL entries reserved in each block of the table. Each ITL slot occupies 24 bytes and can hold one concurrent transaction touching that block. Insufficient INITRANS on high-concurrency tables causes `enq: TX - allocate ITL entry` waits.

> "What causes row chaining vs row migration?"

Chaining = row too big for one block; splits across multiple. Migration = row grew during UPDATE and can't fit in original block, so it's moved with a forwarding pointer.

> "What's inside an ITL entry?"

XID (transaction ID pointing to undo segment TX table), UBA (pointer to the undo records for this TX's changes on this block), flag (state), lock count (rows locked by this TX in this block), and SCN (commit SCN once cleaned out).

## Related

- [Block Structure](../04-storage/block-structure.md).
- [Row Chaining](../04-storage/row-chaining.md).
- [Row Migration](../04-storage/row-migration.md).
- [Row Storage](../04-storage/row-storage.md).
- [High Water Mark](../04-storage/high-water-mark.md).
- [Segment Space Management](../04-storage/segment-space-management.md).
- [Undo & CR Internals](undo-cr-internals.md).
- [Buffer Cache Internals](buffer-cache-internals.md).
- [ORA-00060](../26-errors/ora-00060.md).
