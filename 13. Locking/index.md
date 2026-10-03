# Locking

Oracle's locking mechanics are elegant on paper (row locks are unlimited, writers don't block readers, no lock escalation) but the runtime reality includes TM and TX enqueues, ITL contention, deadlocks from correlated update patterns, and lock escalation-lite behaviors around DDL and foreign keys.

This section covers the enqueue types you'll see in wait events, the classic blocking-session runbook, and deadlock analysis.

## Contents

| Page                                      | Purpose                                   |
| ----------------------------------------- | ----------------------------------------- |
| [TM Locks](tm-locks.md)                   | Table-level DML locks                     |
| [TX Locks](tx-locks.md)                   | Transaction / row locks                   |
| [Blocking Sessions](blocking-sessions.md) | Finding blockers, killing sessions safely |
| [Deadlocks](deadlocks.md)                 | `ORA-00060` analysis and prevention       |

## Related

- [Enqueue Locks](../03-instance-architecture/internals/enqueue-locks.md) — enqueue subsystem.
- [Runbooks / Blocking Sessions](../27-runbooks/blocking-sessions.md).
- [Undo / Consistent Read](../05-undo/consistent-read.md) — why readers don't block writers.
