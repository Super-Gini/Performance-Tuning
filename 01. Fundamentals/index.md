# Oracle Fundamentals

The foundation section. Everything else in this encyclopedia assumes you understand what an Oracle **instance** is versus a **database**, how the process and memory architectures fit together, how the database transitions through startup and shutdown states, and which edition/feature set you are running on.

## Contents

| Page                                              | Purpose                                                                                                                                              |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Overview](overview.md)                           | Positioning of Oracle 19c in the release timeline, LTS vs innovation releases, high-level product family.                                            |
| [Oracle Editions](oracle-editions.md)             | Enterprise Edition, Standard Edition 2, Express, Personal — feature matrix, licensing pitfalls.                                                      |
| [Oracle 19c Features](oracle-19c-features.md)     | The 19c-specific capabilities you must know: Automatic Indexing, Active Data Guard DML redirection, Hybrid Partitioned Tables, Real-Time Statistics. |
| [Database vs Instance](database-vs-instance.md)   | The single most misunderstood distinction in Oracle.                                                                                                 |
| [Architecture Overview](architecture-overview.md) | The 30,000-foot view: instance + database + listener + client.                                                                                       |
| [Startup Process](startup-process.md)             | `NOMOUNT → MOUNT → OPEN`, what each state unlocks.                                                                                                   |
| [Shutdown Process](shutdown-process.md)           | `NORMAL / TRANSACTIONAL / IMMEDIATE / ABORT` — when each is safe.                                                                                    |

---

## How to Use This Section

Read in order if you're new to Oracle. Skip to [Database vs Instance](database-vs-instance.md) and [Architecture Overview](architecture-overview.md) if you're refreshing. Interviewers routinely ask about the startup and shutdown modes — those two pages contain the exact wording you should use.

## Related Sections

- [Instance Architecture](../03-instance-architecture/index.md) — deep dive into SGA, PGA, and background processes.
- [Installation](../02-installation/index.md) — get a 19c database running.
- [Multitenant](../08-multitenant/index.md) — the CDB/PDB model that all 19c databases use.
