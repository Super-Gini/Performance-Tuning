# Undo Management

Undo is the mechanism Oracle uses to provide **read consistency**, **rollback**, and **flashback**. Every DML records a "before image" of the changed data into an undo segment. Readers of the same block use those before-images to reconstruct the block as it looked at their query's start SCN. This is Oracle's read-consistency model — no readers block writers, no writers block readers, and no shared locks needed for SELECT.

## Contents

| Page                                      | Purpose                                                             |
| ----------------------------------------- | ------------------------------------------------------------------- |
| [Undo Architecture](undo-architecture.md) | UNDO tablespace, undo segments, transaction tables, extent stealing |
| [Undo Management](undo-management.md)     | AUM (Automatic Undo Management), sizing, monitoring                 |
| [Undo Retention](undo-retention.md)       | `UNDO_RETENTION`, guaranteed retention, tuned retention             |
| [Consistent Read](consistent-read.md)     | How Oracle produces a read-consistent view using undo               |
| [ORA-01555](ora-01555.md)                 | Snapshot too old — causes and remediation                           |

## Related

- [Flashback](../16-flashback/index.md) — Flashback Query, Version Query, Data Archive all depend on undo (or FDA).
- [Locking](../13-locking/index.md) — TX enqueues and their relationship to undo.
- [Redo](../06-redo/index.md) — every undo record is itself redo-protected.
