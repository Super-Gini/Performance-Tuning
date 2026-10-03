# Case: RMAN Recovery Across Incarnations

## Setup

- 19c EE non-RAC.
- Business requirement: restore a table's state as of Aug 1st 09:00 UTC.
- Complication: a `RESETLOGS` happened on Aug 2nd (planned upgrade).
- Current time: Aug 6th.
- RMAN retention: 30 days recovery window; full backups nightly + archive backups every 15 min.

## Understanding the Problem

`RESETLOGS` creates a new **incarnation** of the database. Backups from before RESETLOGS belong to the "previous incarnation". Recovery across incarnations requires:

1. Restore from a backup **before RESETLOGS**.
2. Recover using archive logs from **the previous incarnation**.
3. `ALTER DATABASE OPEN RESETLOGS`.

## Approach

Since only a single table is needed, don't restore the whole database — do a **PITR of just that tablespace** (TSPITR).

## Step 1 — Identify the Incarnation

```
rman target /

RMAN> LIST INCARNATION;
```

Result:

```
List of Database Incarnations
DB Key  Inc Key  DB Name  DB ID       STATUS   Reset SCN   Reset Time
1       1        PRD      1234567890  PARENT   1           2020-01-15 00:00
2       2        PRD      1234567890  ORPHAN   45678900    2024-06-01 14:00
3       3        PRD      1234567890  PARENT   67890120    2025-11-01 09:00
4       4        PRD      1234567890  CURRENT  89012340    2026-08-02 03:00
```

Current is incarnation 4 (Aug 2nd RESETLOGS). Aug 1st data is in incarnation 3.

## Step 2 — Confirm We Have the Right Backups

```
RMAN> LIST BACKUP OF DATABASE COMPLETED AFTER 'SYSDATE-10';
RMAN> LIST BACKUP OF ARCHIVELOG FROM TIME 'SYSDATE-10';
```

Verify the archives from Aug 1st are still available (not deleted).

## Step 3 — Compute Target SCN or Time

We want data as of Aug 1st 09:00 UTC:

```sql
SELECT TIMESTAMP_TO_SCN(TIMESTAMP '2026-08-01 09:00:00') FROM dual;
```

Say result: SCN `89011000`.

## Step 4 — Reset RMAN to the Previous Incarnation

```
RMAN> RESET DATABASE TO INCARNATION 3;
```

RMAN now considers incarnation 3 the "current" for backup purposes.

## Step 5 — Set Up TSPITR (Tablespace Point-in-Time Recovery)

The target table is in tablespace `APP_DATA`. Restore that tablespace to Aug 1st 09:00:

```
RMAN> RECOVER TABLESPACE APP_DATA
      UNTIL SCN 89011000
      AUXILIARY DESTINATION '/u01/tspitr_aux';
```

RMAN internally:

1. Restores the tablespace's datafiles + system + sysaux + undo to `/u01/tspitr_aux`.
2. Creates an auxiliary instance.
3. Recovers to SCN 89011000.
4. Opens the auxiliary DB read-only.
5. Exports the tablespace metadata.
6. Transports the tablespace back into the main DB.

**But** if the tablespace has changed structure between Aug 1st and now (added tables, changed column types), this will conflict. Verify first:

```
RMAN> TRANSPORT TABLESPACE APP_DATA
      TABLESPACE DESTINATION '/tmp/tts_metadata'
      AUXILIARY DESTINATION '/u01/tspitr_aux'
      UNTIL SCN 89011000;
```

`TRANSPORT` is safer for a single-table extract because it doesn't overwrite the current tablespace.

## Alternative — Restore Just the Table

Since 12c, **RMAN table-level recovery** is available:

```
RMAN> RECOVER TABLE APP.ORDERS_STATE
      UNTIL SCN 89011000
      AUXILIARY DESTINATION '/u01/tspitr_aux'
      DATAPUMP DESTINATION '/u01/tspitr_dump'
      DUMP FILE 'orders_state_recover.dmp'
      REMAP TABLE 'APP.ORDERS_STATE':'APP.ORDERS_STATE_AUG1';
```

RMAN:

1. Creates aux instance.
2. Restores tablespace to aux.
3. Recovers to SCN.
4. Exports the table via Data Pump.
5. Imports as `APP.ORDERS_STATE_AUG1` into your live DB.
6. Cleans up aux.

Result: `APP.ORDERS_STATE_AUG1` in the live DB, with Aug 1st content. Zero impact on the live `APP.ORDERS_STATE`.

## Execute

Ran the table-level recovery. Took ~90 minutes for a 8 GB tablespace.

Result:

```
RMAN> RECOVER TABLE APP.ORDERS_STATE
Starting recover at 2026-08-06 14:15:00
using channel ORA_DISK_1

Creating automatic instance, with SID='aBc1'
...
Datapump import complete
Table has been imported as APP.ORDERS_STATE_AUG1
Completed at 2026-08-06 15:47:00
```

## Verify

```sql
SELECT COUNT(*) FROM app.orders_state_aug1;
-- Should match the count from Aug 1st

SELECT MAX(last_modified) FROM app.orders_state_aug1;
-- Should be < Aug 1st 09:00
```

Business team confirms — extract the rows they needed, then drop:

```sql
DROP TABLE app.orders_state_aug1 PURGE;
```

## Return RMAN Context

```
RMAN> RESET DATABASE TO INCARNATION 4;
```

Bring RMAN back to current incarnation so future backups/restores work normally.

## Lessons Learned

- **RESETLOGS creates a new incarnation** — always check `LIST INCARNATION` before cross-time restore.
- **`RECOVER TABLE`** is the modern one-shot; no separate TTS gymnastics.
- **Auxiliary destination** needs 2× the tablespace size in free space.
- **`RESET DATABASE TO INCARNATION`** is critical for backups from before RESETLOGS.
- **Keep archive logs across RESETLOGS** — retention should span at least the "far side" of any planned RESETLOGS.

## Related

- [Recovery](../15-rman/recovery.md).
- [PITR](../15-rman/pitr.md).
- [TSPITR](../15-rman/tspitr.md).
- [RMAN Architecture](../15-rman/rman-architecture.md).
