# Mutex Contention

## Overview

**Mutexes** are finer-grained serialization primitives than latches — introduced in 11g for the library cache and cursor pinning. Where a `library cache` latch protected the entire library cache in 10g, mutexes now protect individual cursors. Mutex contention appears as `library cache: mutex X` and `cursor: pin S wait on X` waits.

## Common Mutex Events

| Wait Event                      | Meaning                                          |
| ------------------------------- | ------------------------------------------------ |
| `cursor: pin S`                 | Shared pin request; another session holds S or X |
| `cursor: pin X`                 | Exclusive pin; another session holds any pin     |
| `cursor: pin S wait on X`       | Shared pin blocked by exclusive holder           |
| `library cache: mutex X`        | Library cache handle mutex                       |
| `library cache: bucket mutex X` | Cache bucket mutex (18c+)                        |
| `library cache load lock`       | Loading a compiled object                        |

## Root Causes

- **Excessive child cursors** — Same SQL, many bind mismatches → each parse acquires mutex.
- **Frequent invalidations** — DDL storm invalidates cursors, forcing re-parse under mutex.
- **Statistics gather with `NO_INVALIDATE=FALSE`** — Instant invalidation storm.
- **Application connection churn** — every new session forces cursor loads.
- **Hot cursor** — one SQL executed by hundreds of concurrent sessions.

## Detection

```sql
-- Top mutex sleeps
SELECT mutex_type, location, SUM(sleeps) AS sleeps, SUM(gets) AS gets
FROM   v$mutex_sleep
GROUP  BY mutex_type, location
ORDER  BY sleeps DESC
FETCH FIRST 20 ROWS ONLY;

-- Recent mutex sleep history
SELECT sleep_timestamp, mutex_type, location,
       requesting_session, blocking_session, mutex_identifier
FROM   v$mutex_sleep_history
ORDER  BY sleep_timestamp DESC
FETCH FIRST 30 ROWS ONLY;

-- Wait events
SELECT event, total_waits,
       ROUND(time_waited_micro/1e6, 1) AS total_sec,
       ROUND(time_waited_micro/DECODE(total_waits,0,1,total_waits)/1000, 2) AS avg_ms
FROM   v$system_event
WHERE  event LIKE '%mutex%' OR event LIKE 'cursor:%'
ORDER  BY total_sec DESC;
```

## Investigate a Hot Cursor

```sql
-- Which SQL has the most child cursors?
SELECT sql_id, version_count, loaded_versions, invalidations,
       SUBSTR(sql_text, 1, 80) AS sql
FROM   v$sqlarea
WHERE  version_count > 5
ORDER  BY version_count DESC
FETCH FIRST 20 ROWS ONLY;

-- Why did children not share?
SELECT * FROM v$sql_shared_cursor
WHERE  sql_id = '&sql_id';

-- Each 'Y' column = a reason for a new child cursor
```

Common mismatch reasons:

- `bind_mismatch` — Bind data types differ (VARCHAR2(10) vs VARCHAR2(20)).
- `optimizer_mismatch` — Session parameter difference.
- `nls_settings_mismatch` — NLS_LANG diff.
- `literal_mismatch` — Literal SQL, not binds.

## Fix Patterns

### 1. Fix child cursor bloat

- Standardize NLS on connections.
- Standardize `optimizer_*` parameters.
- Fix bind lengths: use consistent widths (e.g., `VARCHAR2(4000)` in JDBC).

### 2. `no_invalidate=AUTO_INVALIDATE`

```sql
EXEC DBMS_STATS.GATHER_TABLE_STATS('HR','EMPLOYEES',
       no_invalidate=>DBMS_STATS.AUTO_INVALIDATE);
```

Spreads invalidation over ~5 hours instead of instantly. Avoid `NO_INVALIDATE=FALSE` in production.

### 3. Reduce DDL

Every DDL invalidates dependent cursors. Batch DDL to maintenance windows.

### 4. Session cursor cache

```sql
ALTER SYSTEM SET session_cached_cursors = 200 SCOPE=BOTH;
-- Session-scope
ALTER SESSION SET session_cached_cursors = 200;
```

Reduces soft parses by caching cursor handles per session.

### 5. Application changes

- Use bind variables.
- Consistent PLSQL/JDBC datatype declarations.

## Diagnostic Queries

```sql
-- Hard vs soft parse rate
SELECT name, value FROM v$sysstat
WHERE  name LIKE 'parse count%';

-- Session cursor cache hit
SELECT name, value FROM v$sysstat
WHERE  name LIKE 'session cursor cache%';

-- Currently waiting on cursor mutex
SELECT sid, event, seconds_in_wait, p1raw AS mutex_addr, p2 AS mutex_val
FROM   v$session_wait
WHERE  event LIKE 'cursor:%';
```

## Common Issues

- **`cursor: pin S wait on X` climbing** — Cursor being invalidated while others read. `NO_INVALIDATE=AUTO` fixes DBMS_STATS-driven cases.
- **`library cache: mutex X`** — Many concurrent parses. Check for hard-parse burst.
- **RAC — cross-instance cursor mutex** — Sessions coordinating shared pool structures across nodes.

## Best Practices

1. **Bind variables** everywhere.
2. `session_cached_cursors = 100–200`.
3. `no_invalidate=AUTO_INVALIDATE` for DBMS_STATS.
4. Standardize NLS and optimizer environment.
5. Consistent bind widths in JDBC / OCI.
6. Monitor `V$SQLAREA.VERSION_COUNT` — > 10 needs investigation.
7. Batch DDL to maintenance windows.
8. Alert on cursor mutex waits > 5% of DB Time.

## Interview Questions

1. **Q:** Mutex vs latch?
   **A:** Both are lightweight serialization. Mutex is finer-grained (per-object) and introduced in 11g for library cache.

2. **Q:** `cursor: pin S wait on X`?
   **A:** Shared pin request blocked by exclusive holder — typically because a session is invalidating the cursor while others read.

3. **Q:** How to reduce child cursor count?
   **A:** Fix bind mismatches (widths, types), standardize NLS + optimizer env, avoid literal SQL.

4. **Q:** `V$SQL_SHARED_CURSOR`?
   **A:** Explains why child cursors don't share; each Y column is a mismatch reason.

5. **Q:** How to detect DBMS_STATS-caused mutex storm?
   **A:** Check `V$MUTEX_SLEEP_HISTORY` timing correlated with stats gather jobs. Fix with AUTO_INVALIDATE.

## References

- Oracle Database Performance Tuning Guide 19c — Cursor Sharing
- MOS Doc ID 1341736.1 — Diagnosing Cursor Mutex Waits
- MOS Doc ID 1298015.1 — cursor: pin S wait on X
