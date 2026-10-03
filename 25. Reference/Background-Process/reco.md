# RECO — Recoverer

## Purpose

Resolves **in-doubt distributed transactions** — the "prepared" phase of a two-phase commit that got interrupted by a network or instance failure. Wakes up periodically, checks `DBA_2PC_PENDING`, contacts the coordinator, decides commit or rollback.

## Behavior

- Interval controlled by `DISTRIBUTED_LOCK_TIMEOUT` (default 60 s).
- Only runs on databases with `GLOBAL_NAMES=TRUE` and completed distributed transactions.
- On idle databases, essentially invisible.

## Check

```sql
SELECT program, spid FROM v$process WHERE program LIKE '%(RECO)%';

-- Any pending distributed TX
SELECT local_tran_id, global_tran_id, state, mixed, tran_comment
FROM   dba_2pc_pending;

-- History
SELECT * FROM dba_2pc_neighbors;
```

## Related Views

- `DBA_2PC_PENDING` — in-doubt distributed TX.
- `DBA_2PC_NEIGHBORS` — connected participants.
- `V$SESSION` for RECO's session state.

## Common Issues

- **`ORA-01591: lock held by in-doubt distributed transaction`** — Manual intervention required. `COMMIT FORCE 'tran_id'` or `ROLLBACK FORCE 'tran_id'`.
- **RECO can't reach remote** — Network / listener down. RECO retries; monitor `DBA_2PC_PENDING`.
- **In-doubt TX won't clear** — `TX.state = collecting` for weeks: check `local_tran_id`, contact remote DBA, manually resolve.

## Manual Resolution

```sql
-- If you know the outcome
COMMIT FORCE '1.2.3';         -- transaction ID from DBA_2PC_PENDING
ROLLBACK FORCE '1.2.3';

-- Purge after resolved
EXEC DBMS_TRANSACTION.PURGE_LOST_DB_ENTRY('1.2.3');
```

## References

- Oracle Database Administrator's Guide 19c — Distributed Transactions
- MOS Doc ID 100664.1 — RECO troubleshooting
