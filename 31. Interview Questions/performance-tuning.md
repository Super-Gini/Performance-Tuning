# Performance Tuning — Interview Questions

**Q: How do you know a DB is slow?**
A: Users report it 🙂. Objectively: `V$SYSMETRIC` for `Average Active Sessions` and `Database Time Per Sec` trending up. AWR compare vs baseline. ASH shows non-idle wait classes climbing.

**Q: What's your first move when the DB is slow?**
A: Look at `V$SESSION` → active sessions, wait events, sql_id. Then ASH last 15 min: top event, top SQL, top wait class. Match to whether it's CPU, IO, concurrency, or user-app.

**Q: What is `db file sequential read`?**
A: Single-block wait — usually index access. If avg > 10 ms, storage is slow or working set doesn't fit buffer cache.

**Q: What is `db file scattered read`?**
A: Multi-block wait — full table/index scan. Optimizer picked it; might indicate missing index or a legitimate DW scan.

**Q: What is `direct path read`?**
A: Foreground process reads blocks directly into PGA, bypassing buffer cache. 11g+ auto for large tables. Efficient for scans but no cache benefit for next execution.

**Q: What is `log file sync`?**
A: Session waiting for LGWR to persist commit redo. Slow (> 10 ms avg) → storage / LGWR issue. Batch commits reduce total waits.

**Q: What is `enq: TX - row lock contention`?**
A: Session waiting for another session's transaction lock on a specific row. Application concurrency issue.

**Q: `cursor: pin S wait on X` — what causes it?**
A: Multiple sessions parsing the same SQL concurrently — hard parse contention. Fix: use bind variables so parsing happens once.

**Q: `library cache: mutex X` — cause?**
A: Extreme cursor concurrency — same SQL_ID executed by many sessions at once. Mitigations: bind variables, `_kgl_hot_object_copies`.

**Q: How do you tune a specific slow SQL?**
A: 1) Get SQL_ID from V$SQL / V$SESSION. 2) `DBMS_XPLAN.DISPLAY_CURSOR('sql_id',NULL,'ALL ALLSTATS LAST')` — actual plan and cardinality vs estimate. 3) Look for row-estimate skews. 4) Check stats. 5) Consider index / hint / SPB.

**Q: What are the top 3 causes of a bad SQL plan?**
A: Stale/missing statistics; bind peeking issue; adaptive plan flip. Also: newer optimizer_features_enable after RU.

**Q: What's an execution plan cost vs elapsed time?**
A: **Cost** = optimizer's estimate of work relative to a fixed baseline (`SREADTIM`). **Elapsed** = actual runtime. Cost isn't ms — it's a comparison metric among alternatives.

**Q: How does Oracle estimate cardinality?**
A: From column statistics — `NUM_DISTINCT`, `LOW_VALUE`, `HIGH_VALUE`, histograms. If you have `NUM_ROWS` accurate and column-level histograms, estimates are usually close.

**Q: When would you gather statistics?**
A: After bulk load (10% or more rows changed), after schema changes, after Data Pump import. Also: index rebuild, MV refresh. Auto-stats job (in maintenance window) handles most cases.

**Q: Dynamic sampling?**
A: Optimizer collects small on-the-fly samples during parse when stats are missing/stale. Level 0–11. Cost vs accuracy trade-off.

**Q: What is `optimizer_features_enable` used for?**
A: Rollback lever — set to older version to disable new optimizer behaviors. Use after an RU introduces plan regressions, while you triage.

**Q: How do you identify high-CPU sessions?**
A: `V$SESSION.EVENT IS NULL AND STATE='WAITED SHORT TIME'` = on CPU. Or ASH `SESSION_STATE='ON CPU'`. Join V$PROCESS for OS PID; compare to `top`.

**Q: What is SQL Monitor?**
A: 11g+ auto-monitors long-running SQL (> 5s or parallel). Report via `DBMS_SQLTUNE.REPORT_SQL_MONITOR('sql_id')` or OEM. Shows per-line actuals — where time is really spent.

**Q: PGA tuning?**
A: Set `PGA_AGGREGATE_TARGET` (soft) and `PGA_AGGREGATE_LIMIT` (hard cap, 12c+). Monitor `V$PGA_TARGET_ADVICE`. Under-sized PGA causes hash/sort spills to TEMP.

**Q: When do you use hints?**
A: Rarely — as a last resort. Prefer stats, indexes, SPB. If you must, prefer stability-preserving hints (`INDEX`, `FULL`, `USE_NL`, `USE_HASH`) over ones that pin plans forever.

**Q: What's your favorite tuning script?**
A: (Personal — mine is top-SQL by DB Time from ASH, joined with V$SQLAREA for full text, plus a plan comparison across snapshots.)

## Related

- [Performance Tuning chapter](../12-performance-tuning/index.md).
- [Top SQL](../12-performance-tuning/top-sql.md).
- [Wait Events](../12-performance-tuning/wait-events.md).
- [SQL Tuning chapter](../36-sql-tuning/index.md).
