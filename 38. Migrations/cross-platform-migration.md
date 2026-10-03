# Cross-Platform Migration

## Overview

Moving an Oracle database between different **hardware architectures** (AIX POWER, HP-UX Itanium, Solaris SPARC, Linux x86_64) involves either:

- **Same-endian** platforms → RMAN can move datafiles as-is.
- **Cross-endian** → need conversion (RMAN CONVERT or XTTS).

## Platform Endian Reference

```sql
SELECT platform_id, platform_name, endian_format
FROM   v$transportable_platform
ORDER  BY platform_id;
```

Key ones:

| Platform ID | Platform Name                     | Endian |
| ----------- | --------------------------------- | ------ |
| 1           | Solaris[tm] OE (32-bit)           | Big    |
| 2           | Solaris[tm] OE (64-bit)           | Big    |
| 3           | HP-UX (64-bit)                    | Big    |
| 4           | HP-UX IA (64-bit)                 | Big    |
| 6           | AIX-Based Systems (64-bit)        | Big    |
| 10          | Linux x86 64-bit                  | Little |
| 11          | Linux IA (64-bit)                 | Little |
| 12          | Microsoft Windows x86 64-bit      | Little |
| 13          | Linux x86 64-bit                  | Little |
| 15          | HP-Open VMS                       | Little |
| 16          | Apple Mac OS                      | Big    |
| 17          | Solaris Operating System (x86-64) | Little |
| 18          | IBM Power Based Linux             | Little |
| 19          | Oracle Cloud Infrastructure       | Little |

**Most moves today**: AIX/HP-UX (Big) → Linux (Little).

## Method 1 — RMAN CONVERT (Simplest, Small-Medium)

For a manageable-sized DB (< ~2 TB):

```
-- On source (as SYSDBA)
RMAN> BACKUP DATABASE FORMAT '/u01/backup/%U';

-- Convert to target platform
RMAN> CONVERT DATABASE
      NEW DATABASE PRD_NEW
      TRANSPORT SCRIPT '/tmp/transport_script.sql'
      TO PLATFORM 'Linux x86 64-bit'
      DB_FILE_NAME_CONVERT ('/u01/oradata/PRD','/tmp/converted');

-- Ship converted files + transport_script.sql to target
```

The `transport_script.sql` is a ready-to-run script that:

1. Creates a new spfile.
2. Creates control files.
3. Adds tempfiles.
4. Opens DB with RESETLOGS.

## Method 2 — Full Transportable Export/Import

Combines TTS + Data Pump metadata for a **whole-database** move in one command.

Source (11.2.0.3+):

```sql
-- All user tablespaces → READ ONLY
ALTER TABLESPACE USERS READ ONLY;
ALTER TABLESPACE APP_DATA READ ONLY;
-- ... every user tablespace
```

```bash
expdp system/... full=yes transportable=always version=19 \
      directory=DP_DUMP dumpfile=full_tts.dmp logfile=full_tts.log
```

Ship dump + tablespaces to target.

For **cross-endian**, run RMAN CONVERT on tablespaces first:

```
RMAN> CONVERT TABLESPACE 'USERS','APP_DATA'
      TO PLATFORM 'Linux x86 64-bit'
      FORMAT '/u01/xtts/%U';
```

Target:

```bash
impdp system/... full=yes directory=DP_DUMP dumpfile=full_tts.dmp \
      transport_datafiles='/u01/oradata/tgt/users01.dbf,/u01/oradata/tgt/appdata01.dbf,...' \
      logfile=full_tts_imp.log
```

## Method 3 — XTTS Incremental (Very Large)

For 10+ TB where downtime must be minimized. See [Transportable Tablespaces](transportable-tablespaces.md).

## Pre-Migration Checks

```sql
-- Source and target endian
SELECT platform_id, platform_name, endian_format FROM v$database;

-- Compatible parameter
SHOW PARAMETER compatible

-- Character set
SELECT parameter, value FROM nls_database_parameters
WHERE  parameter IN ('NLS_CHARACTERSET','NLS_NCHAR_CHARACTERSET');

-- Block size
SHOW PARAMETER db_block_size
```

Target must have:

- Same or higher `COMPATIBLE`.
- Same block size (or non-default pools configured).
- Same NLS charset (or target is superset).
- Same national charset.

## Timezone Version

19c ships DSTv32; older sources may be lower. Options:

- Bump target TZ before import.
- Or bump source TZ before starting.

## Handle Non-Tablespace Objects

TTS + full transportable moves the tablespaces, but you still need:

- **Users + roles + grants** — carried by full transportable, or exported/imported separately.
- **Sequences** — not in tablespaces; export separately or in the full_tts dump.
- **DB links** — export/import; passwords may need reset.
- **Directory objects** — recreate on target (paths differ).
- **Scheduler jobs** — export/import.
- **Global tempfiles / undo** — recreate on target.
- **Standby / DG setup** — reconfigure.

## Post-Migration

1. Gather dictionary + fixed stats:
   ```sql
   EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
   EXEC DBMS_STATS.GATHER_FIXED_OBJECTS_STATS;
   ```
2. Recompile invalids:
   ```sql
   @$ORACLE_HOME/rdbms/admin/utlrp.sql
   ```
3. Verify all `dba_registry` components VALID.
4. Apply latest RU.
5. Run application smoke test.
6. Configure new Data Guard, backups, monitoring.

## Common Issues

- **`ORA-19722` block header mismatch** — Missed RMAN CONVERT step. Convert and re-copy.
- **`ORA-19870` datafile version** — Source's `COMPATIBLE` newer than target. Set higher on target.
- **`ORA-01722` invalid number in impdp** — Charset mismatch.
- **Bloated dictionary after import** — Stats not gathered. Gather explicitly.
- **Sequences duplicating** — Sequence numbers reset; may collide with data. Set sequences to max_id + safety after import.

## Related

- [Migration Methods](../23-upgrade-migration/migration-methods.md).
- [Transportable Tablespaces](transportable-tablespaces.md).
- [Data Pump](../21-data-pump/index.md).
