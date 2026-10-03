# Case: AWS DMS Silent Data Mismatch

## Setup

- Migrating 3 TB from on-prem Oracle 12.2 → AWS RDS Oracle 19c.
- **AWS DMS** with full-load + CDC.
- Cutover planned in 3 weeks. Ongoing CDC sync.
- Two weeks in: reconciliation shows **1,247 rows different** in `ORDERS` table.

## Investigation

### Step 1 — Reproduce

Row counts match:

```sql
-- Source
SELECT COUNT(*) FROM orders;  -- 12,048,231

-- Target (RDS)
SELECT COUNT(*) FROM orders;  -- 12,048,231
```

Full counts equal. So it's row-level content:

```sql
-- Comparison in a rec script
SELECT order_id, order_date, amount, status
FROM   orders WHERE order_id = 8234156;
```

Source: `amount = 145.20`, `status = 'SHIPPED'`.
Target: `amount = 145.20`, `status = 'PROCESSING'`.

The `status` column is out of sync for some rows.

### Step 2 — Which Rows Differ?

Recon script found 1,247 rows across last 30 days. Sample:

```
ORDER_ID    SRC_STATUS    TGT_STATUS    LAST_UPDATE
8234156     SHIPPED       PROCESSING    2026-07-25 10:23:00
8234891     COMPLETED     SHIPPED       2026-07-25 10:23:15
8235022     SHIPPED       PROCESSING    2026-07-25 10:24:01
...
```

All ~10:23-10:24 UTC on 2026-07-25. Cluster.

### Step 3 — DMS Task Logs

Look for that window:

```
> aws logs get-log-events --log-group-name dms-tasks/prd-migrate \
    --log-stream-name replicate-task \
    --start-time <2026-07-25 10:20 UTC epoch> \
    --end-time <2026-07-25 10:30 UTC epoch>
```

Log excerpt:

```
2026-07-25 10:23:15 REPLICATION_SLAVE: Full load done for ORDERS
2026-07-25 10:23:16 CDC: Starting CDC from SCN 12345678
2026-07-25 10:23:16 CDC: LOB truncation detected on column status VARCHAR2(20), truncated to 12 bytes
2026-07-25 10:23:17 CDC: Applying changes at SCN 12345680
```

Two things:

1. **CDC started from an SCN 5 minutes after full load completed** — normal DMS behavior, but **that gap = data loss window**.
2. **"LOB truncation" warning** — DMS treated `status VARCHAR2(20)` as a LOB and truncated to 12 bytes for some rows.

### Step 4 — Verify Gap

DMS full-load's captured SCN vs CDC start SCN:

```
Full load captured up to SCN 12345670
CDC started at         SCN 12345678
```

Gap of 8 SCNs = ~30 seconds of transactions **missed** in some tables.

Actually the "1,247 differences" are from two causes mixed:

- Some are truncation on `status = 'PROCESSING_LATE'` → target sees `'PROCESSING_L'`.
- Some are the SCN gap — transactions that committed between full-load-end and CDC-start.

### Root Cause

1. **DMS default behavior**: full load and CDC don't share an SCN — there's a gap where transactions can be lost.
2. **DMS LOB-mode default**: certain string types get treated as LOBs and truncated.

### Fix

#### 1. Restart Migration with SCN-Coordinated Approach

Use **DMS + LogMiner** with **timestamp-based CDC start** aligned to the full-load timestamp:

```json
{
  "TargetMetadata": {
    "SupportLobs": true,
    "FullLobMode": false,
    "LimitedSizeLobMode": true,
    "LobMaxSize": 32,
    "InlineLobMaxSize": 0
  },
  "FullLoadSettings": {
    "TargetTablePrepMode": "TRUNCATE_BEFORE_LOAD",
    "MaxFullLoadSubTasks": 8,
    "TransactionConsistencyTimeout": 600,
    "CommitRate": 10000
  },
  "ChangeProcessingTuning": {
    "MinTransactionSize": 1000,
    "CommitTimeout": 1,
    "MemoryLimitTotal": 1024
  }
}
```

Key changes:

- `TransactionConsistencyTimeout: 600` — DMS waits 10 min for open transactions to close before starting CDC → smaller gap.
- `LimitedSizeLobMode + LobMaxSize=32` for `status` — prevents truncation.

#### 2. Add SCN Column to Reconcile

Add a column-level check in the recon script that reads source SCN via `ORA_ROWSCN` and compares.

#### 3. Better Alternative — GoldenGate

DMS is convenient but not ideal for large migrations with strict consistency. **GoldenGate** does the SCN-coordinated bridge natively.

## Fix Applied

Switched to **GoldenGate** for the last 5 days of migration. Full load via Data Pump at a known SCN, GoldenGate started from that SCN forward. Zero data mismatches at cutover.

## Lessons Learned

- **DMS full-load + CDC has a data gap by default.** Read the docs carefully.
- **DMS LOB modes matter.** VARCHAR2 > `LobMaxSize` may be treated as LOB and truncated.
- **Reconciliation is not optional** for critical data migrations. Row-level diff on a sample of 5–10% of rows.
- For large / critical migrations, **GoldenGate or Data Pump network-mode** is more predictable than DMS.
- Validate at scale, not just on the smoke test table.

## Related

- [AWS DMS](../38-migrations/aws-dms.md).
- [GoldenGate](../37-goldengate/index.md).
- [Migration Methods](../23-upgrade-migration/migration-methods.md).
- [Data Pump](../21-data-pump/index.md).
