# Intermediate — Oracle DBA Interview Questions

## Memory Management

**Q: ASMM vs AMM vs Manual?**
A: **Manual** — you size each pool (`DB_CACHE_SIZE`, `SHARED_POOL_SIZE`). **ASMM** — set `SGA_TARGET` and Oracle auto-tunes pools. **AMM** — `MEMORY_TARGET` also manages PGA. On modern Linux with HugePages, use ASMM.

**Q: Why not AMM on Linux?**
A: AMM uses `/dev/shm` (POSIX SHM) which can't use HugePages. Result: high TLB pressure on large SGAs. Use ASMM (`SGA_TARGET`) + manual `PGA_AGGREGATE_TARGET` + `USE_LARGE_PAGES=ONLY`.

**Q: What is HugePages?**
A: Linux 2 MB (default) or 1 GB (GB huge pages) memory pages instead of 4 KB. Reduces TLB misses for large SGAs. Reserve at boot; can't be swapped.

## Undo

**Q: What is undo?**
A: Pre-image data for uncommitted transactions. Used for rollback, read consistency, and flashback query.

**Q: What causes ORA-01555?**
A: Long-running query needs pre-image that was overwritten. Cause: undo tablespace too small, `UNDO_RETENTION` too low, high DML overwriting the needed undo.

**Q: What is `RETENTION GUARANTEE`?**
A: Undo tablespace attribute that prevents unexpired undo from being overwritten even under pressure. Prevents ORA-01555 but may cause ORA-30036 (undo full).

## Redo

**Q: What is a log switch?**
A: LGWR fills the current online redo log and moves to the next group. Triggers a checkpoint and (in archivelog mode) an ARCn archive.

**Q: What's the difference between COMMIT and checkpoint?**
A: COMMIT flushes redo for **that transaction** to disk (via LGWR). Checkpoint flushes **all dirty buffers** up to a specific SCN and updates control/datafile headers.

**Q: When would you increase redo log size?**
A: When log switches happen too frequently (typical target: every 15–20 min at peak load). Frequent switches → too many archived logs, `log file switch (checkpoint incomplete)` waits.

## Optimizer

**Q: What's an execution plan?**
A: The set of operations Oracle uses to run a SQL statement — access paths (index, full scan), join methods (NL, hash, merge), and order. `EXPLAIN PLAN` shows the estimated plan; `DBMS_XPLAN.DISPLAY_CURSOR` shows the actual runtime plan.

**Q: NL vs Hash vs Sort-Merge Join?**
A: **NL** — probe outer rows; fast when small outer. **Hash** — build hash table on smaller side; fast for large sets. **Sort-Merge** — sort both, merge; fast when inputs already sorted. Optimizer picks based on cardinality.

**Q: What's `cardinality feedback`?**
A: Oracle 11.2+ optimizer feature: after a run, it may adjust cardinality estimates and re-parse. Sometimes causes plan instability. Often disabled via `_optimizer_use_feedback=FALSE`.

**Q: What is a SQL Plan Baseline?**
A: A stored, "known good" plan for a SQL_ID. Optimizer prefers it over new plans. Prevents plan instability after RUs / stats gathering.

## Performance Diagnostics

**Q: What is AWR?**
A: **Automatic Workload Repository** — Oracle's built-in performance snapshot store. MMON takes a snapshot each hour; `DBA_HIST_*` views expose history. AWR reports compare snapshots.

**Q: What is ASH?**
A: **Active Session History** — MMNL samples every active session every second into a memory ring. Persisted (every 10th) to `DBA_HIST_ACTIVE_SESS_HISTORY`.

**Q: Difference between AWR and ASH?**
A: AWR = aggregated stats snapshots (top SQL, wait events, sysstat) at 60-min granularity. ASH = per-session-per-second sampling — for time-based drill-down.

## Locking

**Q: TX vs TM lock?**
A: **TX** = transaction lock (row-level). Held for the life of a transaction on rows modified. **TM** = table-level lock. Held during any DML/DDL on the table. Missing FK indexes cause TM cascades.

**Q: What is a deadlock?**
A: A cycle where sessions each hold a lock the other wants. Oracle detects every ~3s, rolls back one victim's current statement (not full transaction) and raises ORA-00060.

## Networking

**Q: Difference between dedicated and shared server?**
A: **Dedicated** — one server process per client connection. **Shared** — dispatchers hand requests to a pool of shared servers. Modern default is dedicated; shared server is legacy but occasionally used for connection thousands.

**Q: What is TNS?**
A: **Transparent Network Substrate** — Oracle's networking layer. `tnsnames.ora` is the client-side alias directory; `listener.ora` is the server-side listener config.

## Related

- [Beginner](beginner.md), [Advanced](advanced.md).
- [Performance Tuning](performance-tuning.md).
