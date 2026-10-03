# Standby Redo Logs (SRLs)

## Overview

**Standby Redo Logs (SRLs)** are additional redo log groups on the standby database that receive incoming redo from the primary. They're mandatory for **SYNC transport** and **Real-Time Apply**, and strongly recommended for all Data Guard configurations.

Without SRLs, incoming redo goes only to archive logs on the standby — increasing lag and preventing SYNC.

## Sizing and Count

- **Same size** as primary online redo logs.
- **One more group** than the primary has per thread.

Example: primary has 3 log groups per thread → standby needs 4 SRL groups per thread.

The extra group prevents RFS from running out of writable SRLs during heavy activity.

## Creation

On the standby (must be MOUNTED or OPEN R/O):

```sql
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1
  GROUP 11 ('+RECO/prod_dr/srl11.log') SIZE 2G,
  GROUP 12 ('+RECO/prod_dr/srl12.log') SIZE 2G,
  GROUP 13 ('+RECO/prod_dr/srl13.log') SIZE 2G,
  GROUP 14 ('+RECO/prod_dr/srl14.log') SIZE 2G;
```

For multi-thread (RAC primary):

```sql
ALTER DATABASE ADD STANDBY LOGFILE THREAD 2
  GROUP 21 (...) SIZE 2G, ...;
```

## Also on Primary

For future switchover (primary becomes standby), create SRLs on the primary too:

```sql
-- On primary
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1
  GROUP 11 (...) SIZE 2G, ...;
```

## Multiplex

Same practice as online redo logs — 2 members per group on independent storage:

```sql
ALTER DATABASE ADD STANDBY LOGFILE THREAD 1
  GROUP 11 ('+DATA/srl11a.log','+RECO/srl11b.log') SIZE 2G;
```

## Diagnostic Queries

```sql
-- SRL configuration
SELECT group#, thread#, sequence#, bytes/1024/1024 AS mb,
       status, archived
FROM   v$standby_log
ORDER  BY thread#, group#;

-- SRL members
SELECT group#, status, type, member
FROM   v$logfile
WHERE  type = 'STANDBY'
ORDER  BY group#, member;

-- Are SRLs being used?
SELECT process, status, sequence#, block#
FROM   v$managed_standby
WHERE  process = 'RFS';
```

## Common Issues

- **`ORA-16086: standby database does not contain available standby log files`** — Missing SRLs. Add them.
- **All SRLs `ACTIVE`, no `UNASSIGNED`** — Not enough SRLs; primary is generating redo faster than standby can archive. Add one more group.
- **SRL member missing** — Same fix as online redo missing member: drop + re-add.
- **SRLs on slow storage** — RFS writes suffer, lag grows. Move to fast storage.

## Best Practices

1. **Same size** as primary online redo logs.
2. **One more group** than primary per thread.
3. Multiplex SRL members on **independent storage**.
4. Place on fast storage matching primary redo storage.
5. Create SRLs on **primary too** — enables switchover.
6. In RAC, create SRLs for each thread.
7. After resizing primary redo, resize SRLs.
8. Alert on `V$STANDBY_LOG.STATUS` staying `ACTIVE` for too long.

## Interview Questions

1. **Q:** What are SRLs?
   **A:** Standby-side log groups where RFS writes incoming redo. Required for SYNC + real-time apply.

2. **Q:** How many?
   **A:** One more group than primary per thread.

3. **Q:** Same size as primary?
   **A:** Yes — RFS matches to primary log group size.

4. **Q:** Why create SRLs on primary?
   **A:** For future switchover — primary will become standby.

5. **Q:** Storage?
   **A:** Fast; multiplexed on independent devices — mirrors primary redo practice.

## References

- Oracle Data Guard Concepts and Administration 19c — Managing Standby Redo Log Files
- MOS Doc ID 219344.1 — SRL Configuration
- MOS Doc ID 1265700.1 — DG Best Practices
