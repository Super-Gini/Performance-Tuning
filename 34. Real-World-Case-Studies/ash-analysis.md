# Case: Diagnosing a Spike Using ASH

## Setup

- 19c CDB on OCI DBCS, 4-node RAC.
- Spike observed: app-tier P95 latency doubled from 200 ms to 400 ms for a 15-minute window at 14:22 UTC.
- Alert received at 14:35; you're investigating at 14:40 — data still in memory ASH.

## Approach

Start with a top-down ASH query. Don't dive into individual SQLs yet.

## Step 1 — Top Wait Classes During the Window

```sql
SELECT wait_class, COUNT(*) samples,
       ROUND(COUNT(*)*100.0/SUM(COUNT(*)) OVER (), 1) pct
FROM   gv$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:20:00'
                       AND TIMESTAMP '2026-08-06 14:37:00'
   AND session_state = 'WAITING'
GROUP  BY wait_class
ORDER  BY 2 DESC;
```

Result:

```
WAIT_CLASS         SAMPLES    PCT
Cluster            8,420      42%
Concurrency        3,900      19%
User I/O           3,200      16%
Application        2,100      10%
Other              1,200       6%
Commit             1,000       5%
Idle                 380       2%
```

`Cluster` waits dominant — 42%. RAC coordination is the primary bottleneck.

## Step 2 — Top Cluster Events

```sql
SELECT event, COUNT(*) samples
FROM   gv$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:20:00'
                       AND TIMESTAMP '2026-08-06 14:37:00'
   AND wait_class = 'Cluster'
GROUP  BY event
ORDER  BY 2 DESC
FETCH  FIRST 10 ROWS ONLY;
```

Result:

```
EVENT                             SAMPLES
gc buffer busy release            5,200
gc current block busy             1,800
gc cr block busy                  1,300
gc current grant busy               120
```

`gc buffer busy release` — classic hot block symptom.

## Step 3 — Find the Hot Block

```sql
SELECT current_obj#, current_file#, current_block#,
       COUNT(*) samples
FROM   gv$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:20:00'
                       AND TIMESTAMP '2026-08-06 14:37:00'
   AND event = 'gc buffer busy release'
   AND current_obj# > 0
GROUP  BY current_obj#, current_file#, current_block#
ORDER  BY 4 DESC
FETCH  FIRST 10 ROWS ONLY;
```

Result:

```
CURRENT_OBJ#   FILE#   BLOCK#     SAMPLES
83421          17      1234       3,800
83421          17      1235         920
83421          17      1236         480
```

One object dominates. Which is it?

```sql
SELECT owner, object_name, object_type, subobject_name
FROM   dba_objects
WHERE  object_id = 83421;
```

Result:

```
OWNER  OBJECT_NAME                OBJECT_TYPE
APP    SEQ_ORDER_ID_$SEQUENCE     SEQUENCE
```

A sequence — its "audit block" is the hot block.

## Step 4 — Which SQLs Are Hitting It

```sql
SELECT sql_id, COUNT(*) samples,
       MIN(sample_time), MAX(sample_time)
FROM   gv$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:20:00'
                       AND TIMESTAMP '2026-08-06 14:37:00'
   AND event = 'gc buffer busy release'
   AND current_obj# = 83421
GROUP  BY sql_id
ORDER  BY 2 DESC
FETCH  FIRST 5 ROWS ONLY;
```

Result:

```
SQL_ID          SAMPLES
7ab2c5xyz       3,600
3zzq...          140
```

`7ab2c5xyz` — likely an INSERT using the sequence.

```sql
SELECT sql_fulltext FROM v$sql WHERE sql_id = '7ab2c5xyz' AND ROWNUM = 1;
```

Result:

```sql
INSERT INTO orders (id, customer_id, order_dt, status)
VALUES (seq_order_id.nextval, :b1, :b2, 'NEW')
```

Confirmed — the sequence is the hot block.

## Step 5 — Confirm Sequence Config

```sql
SELECT sequence_name, cache_size, order_flag, cycle_flag
FROM   dba_sequences
WHERE  sequence_name = 'SEQ_ORDER_ID';
```

Result:

```
SEQUENCE_NAME    CACHE_SIZE   ORDER_FLAG   CYCLE_FLAG
SEQ_ORDER_ID     20           Y            N
```

- `CACHE_SIZE = 20` — too small.
- `ORDER_FLAG = Y` — every instance forces global ordering.

Under RAC, `ORDER_FLAG = Y` means every `nextval` requires GCS coordination on the sequence audit block. With 4 nodes doing thousands of inserts per second, that block becomes molten.

## Step 6 — Root Cause

Application was configured to use ORDER sequences (mistakenly, "for consistency"). Cache size of 20 exhausted every ~10 ms per instance, causing constant re-refresh coordinating across nodes.

## Fix

```sql
ALTER SEQUENCE seq_order_id CACHE 10000 NOORDER;
```

`CACHE 10000` = each instance grabs 10000 numbers at a time. `NOORDER` = no cross-instance ordering (numbers may be out of order between instances, but strictly increasing within an instance).

Order of IDs doesn't affect anything in this app. If it did:

- Use a **timestamp** column for actual ordering.
- Or keep ORDER but increase cache to 100k+ to reduce refresh frequency.

## Verify

Within 5 minutes:

```sql
SELECT wait_class, COUNT(*) samples
FROM   gv$active_session_history
WHERE  sample_time > SYSDATE - 5/1440
   AND session_state = 'WAITING'
GROUP  BY wait_class
ORDER  BY 2 DESC;
```

`Cluster` waits back to baseline (~5%). P95 latency returned to 200 ms.

## Lessons Learned

- Cluster waits + hot object → almost always a sequence or a hot table index.
- `ORDER` sequences on RAC are almost never necessary.
- Cache size 20 is Oracle's default; **it's tuned for demos, not production**.
- ASH is the fastest tool for "what happened 15 minutes ago" — always start there.
- Consider `AUDSID`-style sequences (SYS_GUID) for high-throughput ID generation.

## Related

- [ASH](../12-performance-tuning/ash.md).
- [Cache Fusion](../18-rac/cache-fusion.md).
- [V$ACTIVE_SESSION_HISTORY](../25-reference/v-views/v-active-session-history.md).
