# PITR — Point-in-Time Recovery

## Overview

**Point-in-Time Recovery (PITR)** restores the database (or tablespace) to a specific SCN, time, or log sequence — not the current time. Used when data was corrupted, a bad batch was committed, or a table was accidentally dropped. Any recovery that stops before applying all available redo is a **DBPITR** (Database Point-in-Time Recovery).

PITR is _incomplete_ recovery — after it, you must `OPEN RESETLOGS` because you've branched off the primary redo timeline.

Alternatives: [Flashback Database](../16-flashback/flashback-database.md) for the same use case with lower downtime (if enabled).

## When to Use PITR

- Bad DDL / DML committed and can't be flashed back.
- Wrong data loaded via ETL.
- Application error corrupted a schema.
- Restore for testing to a specific time.

Not for hardware failure — that's complete recovery.

## Prerequisites

- Backup covering the target time.
- All archive logs from backup SCN through target SCN.
- Enough disk for the restored database.

## DBPITR Workflow

Full database rolled back to a target time:

```rman
$ rman target /
RMAN> SHUTDOWN IMMEDIATE;
RMAN> STARTUP MOUNT;

RUN {
  SET UNTIL TIME "TO_DATE('2026-08-06 14:30:00','YYYY-MM-DD HH24:MI:SS')";
  RESTORE DATABASE;
  RECOVER DATABASE;
  ALTER DATABASE OPEN RESETLOGS;
}
```

`SET UNTIL` variants:

- `SET UNTIL TIME '...'`
- `SET UNTIL SCN 1234567890`
- `SET UNTIL SEQUENCE 1050 THREAD 1`

## SCN-based

More precise than time — no ambiguity around DST or clock skew:

```
SELECT current_scn FROM v$database;    -- capture BEFORE the incident
```

Then later:

```rman
SET UNTIL SCN 5678901234;
```

## Time-based

```rman
SET UNTIL TIME "TO_DATE('2026-08-06 14:30:00','YYYY-MM-DD HH24:MI:SS')";
```

Or for NLS-portable:

```rman
SET UNTIL TIME "TIMESTAMP '2026-08-06 14:30:00'";
```

## Sequence-based

When you know the last-good log sequence:

```rman
SET UNTIL SEQUENCE 1050 THREAD 1;   -- stop before sequence 1050
```

## After PITR — Incarnation Management

`OPEN RESETLOGS` creates a new database incarnation. Older backups on the previous incarnation may still be usable for further PITRs within the same incarnation.

```rman
LIST INCARNATION;

-- Reset to a specific incarnation if you need to PITR back through it
RESET DATABASE TO INCARNATION 2;
```

## PITR to Alternate Location

For "extract data without touching production": use [Duplicate Database](duplicate-database.md) with `UNTIL TIME`.

## Diagnostic Queries

```sql
-- Current SCN
SELECT current_scn FROM v$database;

-- Approximate SCN for a past time (24-hour window)
SELECT TIMESTAMP_TO_SCN(SYSTIMESTAMP - INTERVAL '1' HOUR) FROM dual;

-- Or reverse
SELECT SCN_TO_TIMESTAMP(1234567890) FROM dual;

-- Archive logs required for PITR
SELECT thread#, sequence#, first_time, next_time
FROM   v$archived_log
WHERE  next_time > TO_DATE('2026-08-06 14:00','YYYY-MM-DD HH24:MI')
   AND first_time < TO_DATE('2026-08-06 14:30','YYYY-MM-DD HH24:MI')
ORDER  BY thread#, sequence#;
```

## Common Issues

- **Missing archive logs** — Cannot PITR past the gap. Restore from backup or accept partial recovery.
- **`ORA-01152: file N was not restored from a sufficiently old backup`** — Wrong backup selected; restore an older backup.
- **`RMAN-06196: cannot ...` — inconsistent SCN targets** — Target is too early or too late; adjust `SET UNTIL`.
- **`OPEN RESETLOGS` fails** — Recovery didn't complete; check `V$RECOVER_FILE`.
- **Standby diverges** — PITR on primary breaks standby. Recreate or flashback standby.

## Best Practices

1. **Capture current SCN before risky operations** for exact PITR target.
2. Use **SCN** for maximum precision.
3. Test PITR on a scratch host quarterly.
4. **Time zone caveat** — always fully qualify times to avoid DST confusion.
5. Enable **Flashback Database** to avoid the full RESTORE penalty when possible.
6. Keep enough archive logs on hand for max PITR window.
7. Coordinate PITR on Data Guard: standby needs matching flashback / recreation.
8. Communicate `OPEN RESETLOGS` implications — backups after the reset are on a new incarnation.
9. Right after PITR: take a fresh L0 backup on the new incarnation.

## Interview Questions

1. **Q:** What is DBPITR?
   **A:** Database Point-in-Time Recovery — restore and recover to a specific SCN/time/sequence, then `OPEN RESETLOGS`.

2. **Q:** Time, SCN, or Sequence?
   **A:** SCN is most precise. Time simplest for humans. Sequence useful when you know last-good log.

3. **Q:** Why `OPEN RESETLOGS`?
   **A:** PITR is incomplete recovery; a new incarnation is required to prevent redo confusion.

4. **Q:** Flashback vs PITR?
   **A:** Flashback is faster (no restore); requires Flashback Database enabled. PITR restores from backup.

5. **Q:** Impact on Data Guard?
   **A:** Standby diverges. Recreate or flashback standby to matching SCN.

6. **Q:** What if you need to go back before the last RESETLOGS?
   **A:** `RESET DATABASE TO INCARNATION <n>` to a previous incarnation, then PITR.

## References

- Oracle Database Backup and Recovery User's Guide 19c
- MOS Doc ID 388422.1 — RMAN Recovery
- MOS Doc ID 1521286.1 — DBPITR
