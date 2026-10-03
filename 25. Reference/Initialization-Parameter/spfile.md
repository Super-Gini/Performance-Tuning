# SPFILE

## Purpose

The **Server Parameter File** — a binary version of the parameter file managed inside the database. `ALTER SYSTEM SET ... SCOPE=SPFILE|MEMORY|BOTH` writes to it. It's the recommended file since 9i.

## Location

Default:

```
$ORACLE_HOME/dbs/spfile<SID>.ora   (Linux/Unix)
$ORACLE_HOME/database/SPFILE<SID>.ORA (Windows)
```

Or ASM location like `+DATA/PRD/spfilePRD.ora`. Typically Oracle chooses SPFILE > SPFILE<SID> > init<SID>.ora at startup, in that order.

## Why SPFILE Over Pfile

- **Persistent `ALTER SYSTEM SET`** — no manual init.ora editing.
- **Auto-recovery** — SPFILE included in RMAN backups.
- **RAC-consistent** — instances read same file if placed on shared storage.
- **`SCOPE=MEMORY|SPFILE|BOTH`** — fine-grained control.

## Reading SPFILE Values

```sql
-- Runtime (may differ if SCOPE=SPFILE was used)
SHOW PARAMETER db_cache_size
SELECT name, value FROM v$parameter WHERE name = 'db_cache_size';

-- What's stored in SPFILE
SELECT name, value FROM v$spparameter WHERE name = 'db_cache_size';

-- Compare
SELECT p.name, p.value in_memory, sp.value in_spfile
FROM   v$parameter p LEFT JOIN v$spparameter sp USING (name)
WHERE  p.value <> NVL(sp.value, '')
   OR (p.value IS NOT NULL AND sp.value IS NULL);
```

## ALTER SYSTEM SCOPE

```sql
-- Persist + apply now
ALTER SYSTEM SET open_cursors = 1000 SCOPE=BOTH;

-- Persist only (needs bounce)
ALTER SYSTEM SET db_cache_size = 20G SCOPE=SPFILE;

-- Apply now, don't persist (temporary)
ALTER SYSTEM SET session_cached_cursors = 200 SCOPE=MEMORY;

-- Reset to default (removes from SPFILE)
ALTER SYSTEM RESET open_cursors SCOPE=BOTH;

-- For RAC-specific (per instance)
ALTER SYSTEM SET open_cursors = 1000 SCOPE=BOTH SID='PRD1';
```

## Creating

```sql
-- From current SPFILE
CREATE PFILE FROM SPFILE;

-- Back to SPFILE from PFILE
CREATE SPFILE FROM PFILE;
CREATE SPFILE = '+DATA/PRD/spfilePRD.ora' FROM PFILE = '/tmp/initPRD.ora';

-- From live memory
CREATE PFILE FROM MEMORY;
```

## Startup Behavior

- `STARTUP` — Oracle looks for `spfile<SID>.ora` first, then `spfile.ora`, then `init<SID>.ora`.
- `STARTUP PFILE='...'` — bypass SPFILE.
- `STARTUP SPFILE='...'` — explicit.

## RAC SPFILE

In RAC the SPFILE typically lives on shared ASM. All instances read/write the same file. Per-instance overrides use `SID='PRD1'` clause.

```sql
-- Different open_cursors per instance
ALTER SYSTEM SET open_cursors=1500 SCOPE=SPFILE SID='PRD1';
ALTER SYSTEM SET open_cursors=1000 SCOPE=SPFILE SID='PRD2';
ALTER SYSTEM SET open_cursors= 800 SCOPE=SPFILE SID='*';   -- default for others
```

## Emergency Recovery

If SPFILE is corrupt:

1. `STARTUP NOMOUNT` fails.
2. Create pfile from RMAN backup: `RMAN> RESTORE SPFILE FROM AUTOBACKUP;`
3. Or manually: `CREATE PFILE FROM SPFILE='/backup/spfilePRD.ora';`.
4. Start with the pfile: `STARTUP PFILE='/tmp/initPRD.ora';`.
5. Recreate the SPFILE: `CREATE SPFILE FROM PFILE;`.

## Related

- [Pfile](pfile.md) — text form.
- `V$PARAMETER`, `V$SPPARAMETER`.

## References

- Oracle Database Administrator's Guide 19c — Server Parameter File
