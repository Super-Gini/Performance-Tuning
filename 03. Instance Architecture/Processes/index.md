# Instance Architecture

The Oracle **instance** is the SGA (shared memory) plus the set of background and foreground processes running on the database server. This section is the deep dive: every memory pool, every background process, and the internal control structures (latches, mutexes, enqueues) that coordinate them.

## Contents

### Memory

| Page                                   | Purpose                                           |
| -------------------------------------- | ------------------------------------------------- |
| [SGA](memory/sga.md)                   | System Global Area — the big picture              |
| [Buffer Cache](memory/buffer-cache.md) | LRU, dirty list, working sets, keep/recycle pools |
| [Shared Pool](memory/shared-pool.md)   | Library cache, dictionary cache, cursor caching   |
| [Large Pool](memory/large-pool.md)     | RMAN, parallel exec, shared server                |
| [Java Pool](memory/java-pool.md)       | JVM heap                                          |
| [Streams Pool](memory/streams-pool.md) | LogMiner + GoldenGate + XStream buffer            |
| [Result Cache](memory/result-cache.md) | SQL and PL/SQL result cache                       |
| [PGA](memory/pga.md)                   | Private per-process memory                        |
| [ASMM](memory/asmm.md)                 | Automatic Shared Memory Management                |
| [AMM](memory/amm.md)                   | Automatic Memory Management (SGA + PGA)           |
| [HugePages](memory/hugepages.md)       | Linux HugePages for SGA                           |

### Processes

| Page                                                  | Purpose                              |
| ----------------------------------------------------- | ------------------------------------ |
| [PMON](processes/pmon.md)                             | Process Monitor                      |
| [SMON](processes/smon.md)                             | System Monitor                       |
| [DBWn](processes/dbwn.md)                             | Database Writer                      |
| [LGWR](processes/lgwr.md)                             | Log Writer                           |
| [CKPT](processes/ckpt.md)                             | Checkpoint                           |
| [ARCn](processes/arcn.md)                             | Archiver                             |
| [MMON / MMNL / MMAN](processes/mmon.md)               | Manageability monitors               |
| [RECO](processes/reco.md)                             | Recoverer (distributed transactions) |
| [LREG](processes/lreg.md)                             | Listener Registration                |
| [VKTM](processes/vktm.md)                             | Virtual Keeper of Time               |
| [DIAG](processes/diag.md)                             | Diagnostic Process                   |
| [CJQ0](processes/cjq0.md)                             | DBMS_SCHEDULER coordinator           |
| [FBDA](processes/fbda.md)                             | Flashback Data Archiver              |
| [TT00](processes/tt00.md)                             | Redo transport slaves                |
| [Parallel Execution](processes/parallel-execution.md) | PX coordinator and slaves            |

### Internals

| Page                                                      | Purpose                             |
| --------------------------------------------------------- | ----------------------------------- |
| [Foreground Processes](internals/foreground-processes.md) | Server process anatomy              |
| [Server Processes](internals/server-processes.md)         | Dedicated vs shared                 |
| [Latches](internals/latches.md)                           | Low-level SGA serialization         |
| [Mutexes](internals/mutexes.md)                           | Library cache concurrency           |
| [Enqueue Locks](internals/enqueue-locks.md)               | TM, TX, and other queue-based locks |

## Related

- [Architecture Overview](../01-fundamentals/architecture-overview.md) — the 30,000-ft view.
- [Reference / Background Processes](../25-reference/background-processes/background-process-reference.md) — one-line reference for every background process.
