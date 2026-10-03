# RECO — Recoverer

## Overview

**RECO** is the background process that resolves **in-doubt distributed transactions**. When a transaction spans two or more databases via a database link (`db_link`), a two-phase commit (2PC) coordinates it. If a network or database failure occurs after the prepare phase but before commit, the transaction is left "in doubt." RECO periodically retries to resolve these.

## Architecture

```mermaid
flowchart LR
    Dist[Distributed Transaction<br/>2PC via DB Link] --> Prep[Prepare phase]
    Prep --> Commit1[Commit at branch A]
    Commit1 --> Fail[Network fails]
    Fail --> InDoubt[Branch B: in-doubt]
    InDoubt --> RECO
    RECO -->|periodic| Resolve[Contact remote DB]
    Resolve --> Complete[Complete commit or rollback]
    Complete --> Cleanup[DBA_2PC_PENDING cleared]
```

## Internal Working

Two-phase commit (2PC) works as follows:

1. **Coordinator** issues PREPARE to every participant.
2. Each participant PREPARES — writes redo, holds locks, but does not commit.
3. If all PREPARE, coordinator sends COMMIT to each.
4. Each participant commits.

If step 3 or 4 is interrupted for any participant, that participant is left in-doubt.

RECO:

- Periodically scans `DBA_2PC_PENDING`.
- Contacts each involved remote database.
- Reads the outcome (was there a COMMIT recorded?).
- Applies the same outcome locally: commit or rollback.

If the remote is unreachable, the DBA can manually **force commit** or **force rollback** with `COMMIT FORCE 'txn_id';` — irreversible; use only after confirming outcome.

## Components

Single process: `ora_reco_<sid>`.

## Important Parameters

| Parameter                  | Purpose                                     |
| -------------------------- | ------------------------------------------- |
| `distributed_lock_timeout` | Timeout for distributed locks (default 60s) |
| `global_names`             | Enforce global naming for DB links          |
| `open_links`               | Max concurrent DB links per session         |

## Important Views

| View                | Purpose                                 |
| ------------------- | --------------------------------------- |
| `DBA_2PC_PENDING`   | In-doubt distributed transactions       |
| `DBA_2PC_NEIGHBORS` | Involved databases for each pending txn |
| `V$BGPROCESS`       | RECO PID                                |

## Diagnostic Queries

```sql
-- Any in-doubt distributed transactions?
SELECT local_tran_id, global_tran_id, state, mixed, host, commit#, tran_comment
FROM   dba_2pc_pending;

-- What other databases are involved?
SELECT local_tran_id, in_out, database, dbuser_owner, interface
FROM   dba_2pc_neighbors;

-- RECO alive?
SELECT name, description, paddr FROM v$bgprocess WHERE name = 'RECO';
```

## Common Issues

- **In-doubt txns accumulate** — Repeated network failures + long WAN latency. Investigate root cause; consider redesigning to avoid 2PC.
- **`ORA-01591: lock held by in-doubt distributed transaction`** — Session tried to modify a row locked by an in-doubt txn. Resolve or force.
- **`ORA-02014` on forced action** — Attempt to modify data locked by in-doubt.

## Troubleshooting

1. `SELECT * FROM dba_2pc_pending;` — start here.
2. Cross-check with remote database's `DBA_2PC_PENDING`.
3. If both agree on outcome, RECO should resolve on next cycle. Force retry: `EXEC DBMS_TRANSACTION.PURGE_LOST_DB_ENTRY('txn_id');` after outcome is confirmed.
4. `COMMIT FORCE 'txn_id';` or `ROLLBACK FORCE 'txn_id';` for manual resolution — only after being certain of outcome (contact remote DBA).

## Best Practices

1. Avoid distributed transactions where possible. Consider replication (GoldenGate, materialized views) or event-driven designs.
2. Keep `global_names = TRUE` to prevent link confusion.
3. Monitor `DBA_2PC_PENDING` — should normally be empty.
4. Set `distributed_lock_timeout` appropriately for your WAN latency (default 60s).

## Interview Questions

1. **Q:** What does RECO do?
   **A:** Resolves in-doubt distributed transactions created by two-phase commit failures.

2. **Q:** What is a 2PC?
   **A:** Two-phase commit — a protocol for atomic commit across multiple databases: PREPARE phase followed by COMMIT phase.

3. **Q:** What is an in-doubt transaction?
   **A:** A distributed transaction that has PREPARED but the coordinator's outcome (COMMIT or ROLLBACK) is unknown to a participant.

4. **Q:** How do you view in-doubt txns?
   **A:** `SELECT * FROM dba_2pc_pending;`.

5. **Q:** How do you manually resolve?
   **A:** `COMMIT FORCE 'txn_id';` or `ROLLBACK FORCE 'txn_id';` — dangerous; verify remote outcome first.

## References

- Oracle Database Concepts 19c — Distributed Transactions
- Oracle Database Administrator's Guide 19c — Managing Distributed Transactions
- MOS Doc ID 126069.1 — In-Doubt Distributed Transactions
