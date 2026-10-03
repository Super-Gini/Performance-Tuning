# GoldenGate Troubleshooting

Common issues by symptom, with fix path.

## Symptom: Extract Abends on Startup

### Case 1: OGG-01031 — Archived log not found

Extract needs a specific SCN or archived log that's been purged.

```
GGSCI> INFO EXTRACT e_prd, DETAIL
```

Fix:

- Restore the archived log from RMAN backup.
- Or `ALTER EXTRACT e_prd BEGIN NOW` (loses the gap; may need to reinit target).

Prevent: increase archive retention to cover Extract's max lag + 2x safety.

### Case 2: OGG-01161 — Missing supplemental logging

```
GGSCI> DBLOGIN USERID ggadmin PASSWORD <pw>
GGSCI> ADD SCHEMATRANDATA APP
```

### Case 3: OGG-00868 — Table not defined

Table added but no `TABLE APP.NEW_TABLE;` in extract param. Add it, restart.

## Symptom: Replicat Abends on Startup

### Case 1: OGG-00519 — ORA-00001 (unique constraint)

Common during initial synchronization: a row already exists.

Fix:

- `HANDLECOLLISIONS` in Replicat param — converts INSERT to UPDATE on duplicates.
- After catch-up, remove `HANDLECOLLISIONS`.

### Case 2: Target row missing for UPDATE

`REPERROR (1403, DISCARD)` writes to discard file for later analysis.

### Case 3: Column length exceeded

Source column is larger than target. Extend target column.

## Symptom: Lag Growing

```
GGSCI> LAG EXTRACT e_prd
GGSCI> LAG REPLICAT r_prd
```

### Extract lag:

- Source redo generation rate > Extract capture rate.
- Integrated Extract can use more parallelism: `TRANLOGOPTIONS INTEGRATEDPARAMS (max_sga_size 1024)`.
- Classic Extract: increase `PROCESSTHREADS` or migrate to Integrated.

### Data Pump lag:

- Network is slow. Check `iftop` between source and target.
- Compress trail files: `RMTHOST ..., COMPRESS` (or use OGG native compression).

### Replicat lag:

- Target DB slow. Look at target's wait events.
- Increase Integrated Replicat parallelism.
- Consider Coordinated Replicat with more threads.

## Symptom: Data Discrepancy on Target

Reconciliation shows source and target diverging.

### Steps

1. **Row count check** on both sides.
2. **Checksum** critical columns.
3. **Discard file** — check for skipped rows:
   ```
   GGSCI> VIEW REPORT r_prd
   ```
4. **Position** in trail — is Replicat caught up?
5. **Bidirectional setup?** — `TRANLOGOPTIONS EXCLUDEUSER ggadmin` prevents loopback.

Compare with `DBMS_COMPARISON` (12c+):

```sql
BEGIN
  DBMS_COMPARISON.CREATE_COMPARISON(
    comparison_name => 'CMP_ORDERS',
    schema_name     => 'APP',
    object_name     => 'ORDERS',
    dblink_name     => 'TARGET_DB');
END;
/

DECLARE
  s DBMS_COMPARISON.COMPARISON_TYPE;
BEGIN
  DBMS_COMPARISON.COMPARE('CMP_ORDERS', s);
END;
/
```

## Symptom: DDL Not Replicating

- Confirm `DDL INCLUDE MAPPED` in both Extract and Replicat.
- Classic Extract requires DDL trigger install; Integrated does not.
- Verify:
  ```
  GGSCI> STATS EXTRACT e_prd, DDL
  ```

## Symptom: Trail Files Fill Disk

Extract writes trails faster than Data Pump / Replicat can drain.

Options:

- Speed up Replicat.
- Add compression: `RMTHOST target_host MGRPORT 7809, COMPRESS`.
- Enlarge trail file size cap: `MEGABYTES 1024` — fewer files, easier to manage.
- Purge old trails: `PURGEOLDEXTRACTS`.

Automatic purge in Manager param:

```
PURGEOLDEXTRACTS ./dirdat/*, USECHECKPOINTS, MINKEEPHOURS 4, FREQUENCYMINUTES 60
```

## Symptom: Manager Won't Stay Running

```
GGSCI> INFO MANAGER
```

`STOPPED` or restarts constantly?

- Port conflict: `PORT` in mgr.prm collides with another service.
- Permissions: log directory not writable.
- Deadman timer: `PURGEOLDEXTRACTS` misconfigured.

## Symptom: Integrated Extract Slow

Enable `MAX_SGA_SIZE`:

```
TRANLOGOPTIONS INTEGRATEDPARAMS(max_sga_size 2048)
```

Or check LogMiner performance directly:

```sql
SELECT * FROM v$logmnr_stats;
```

## Diagnostic Files

- **`$OGG_HOME/ggserr.log`** — event log; start here.
- **`$OGG_HOME/dirrpt/<process>.rpt`** — per-process report.
- **`$OGG_HOME/dirdsc/<process>.dsc`** — discard file (skipped rows).
- **`ADR`** on source and target — Oracle DB traces.

## Verify Environment After Restart

```
GGSCI> INFO ALL

Program     Status      Group       Lag at Chkpt  Time Since Chkpt
MANAGER     RUNNING
EXTRACT     RUNNING     E_PRD       00:00:03      00:00:07
EXTRACT     RUNNING     P_PRD       00:00:00      00:00:07
REPLICAT    RUNNING     R_PRD       00:00:05      00:00:07
```

## Best Practices

- Monitor lag continuously — alert at 5+ minutes.
- Trail file purge job — never let disk fill.
- Weekly `INFO ALL` snapshot in ops repository.
- Automate the deployment (Ansible / Terraform) — GG configs are text and version-controllable.
- Test failover scenarios in staging.
- Keep source & target DB versions within GG support matrix.

## Related

- [Architecture](architecture.md).
- [Extract](extract.md).
- [Replicat](replicat.md).
