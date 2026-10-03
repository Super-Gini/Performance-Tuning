# Internals Deep Dive

The pages here go one layer below what `DBA_*` and `V$*` present — into the actual SGA structures, hash tables, chunks, latches, mutexes, redo strands, undo TX tables, ITL entries, and cache fusion resources that make Oracle work. Read these when a symptom doesn't yield to standard tuning and you need to know **why** the engine behaves the way it does.

## Contents

### Core Layer

| Page                                                | Topic                                                                                               |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| [Buffer Cache Internals](buffer-cache-internals.md) | Buffer headers (`X$BH`), CBC latches, working sets, touch counts, CR clones, delayed block cleanout |
| [Cursor Internals](cursor-internals.md)             | Parent/child cursors, ACS state machine, `V$SQL_SHARED_CURSOR` decoding, rolling invalidation       |
| [Library Cache](library-cache.md)                   | KGL layer — namespaces, handles, locks, pins, dependency graph, mutex vs latch                      |
| [Latches vs Mutexes](latches-vs-mutexes.md)         | Get/release state machines, spin/sleep, levels, classes, diagnostics                                |
| [SCN](scn.md)                                       | 48-bit format, generation (`kcmgas`/`kcmgcs`), synchronization across DBs, rate limit               |

### Memory Management

| Page                                                | Topic                                                                    |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| [KGH — Shared Pool Heap](kgh-shared-pool-heap.md)   | Sub-heaps, chunks, size classes, reserved pool, ORA-04031 anatomy        |
| [PGA Workarea Internals](pga-workarea-internals.md) | Optimal/one-pass/multi-pass, `PGA_AGGREGATE_LIMIT`, TEMP spill mechanics |
| [Result Cache Internals](result-cache-internals.md) | Single-latch design, dependency tracking, when it helps vs hurts         |

### Storage & Change Records

| Page                                        | Topic                                                                       |
| ------------------------------------------- | --------------------------------------------------------------------------- |
| [Block Format & ITL](block-format-itl.md)   | Block layout, ITL entries, row directory, chaining, migration, ASSM bitmaps |
| [Redo Internals](redo-internals.md)         | Change vectors, strands, IMU + private redo, LGWR post/wait, RBA            |
| [Undo & CR Internals](undo-cr-internals.md) | Undo segments, TX table, CR block construction, ORA-01555 anatomy           |

### Coordination

| Page                                                | Topic                                                             |
| --------------------------------------------------- | ----------------------------------------------------------------- |
| [Cache Fusion Internals](cache-fusion-internals.md) | 2-way/3-way transfers, PI blocks, LMS behavior, RAC waits decoded |

### Instrumentation

| Page                                            | Topic                                                         |
| ----------------------------------------------- | ------------------------------------------------------------- |
| [Fixed Tables (X$)](fixed-tables-x-dollar.md)   | Naming convention, V$ ↔ X$ mapping, table catalogue by domain |
| [Wait Event Framework](wait-event-framework.md) | KSLE layer, P1/P2/P3 decoding, histograms, ASH sampling       |

## How to Use These

- **When a diagnostic query returns something unexpected**: consult the relevant "Internals" page to understand what the numbers actually represent.
- **When a wait event doesn't match a runbook**: look up the event in [Wait Event Framework](wait-event-framework.md), decode its parameters via [Fixed Tables](fixed-tables-x-dollar.md), and follow the pointer chains into the underlying structures.
- **When Oracle Support asks for evidence**: these pages tell you which `X$` tables and which parameters to sample.

## Related

- [Instance Architecture](../03-instance-architecture/index.md) — the day-to-day view.
- [Advanced Interview Questions](../31-interview-questions/advanced.md) — practice explaining these topics.
- [Real-World Case Studies](../34-real-world-case-studies/index.md) — internals used in practice.
