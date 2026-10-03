# Reference

Quick-lookup reference material — the pages you'll `Ctrl-F` into during triage. Concise formats: purpose, key columns, common queries, gotchas.

## Contents

### Background Processes

| Page                                                                                 | Purpose                    |
| ------------------------------------------------------------------------------------ | -------------------------- |
| [Background Process Reference](background-processes/background-process-reference.md) | Complete process inventory |
| [PMON](background-processes/pmon.md)                                                 | Process Monitor            |
| [SMON](background-processes/smon.md)                                                 | System Monitor             |
| [DBWn](background-processes/dbwn.md)                                                 | Database Writer            |
| [LGWR](background-processes/lgwr.md)                                                 | Log Writer                 |
| [CKPT](background-processes/ckpt.md)                                                 | Checkpoint                 |
| [ARCn](background-processes/arcn.md)                                                 | Archiver                   |
| [RECO](background-processes/reco.md)                                                 | Recoverer                  |
| [MMON](background-processes/mmon.md)                                                 | Manageability Monitor      |
| [MMNL](background-processes/mmnl.md)                                                 | Manageability Monitor Lite |
| [MMAN](background-processes/mman.md)                                                 | Memory Manager             |
| [VKTM](background-processes/vktm.md)                                                 | Virtual Keeper of Time     |
| [LREG](background-processes/lreg.md)                                                 | Listener Registration      |
| [FBDA](background-processes/fbda.md)                                                 | Flashback Data Archiver    |

### V$ Views

| Page                                                            | Purpose                       |
| --------------------------------------------------------------- | ----------------------------- |
| [V$INSTANCE](v-views/v-instance.md)                             | Current instance              |
| [V$DATABASE](v-views/v-database.md)                             | Current database              |
| [V$SESSION](v-views/v-session.md)                               | All sessions                  |
| [V$SESSION_WAIT](v-views/v-session-wait.md)                     | Current waits                 |
| [V$PROCESS](v-views/v-process.md)                               | OS processes                  |
| [V$SQL](v-views/v-sql.md)                                       | Shared SQL statements         |
| [V$ACTIVE_SESSION_HISTORY](v-views/v-active-session-history.md) | ASH samples                   |
| [V$SYSTEM_EVENT](v-views/v-system-event.md)                     | System-wide wait event totals |
| [V$LOG](v-views/v-log.md)                                       | Redo log groups               |
| [V$ARCHIVED_LOG](v-views/v-archived-log.md)                     | Archived redo                 |
| [V$LOCK](v-views/v-lock.md)                                     | Locks                         |
| [V$LOCKED_OBJECT](v-views/v-locked-object.md)                   | Locked objects                |
| [V$UNDOSTAT](v-views/v-undostat.md)                             | Undo statistics               |
| [V$TEMPSEG_USAGE](v-views/v-tempseg-usage.md)                   | Temp segment usage            |
| [V$RECOVERY_FILE_DEST](v-views/v-recovery-file-dest.md)         | FRA state                     |
| [V$RMAN_STATUS](v-views/v-rman-status.md)                       | RMAN operations               |
| [V$DATAGUARD_STATS](v-views/v-dataguard-stats.md)               | Data Guard lag/status         |

### GV$ Views

| Page                                 | Purpose                |
| ------------------------------------ | ---------------------- |
| [GV$SESSION](gv-views/gv-session.md) | Cluster-wide sessions  |
| [GV$PROCESS](gv-views/gv-process.md) | Cluster-wide processes |
| [GV$SQL](gv-views/gv-sql.md)         | Cluster-wide SQL       |

### DBA Views

| Page                                            | Purpose       |
| ----------------------------------------------- | ------------- |
| [DBA_USERS](dba-views/dba-users.md)             | User accounts |
| [DBA_ROLES](dba-views/dba-roles.md)             | Roles         |
| [DBA_TABLESPACES](dba-views/dba-tablespaces.md) | Tablespaces   |
| [DBA_DATA_FILES](dba-views/dba-data-files.md)   | Datafiles     |
| [DBA_SEGMENTS](dba-views/dba-segments.md)       | Segments      |

### Initialization Parameters

| Page                                                                      | Purpose               |
| ------------------------------------------------------------------------- | --------------------- |
| [Memory Parameters](initialization-parameters/memory-parameters.md)       | SGA / PGA sizing      |
| [Optimizer Parameters](initialization-parameters/optimizer-parameters.md) | CBO knobs             |
| [Undo Parameters](initialization-parameters/undo-parameters.md)           | Undo tuning           |
| [Pfile](initialization-parameters/pfile.md)                               | init.ora format       |
| [Spfile](initialization-parameters/spfile.md)                             | Server parameter file |

### Hidden Parameters

| Page                                                                | Purpose                            |
| ------------------------------------------------------------------- | ---------------------------------- |
| [Underscore Parameters](hidden-parameters/underscore-parameters.md) | `_` parameters and how to see them |

### Wait Events

| Page                                      | Purpose                    |
| ----------------------------------------- | -------------------------- |
| [Commit](wait-events/commit.md)           | `log file sync` and family |
| [Concurrency](wait-events/concurrency.md) | Latch, mutex, enqueue      |
| [Network](wait-events/network.md)         | SQL\*Net, DB link          |
| [System I/O](wait-events/system-io.md)    | Background IO              |
| [User I/O](wait-events/user-io.md)        | `db file %` waits          |
