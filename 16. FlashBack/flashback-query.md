# Flashback Query

## Overview

**Flashback Query** returns the state of data as it existed at a past point in time — a query against undo. The classic use: "show me what the row looked like at 3 PM yesterday." No changes to the database; just a read-consistency SCN older than _current SCN_.

Requires `UNDO_RETENTION` to cover the flashback window (or `RETENTION GUARANTEE`).

## Variants

- **`AS OF SCN`** — Query at a specific SCN.
- **`AS OF TIMESTAMP`** — Same, expressed as time.
- **Version Query (`VERSIONS BETWEEN`)** — All versions of a row over a range.

## Examples

### AS OF TIMESTAMP

```sql
SELECT * FROM hr.employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '1' HOUR)
WHERE  employee_id = 100;
```

### AS OF SCN

```sql
SELECT * FROM hr.employees AS OF SCN 1234567890
WHERE  employee_id = 100;
```

### Version Query — All Versions

```sql
SELECT VERSIONS_XID AS transaction_id,
       VERSIONS_STARTTIME AS start_time,
       VERSIONS_ENDTIME AS end_time,
       VERSIONS_OPERATION AS op,
       employee_id, salary
FROM   hr.employees
VERSIONS BETWEEN TIMESTAMP
  SYSTIMESTAMP - INTERVAL '1' DAY AND SYSTIMESTAMP
WHERE  employee_id = 100
ORDER  BY start_time;
```

Columns:

- `VERSIONS_XID` — transaction ID.
- `VERSIONS_OPERATION` — I (insert), U (update), D (delete).
- `VERSIONS_STARTTIME` / `_ENDTIME` — visibility window.
- `VERSIONS_STARTSCN` / `_ENDSCN` — SCN equivalents.

### Join with `V$TRANSACTION` for audit

```sql
SELECT v.*, DBMS_FLASHBACK.GET_SYSTEM_CHANGE_NUMBER FROM v$transaction v;
```

## Undo Retention Requirement

Flashback query works only within undo retention:

```sql
SELECT tuned_undoretention FROM v$undostat WHERE rownum = 1;
```

Beyond that: `ORA-01555: snapshot too old` or `ORA-01466: unable to read data - table definition changed`.

For long retention, use [Flashback Data Archive](../03-instance-architecture/processes/fbda.md).

## Common Uses

- Fix an accidental delete / update (SELECT AS OF, INSERT back).
- Audit changes over a time window.
- Compare row states between two points in time.
- Populate a Data Pump export at a specific SCN.

### Copy prior state back

```sql
INSERT INTO hr.employees
SELECT * FROM hr.employees AS OF TIMESTAMP (SYSTIMESTAMP - INTERVAL '30' MINUTE)
WHERE  employee_id IN (100, 101);
```

Or use MERGE / UPDATE from AS OF.

## Diagnostic Queries

```sql
-- Convert time to SCN
SELECT TIMESTAMP_TO_SCN(SYSTIMESTAMP - INTERVAL '1' HOUR) FROM dual;

-- Convert SCN to time
SELECT SCN_TO_TIMESTAMP(1234567890) FROM dual;

-- Current SCN
SELECT current_scn FROM v$database;

-- Longest query (a hint at undo pressure)
SELECT maxquerylen FROM v$undostat WHERE rownum=1;
```

## Common Issues

- **`ORA-01555`** — Undo not retained long enough. Enlarge undo or enable `RETENTION GUARANTEE`.
- **`ORA-01466: unable to read data - table definition changed`** — DDL happened between the target SCN and now.
- **Not working after truncate / drop** — TRUNCATE and DROP are DDL; not undo-tracked. Use recycle bin (drop) or FDA.
- **SCN vs time inaccuracy** — Multiple SCNs per second; time-based flashback rounds to nearest.

## Best Practices

1. Set **`UNDO_RETENTION`** to cover your flashback window (hours or a day for OLTP).
2. **`RETENTION GUARANTEE`** if certain flashbacks are business-critical.
3. Use **SCN** for exact restore points (`SELECT current_scn ...` before risky ops).
4. For long-term retention, adopt [Flashback Data Archive](../03-instance-architecture/processes/fbda.md).
5. Test flashback query on important tables — verify undo covers realistic incident windows.
6. Version Query for audit-lite over recent windows.
7. Do not depend on flashback query for regulatory compliance — use FDA.

## Interview Questions

1. **Q:** What is Flashback Query?
   **A:** Query returning data as it existed at a past SCN/time; reads undo.

2. **Q:** SCN vs TIMESTAMP?
   **A:** SCN is exact. TIMESTAMP is approximate (multiple SCNs per second).

3. **Q:** Retention window?
   **A:** Bounded by `UNDO_RETENTION` and undo tablespace size.

4. **Q:** VERSIONS BETWEEN?
   **A:** Returns all row versions between two SCNs/times — useful for audit.

5. **Q:** What can go wrong?
   **A:** `ORA-01555` if undo expired; `ORA-01466` if DDL happened.

6. **Q:** DDL / TRUNCATE — flashback query?
   **A:** No — DDL isn't undo-tracked. Use recycle bin for DROP, FDA for long retention.

## References

- Oracle Database Development Guide 19c — Flashback
- MOS Doc ID 258670.1 — Flashback Query
- MOS Doc ID 269814.1 — Undo Retention
