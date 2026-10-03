# Deadlocks

## Overview

A **deadlock** is a cycle of sessions each waiting on locks held by another. Oracle detects deadlocks every ~3 seconds by walking wait-for graphs; when a cycle is found, one victim's **current statement** is rolled back with `ORA-00060: deadlock detected while waiting for resource`. The victim's transaction is _not_ rolled back — only its current statement.

Deadlocks in Oracle are almost always **application code problems**, not database problems.

## Anatomy

Simplest case:

- Session A: `UPDATE t1 SET x=x+1 WHERE id=1;` then `UPDATE t2 SET x=x+1 WHERE id=2;`
- Session B: `UPDATE t2 SET x=x+1 WHERE id=2;` then `UPDATE t1 SET x=x+1 WHERE id=1;`

Both hit the second UPDATE, waiting on each other's uncommitted row.

## Where the Evidence Lives

When a deadlock happens:

1. `ORA-00060` returned to victim.
2. Trace file written to `$ORACLE_BASE/diag/rdbms/<db>/<inst>/trace/`.
3. Alert log records: `ORA-00060: Deadlock detected. See Note 60.1 at My Oracle Support.`

Trace file contains:

- The **Deadlock Graph** (nodes and resources).
- **Rows** in each block (waiter/holder).
- **SQL text** of each participant.
- **Bind values** where captured.

### Example Deadlock Graph

```
Deadlock graph:
                         ---------Blocker(s)--------  ---------Waiter(s)---------
Resource Name                process session holds waits  process session holds waits
TX-000a0011-00042000-00000000     45      152     X             49     167           X
TX-000b0022-00051000-00000000     49      167     X             45     152           X

session 152: DID 0001-002D-00000005     session 167: DID 0001-0031-00000004
session 167: DID 0001-0031-00000004     session 152: DID 0001-002D-00000005
```

Two sessions each hold TX and want the other's TX. Classic 2-way deadlock.

## Common Deadlock Patterns

### 1. Different Row Order in UPDATEs

Two sessions update the same rows in opposite order.
**Fix**: agree on a consistent order at the application layer (`ORDER BY id`).

### 2. Bitmap Index on High-DML Table

Bitmap indexes serialize DML at the bitmap piece level; many concurrent DML on same range → deadlock.
**Fix**: use B-tree indexes for OLTP; bitmap only for DW.

### 3. Missing FK Index

Parent DELETE/UPDATE takes SRX on child; another child DML waits; deadlock via cascading.
**Fix**: index every FK column.

### 4. Table with Batch Update + Single-Row Update

Batch update sweeps rows in one order; user-driven single-row updates hit specific rows out of order.
**Fix**: reorder batch, or reduce batch size.

### 5. Sequence-driven PK + Aggregation

Two sessions insert with `MERGE ... USING dual`; if one path locks the PK index and another uses a different one, deadlock.
**Fix**: use MERGE consistently.

## Diagnostic Queries

```sql
-- Recent deadlocks in alert log
SELECT originating_timestamp, message_text
FROM   v$diag_alert_ext
WHERE  message_text LIKE '%ORA-00060%'
   AND originating_timestamp > SYSDATE - 30
ORDER  BY originating_timestamp DESC;

-- Deadlock trace files
SELECT trace_filename, modify_time
FROM   v$diag_trace_file
WHERE  trace_filename LIKE '%deadlock%'
   AND modify_time > SYSDATE - 7
ORDER  BY modify_time DESC;

-- Extract deadlock content from trace
SELECT payload
FROM   v$diag_trace_file_contents
WHERE  trace_filename = '&trace_file'
   AND payload LIKE '%Deadlock graph%';
```

## Reading a Trace

Key sections:

1. **Deadlock graph** — cycle of waits.
2. **Current SQL for each session** — what statement was hitting the deadlock.
3. **Bind values** — with what values.
4. **Rows and buffer info** — which blocks / rows.

Focus on the **SQL text** of each participant — the fix is almost always in those SQLs or their surrounding transaction.

## Prevention

1. **Consistent lock order.** Application logic must acquire locks in the same order across all code paths.
2. **Small transactions.** Shorter txns = fewer chances for deadlock.
3. **Commit frequently** (but not per-row; batch appropriately).
4. **Index FK columns.**
5. **Avoid bitmap indexes on OLTP.**
6. **Use `SELECT ... FOR UPDATE SKIP LOCKED`** for queue-processing patterns.
7. **Test concurrency early.** Deadlocks only surface at scale.

## When Deadlocks Are OK

Oracle recovers from deadlocks by rolling back one statement. If the application catches `ORA-00060` and retries, deadlocks are recoverable events — not outages. But **frequent** deadlocks (dozens/hour) indicate design problems.

## Common Issues

- **Deadlock storm** — Repeated deadlocks in same code path. Fix code.
- **Deadlock across RAC nodes** — Global deadlock detection handles; check interconnect health.
- **Deadlock in autonomous transaction** — Rare; audit AT triggers.
- **App doesn't handle `ORA-00060`** — Session errors out and transaction is left in an inconsistent state; must retry from safe checkpoint.

## Best Practices

1. **Application-side ORA-00060 retry** with exponential backoff, cap on attempts.
2. Analyze deadlock trace files monthly; fix repeat offenders.
3. Set alert on `ORA-00060` count > threshold per day.
4. Standardize row-order in batch updates (`ORDER BY id`).
5. Avoid `SELECT ... FOR UPDATE` on wide result sets.
6. Index FKs.
7. Prefer B-tree over bitmap in OLTP.
8. Document the app's transaction boundaries — most deadlocks come from unclear boundaries.

## Interview Questions

1. **Q:** What is a deadlock?
   **A:** Cycle of sessions each waiting on locks held by another.

2. **Q:** What does Oracle do?
   **A:** Detects the cycle, rolls back the current statement of one victim (not the whole transaction), throws `ORA-00060`.

3. **Q:** Common cause?
   **A:** Application code updating rows in different orders in different transactions.

4. **Q:** Where do you find the trace?
   **A:** `$ORACLE_BASE/diag/rdbms/<db>/<inst>/trace/` with `deadlock` in filename.

5. **Q:** Prevention?
   **A:** Consistent lock order, FK indexes, avoid bitmap on OLTP, small transactions.

6. **Q:** Should the app retry on ORA-00060?
   **A:** Yes — deadlocks are recoverable events. Retry with backoff.

## References

- Oracle Database Concepts 19c — Data Concurrency
- MOS Doc ID 62365.1 — Interpreting Deadlock Trace
- MOS Doc ID 15476.1 — TX Enqueue Deadlocks
