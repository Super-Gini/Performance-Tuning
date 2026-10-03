# Case: Recurring ORA-00600 — From Symptom to Bug Fix

## Setup

- 19c EE (19.15), single-instance, on-prem.
- Alert: `ORA-00600 [kdscat_leaf_2]` seen 3× in the last week.
- No user-visible impact yet (background process trace), but Oracle recommends investigating any 600.

## Investigation

### Step 1 — Alert Log

```bash
grep "ORA-00600" $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log | tail -20
```

```
Errors in file /u01/app/oracle/diag/rdbms/prd/PRD1/trace/PRD1_j001_15234.trc  (incident=45678):
ORA-00600: internal error code, arguments: [kdscat_leaf_2], [], [], [], [], [], [], [], [], [], [], []
Errors in file .../PRD1_j001_15234.trc  (incident=45680):
ORA-00600: internal error code, arguments: [kdscat_leaf_2], [], [], [], [], [], [], [], [], [], [], []
```

All occurrences involve `j001` = a Scheduler slave. Always the same function `kdscat_leaf_2`.

### Step 2 — Trace File

```bash
less $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/PRD1_j001_15234.trc
```

Key section:

```
Current SQL:
DECLARE
  ...
BEGIN
  DBMS_STATS.GATHER_TABLE_STATS('APP','ORDERS',...);
END;

kdscat_leaf_2 call stack:
  kdscat_leaf_2 <-  kdscat_int <-  kdscat_gather <- kglcov <- pfrfd
```

Stack shows the error fires in `kdscat_*` — data scan / gather. Failure happens during `DBMS_STATS.GATHER_TABLE_STATS` on `APP.ORDERS`.

### Step 3 — MOS Search

Search MOS: `ORA-00600 [kdscat_leaf_2]`.

Match: **Bug 32521906** — Fixed in 19.17 RU. Symptoms match exactly (index corruption on partitioned table stats gather).

### Step 4 — Verify Bug Applies

The bug description says:

- Affects partitioned tables with unusable indexes.
- Fired during `DBMS_STATS` global stats gather.
- Fixed in 19.17.
- Workaround: rebuild the index, or set `_fix_control='32521906:0'` (disable buggy code path).

Check if any indexes on `ORDERS` are unusable:

```sql
SELECT owner, index_name, status, partition_name, subpartition_name
FROM   dba_indexes
WHERE  table_name = 'ORDERS' AND status <> 'VALID';

-- or per-partition
SELECT owner, index_name, partition_name, status
FROM   dba_ind_partitions
WHERE  index_name IN (SELECT index_name FROM dba_indexes WHERE table_name='ORDERS')
   AND status <> 'USABLE';
```

Result: 3 partitions of `ORDERS_STATUS_IDX` are UNUSABLE. Presumably from a truncate partition that skipped `UPDATE INDEXES`.

### Root Cause Confirmed

Match: bug's preconditions are present. Bug's fix version is 19.17. We're on 19.15.

## Fix — Multiple Options

### Option A — Rebuild Unusable Indexes

Immediate:

```sql
ALTER INDEX app.orders_status_idx REBUILD PARTITION P_2026_07 ONLINE;
ALTER INDEX app.orders_status_idx REBUILD PARTITION P_2026_08 ONLINE;
ALTER INDEX app.orders_status_idx REBUILD PARTITION P_2026_09 ONLINE;
```

`GATHER_TABLE_STATS` no longer hits the corrupt path.

### Option B — Disable Fix Control (Workaround)

For future occurrences even if new unusable indexes appear:

```sql
ALTER SYSTEM SET "_fix_control" = '32521906:0' SCOPE=BOTH;
```

### Option C — Apply the RU with the Fix

Apply 19.17 or newer:

```bash
opatch apply /patches/19.17.0.0.0/
datapatch -verbose
```

Fix is in the code, no workaround needed.

## Decision

- **Option A** (rebuild unusable indexes) — immediate, no other risk. **Done**.
- **Option C** (RU upgrade) — planned for the next maintenance window (2 weeks out).
- Option B kept as a backup if new unusable indexes appear.

## Package for Support Verification (Just in Case)

```bash
adrci
adrci> ips create package incident 45678
adrci> ips add file /u01/.../alert_PRD1.log package 4
adrci> ips generate package 4 in /tmp
```

Upload the ZIP to the SR. Oracle Support confirms our diagnosis.

## Verify Fix

Next week, no ORA-00600 in the alert log:

```sql
SELECT COUNT(*) FROM v$diag_alert_ext
WHERE  message_text LIKE 'ORA-00600 [kdscat_leaf_2]%'
   AND originating_timestamp > SYSDATE - 7;
```

Result: 0.

## Lessons Learned

- **Every ORA-00600 goes to MOS first** — most are known bugs with documented fixes.
- **Bracketed function name is the key** — search for it exactly, don't paraphrase.
- **UNUSABLE indexes are a smell** — often accompany bugs like this. Alert on them:
  ```sql
  SELECT COUNT(*) FROM dba_indexes WHERE status <> 'VALID';
  SELECT COUNT(*) FROM dba_ind_partitions WHERE status <> 'USABLE';
  ```
- **`_fix_control` is a real tool** — dangerous but sometimes the fastest fix.
- **Stay current with RUs** — most 600s have been fixed by the time you hit them; you're at the wrong version.

## Related

- [ORA-00600](../26-errors/ora-00600.md).
- [Trace Files](../24-monitoring/trace-files.md).
- [Underscore Parameters](../25-reference/hidden-parameters/underscore-parameters.md).
- [Release Updates](../22-patching/release-updates.md).
