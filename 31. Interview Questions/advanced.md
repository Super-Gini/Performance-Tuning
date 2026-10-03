# Advanced — Oracle DBA Interview Questions

## Internals

**Q: What is a latch?**
A: Low-level, fast, exclusive lock (spinlock) protecting a small in-memory structure. If a session can't acquire, it spins (short) then sleeps (`latch: X`). Latches are held very briefly.

**Q: What is a mutex, and how is it different from a latch?**
A: Since 11g, mutexes replaced some latches (mainly library cache). Similar semantics but with hierarchical structure per KGL object, allowing more concurrency. Waits show as `library cache: mutex X`.

**Q: What is the library cache?**
A: The part of the shared pool that stores parsed SQL, PL/SQL, packages, and their metadata. Managed by the KGL (Kernel Generic Library). Contention here is the top cause of `library cache: mutex X` waits.

**Q: What is cursor sharing?**
A: When the same SQL text (with binds) reuses the same parsed cursor across sessions. Enabled by using bind variables. `CURSOR_SHARING=FORCE` rewrites literals to binds — a bandaid, not a fix.

**Q: What is SCN?**
A: **System Change Number** — monotonically increasing 6-byte value that timestamps every change. Used for read consistency, recovery, Data Guard sync, and every dependency in the database.

**Q: How does Oracle handle read consistency?**
A: When you query, Oracle records the current SCN. If a row's block has been modified since (ITL entry newer than your SCN), Oracle constructs a **CR block** using undo. Same rows, older values.

## Corruption

**Q: How would you detect block corruption?**
A: `DBVERIFY` (external), `DBMS_HM.RUN_CHECK('Data Block Integrity Check')`, `RMAN VALIDATE`. Runtime symptom: ORA-01578.

**Q: How would you fix ORA-01578?**
A: Options: 1) Block Media Recovery via `RMAN BLOCKRECOVER`. 2) Restore datafile. 3) Move the affected data segment (`ALTER TABLE MOVE`) to skip the block. Cross-check with `V$DATABASE_BLOCK_CORRUPTION`.

**Q: What is `DB_LOST_WRITE_PROTECT`?**
A: Enables detection of "lost writes" — when the storage acknowledges a write but doesn't persist it. On primary, records block SCN; on standby, compares. `TYPICAL` recommended for all critical DBs.

## Deep Tuning

**Q: How would you debug ORA-04031?**
A: Query `V$SGASTAT` for shared pool free memory, `V$SQL` for cursors with high `SHARABLE_MEM`, `X$KSMSP` for free chunk distribution. Bind-variable audit. Set `SHARED_POOL_RESERVED_SIZE`. Grow shared pool.

**Q: Explain how to diagnose a plan flip.**
A: `DBA_HIST_SQLSTAT` shows multiple `PLAN_HASH_VALUE` per SQL_ID. Compare elapsed/exec across plans. Capture the good plan as a SQL Plan Baseline (`DBMS_SPM.LOAD_PLANS_FROM_CURSOR_CACHE`). Investigate root cause (stats, adaptive, ACS, feedback, RU regression).

**Q: How does adaptive plans work?**
A: 12c+ feature: optimizer picks a default plan but includes a decision point (usually a join method switch). At runtime, based on actual rows, it may switch. Result: plan can differ from what EXPLAIN PLAN shows.

**Q: What is bind peeking?**
A: On hard parse with bind variables, optimizer peeks at the first bind values to choose a plan. Subsequent executions may run with different values on the same plan → skew problems. `_optim_peek_user_binds` controls; ACS mitigates.

**Q: Explain adaptive cursor sharing.**
A: Oracle tracks bind selectivity across executions. If one bind is high-cardinality and another low, Oracle can generate multiple child cursors — bind-aware. Sits alongside bind peeking. Visible via `V$SQL.IS_BIND_AWARE`.

## Data Guard Deep

**Q: SYNC vs ASYNC redo transport?**
A: **SYNC** — LGWR waits for standby ACK before commit; zero data loss but adds latency. **ASYNC** — LGWR doesn't wait; data loss window = last unshipped redo.

**Q: What is MAX AVAILABILITY?**
A: A protection mode that's SYNC when both sides healthy, temporarily ASYNC when standby is unreachable. Then reverts. Trade-off between zero-data-loss and availability.

**Q: Explain Data Guard broker.**
A: A management framework (DGMGRL) that orchestrates configuration, monitoring, switchover/failover. Uses its own metadata; commands abstract the raw ALTER DATABASE syntax.

## RAC Deep

**Q: What is cache fusion?**
A: RAC's shared-cache mechanism. Blocks needed by another instance are shipped over the interconnect. `LMS` process serves blocks. GCS (Global Cache Service) coordinates ownership.

**Q: What is a `gc buffer busy` wait?**
A: A session waited for another session to finish getting a block from a remote instance. Symptom of hot blocks under RAC + high concurrency.

**Q: What causes a node eviction?**
A: CSSD miscount timeout (interconnect failure), resource starvation, hardware failure. Reads: `alertPRDDB.log`, `ocssd.log`, `crsd.log`.

## Multitenant

**Q: CDB vs PDB vs Application Container?**
A: **CDB** = container database. **PDB** = pluggable database inside CDB. **Application Container** = a hierarchy of PDBs sharing common application objects. **Root** = the CDB metadata layer.

**Q: What are common users vs local users?**
A: **Common** = user defined in CDB root, exists in all PDBs (name starts with `C##`). **Local** = defined in a specific PDB. Both can hold privileges independently.

## Related

- [Internals Deep Dive](../32-internals-deep-dive/index.md).
- [Real-World Case Studies](../34-real-world-case-studies/index.md).
