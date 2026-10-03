# Case: 15-Minute Hang Across App Tier

## Setup

- Retail app, 19c EE single-instance on-prem.
- 800 concurrent app connections.
- 14:03 UTC: app requests started backing up. By 14:05 all API endpoints returning timeouts.
- 14:18 UTC: cleared without intervention. No restart, no config change.

## Investigation Starts 30 min Later

Standard first query:

```sql
-- Nothing right now — it cleared. Go to ASH.
```

## Step 1 — ASH During the Window

```sql
SELECT COUNT(*), wait_class, session_state
FROM   v$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:00:00'
                       AND TIMESTAMP '2026-08-06 14:20:00'
GROUP  BY wait_class, session_state
ORDER  BY 1 DESC;
```

Result:

```
COUNT  WAIT_CLASS      STATE
9,800  Application     WAITING
2,100  ON CPU          -
1,400  User I/O        WAITING
600    Concurrency     WAITING
```

`Application` waits massive — 9,800 samples. `Application` class covers `enq: TX row lock`, `enq: TX index`, etc.

## Step 2 — Specific Event

```sql
SELECT event, COUNT(*) samples
FROM   v$active_session_history
WHERE  sample_time BETWEEN TIMESTAMP '2026-08-06 14:00:00'
                       AND TIMESTAMP '2026-08-06 14:20:00'
   AND wait_class = 'Application'
GROUP  BY event
ORDER  BY 2 DESC;
```

Result:

```
EVENT                                COUNT
enq: TX - row lock contention        9,750
```

Row locks. 9,750 samples over 20 min ≈ 500 avg concurrent sessions blocked.

## Step 3 — Trace the Blocker

```sql
SELECT   blocking_session, session_id, sql_id, sql_plan_hash_value,
         current_obj#, event, COUNT(*) samples
FROM     v$active_session_history
WHERE    sample_time BETWEEN TIMESTAMP '2026-08-06 14:00:00'
                         AND TIMESTAMP '2026-08-06 14:20:00'
     AND event = 'enq: TX - row lock contention'
GROUP BY blocking_session, session_id, sql_id, sql_plan_hash_value,
         current_obj#, event
ORDER BY 7 DESC
FETCH FIRST 10 ROWS ONLY;
```

Result:

```
BLOCKING_SESSION  SESSION_ID  SQL_ID       CURRENT_OBJ#  SAMPLES
842               431         a1b2c3       92341          650
842               452         a1b2c3       92341          620
842               470         a1b2c3       92341          610
...
```

All waiters point to `BLOCKING_SESSION=842`, all waiting on `CURRENT_OBJ#=92341`, all running `sql_id=a1b2c3`.

## Step 4 — What Was Blocker 842 Doing?

```sql
SELECT sample_time, session_state, event, sql_id, sql_plan_operation, current_obj#
FROM   v$active_session_history
WHERE  session_id = 842
   AND sample_time BETWEEN TIMESTAMP '2026-08-06 14:00:00'
                       AND TIMESTAMP '2026-08-06 14:20:00'
ORDER  BY sample_time;
```

Result:

```
SAMPLE_TIME               STATE     EVENT              SQL_ID    OPERATION
14:03:12                  ON CPU    -                  9zyx1...  UPDATE
14:03:22                  WAITING   db file scattered  9zyx1...  UPDATE
14:03:32                  WAITING   db file scattered  9zyx1...  UPDATE
14:03:42                  WAITING   db file scattered  9zyx1...  UPDATE
... (continues for 14 minutes)
14:17:52                  WAITING   db file scattered  9zyx1...  UPDATE
14:18:02                  ON CPU    -                  9zyx1...  UPDATE
14:18:12                  -         -                  -         -    # ended
```

Session 842 spent 15 minutes on `db file scattered read` — a full table scan — while holding a row lock.

## Step 5 — What SQL Was 9zyx1?

```sql
SELECT sql_fulltext FROM dba_hist_sqltext WHERE sql_id = '9zyx1...';
```

Result:

```sql
UPDATE inventory
SET    reserved_qty = reserved_qty + :b1
WHERE  product_id = :b2 AND status = 'AVAILABLE'
```

Then check the plan:

```sql
SELECT plan_hash_value, MIN(sample_time) FROM v$active_session_history
WHERE  sql_id = '9zyx1...' AND session_id = 842
GROUP  BY plan_hash_value;
```

Plan hash `123456`. `DBMS_XPLAN.DISPLAY_AWR('9zyx1...', 123456)`:

```
UPDATE STATEMENT
 UPDATE INVENTORY
  TABLE ACCESS FULL INVENTORY
```

Full table scan on a 200 GB `inventory` table.

## Step 6 — Why Full Scan?

Session 842 had a bind value `:b2` (product_id) with an atypical value — a product that doesn't have an index-covered range, causing the optimizer's bind peeking to pick full scan.

But wait — normally with a bind, it should use the primary key index. Check:

```sql
SELECT sql_id, plan_hash_value, COUNT(*)
FROM   dba_hist_sqlstat
WHERE  sql_id = '9zyx1...'
GROUP  BY sql_id, plan_hash_value;
```

Result:

```
SQL_ID    PLAN_HASH_VALUE   COUNT
9zyx1...  789012            15,340    <-- normal plan (INDEX scan)
9zyx1...  123456                87    <-- bad plan (FULL)
```

Bad plan appeared only 87 times. Look for when:

```sql
SELECT MIN(snap_id) FROM dba_hist_sqlstat
WHERE  sql_id = '9zyx1...' AND plan_hash_value = 123456;
```

First bad-plan snap = today at 13:45 UTC. Between 13:45 and 14:03 there was a hard parse that picked the bad plan, and then session 842 got it.

## Root Cause

1. **Bad bind value** — first query at 13:45 had a low-cardinality bind that made the optimizer estimate high cardinality → full scan plan.
2. **Bind peeking + adaptive cursor sharing** issue — plan cached for shape of that bind, applied to next call by session 842.
3. Session 842 was **holding the row lock during the full scan** — its UPDATE was blocked from progress but the earlier UPDATE hadn't committed. So every waiter was stuck.

## Fix

Immediate:

```sql
-- Kill the blocker (would have unfrozen the app)
ALTER SYSTEM KILL SESSION '842,<serial>' IMMEDIATE;
```

Long-term:

```sql
-- Load the good plan
DECLARE
  n NUMBER;
BEGIN
  n := DBMS_SPM.LOAD_PLANS_FROM_AWR(
         begin_snap => &pre_bad_snap,
         end_snap   => &pre_bad_snap,
         basic_filter => q'[sql_id = '9zyx1...' AND plan_hash_value = 789012]');
END;
/

-- Or force cursor to use only the index
CREATE INDEX inventory_pk_covered ON inventory(product_id, status) COMPRESS 1;
```

## Lessons Learned

- **Long-held row locks + slow query in the same session = massive blocking chain**.
- ASH is essential when the incident is over. Alerts must fire so someone notices in time.
- Bind peeking + plan flip is the #1 cause of intermittent hangs.
- SPB for anything critical (like this UPDATE) prevents recurrence.
- Enforce `SESSION_TIMEOUT` / `IDLE_TIME` at profile level so runaway transactions self-terminate.

## Related

- [Blocking Sessions runbook](../27-runbooks/blocking-sessions.md).
- [Deadlocks](../13-locking/deadlocks.md).
- [ASH](../12-performance-tuning/ash.md).
- [SQL Plan Management](../11-sql-optimizer/sql-plan-management.md).
