# Pfile

## Purpose

The **parameter file** (`pfile`, aka `init.ora`) is a plain-text file listing initialization parameter values. Historically the default; superseded by [SPFILE](spfile.md) since 9i but still used for:

- Initial database creation.
- Recovery from a corrupt SPFILE.
- Cloning parameters between environments.

## Location

Default lookup:

```
$ORACLE_HOME/dbs/init<SID>.ora   (Linux/Unix)
$ORACLE_HOME/database/INIT<SID>.ORA (Windows)
```

Or wherever `STARTUP PFILE='/path/to/file'` points.

## Format

```
# Comments after #
db_name=PRD
db_unique_name=PRD
memory_target=0
sga_target=32G
sga_max_size=32G
pga_aggregate_target=10G
pga_aggregate_limit=20G
db_files=1024
processes=1000
sessions=1500
control_files=('+DATA/PRD/controlfile/current.ctl','+RECO/PRD/controlfile/current.ctl')
db_recovery_file_dest='+RECO'
db_recovery_file_dest_size=500G
db_block_size=8192
compatible=19.0.0
undo_tablespace='UNDOTBS1'
undo_management='AUTO'
open_cursors=500
```

## Multi-Value Parameters

```
control_files=('+DATA/PRD/current.ctl','+RECO/PRD/current.ctl')
log_archive_dest_1='location=+RECO/archive'
log_archive_dest_2='service=stby async'
```

## Creating a Pfile from SPFILE

```sql
CREATE PFILE = '/tmp/initPRD.ora' FROM SPFILE;
CREATE PFILE FROM SPFILE;   -- default location: $ORACLE_HOME/dbs/init<SID>.ora
```

## Creating an SPFILE from a Pfile

```sql
CREATE SPFILE FROM PFILE;
CREATE SPFILE = '+DATA/PRD/spfilePRD.ora' FROM PFILE = '/tmp/initPRD.ora';
```

## Starting With a Pfile

```sql
STARTUP PFILE='/tmp/initPRD.ora'
```

Ignored if not specified and an SPFILE exists (SPFILE wins).

## When Only Pfile Works

- **Emergency recovery** — SPFILE corrupt or missing.
- **Cloning between environments** — edit text before creating an SPFILE at the target.
- **Testing parameter changes** — start with a modified PFILE without altering the SPFILE.

## Common Layouts

Minimal init.ora for a fresh DB:

```
db_name=NEWDB
db_unique_name=NEWDB
control_files=('/u01/oradata/newdb/control01.ctl')
memory_target=0
sga_target=8G
sga_max_size=8G
pga_aggregate_target=2G
undo_tablespace=UNDOTBS1
compatible=19.0.0
db_block_size=8192
processes=300
```

## Related

- [SPFILE](spfile.md) — binary version.
- `V$PARAMETER` — runtime values.
- `V$SPPARAMETER` — values in SPFILE.

## References

- Oracle Database Administrator's Guide 19c — Initialization Parameter Files
