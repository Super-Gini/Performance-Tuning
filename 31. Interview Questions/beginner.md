# Beginner — Oracle DBA Interview Questions

## Architecture

**Q: What's the difference between a database and an instance?**
A: The database is the physical files (datafiles, control files, redo logs) on disk. The instance is the memory structures (SGA) and background processes that access those files. One database can be opened by one instance (single-instance) or multiple (RAC).

**Q: What is the SGA?**
A: **System Global Area** — shared memory allocated at instance startup. Main components: Buffer Cache, Shared Pool, Redo Log Buffer, Large Pool, Java Pool, Streams Pool. All sessions share it.

**Q: What is the PGA?**
A: **Program Global Area** — per-session memory for sort areas, hash areas, session state. Not shared. Controlled by `PGA_AGGREGATE_TARGET`.

**Q: Name five background processes and what they do.**
A: PMON (process cleanup), SMON (instance recovery + temp cleanup), DBWn (writes dirty buffers), LGWR (writes redo), CKPT (updates control file & datafile headers with checkpoint SCN).

**Q: What is a tablespace?**
A: A logical container for storage. Made of one or more datafiles. Segments (tables, indexes, LOBs) live in tablespaces.

## Startup / Shutdown

**Q: What are the STARTUP stages?**
A: NOMOUNT (SGA allocated, parameter file read), MOUNT (control file opened), OPEN (datafiles & redo opened, users can connect).

**Q: What's the difference between SHUTDOWN IMMEDIATE and SHUTDOWN ABORT?**
A: IMMEDIATE rolls back active transactions and closes cleanly. ABORT terminates the instance; on next startup, SMON does crash recovery. Prefer IMMEDIATE; ABORT only in emergencies.

## Backup / Recovery

**Q: What's the difference between hot and cold backups?**
A: Cold backup — DB shut down, copy files. Hot backup — DB open, `ALTER TABLESPACE BEGIN BACKUP` freezes datafile headers so file copies are consistent. RMAN is the modern approach.

**Q: What is RMAN?**
A: Recovery Manager — Oracle's built-in backup/restore/recover utility. Handles incremental backups, compression, encryption, integration with tape libraries, and Data Guard.

**Q: What is archivelog mode?**
A: DB retains filled redo logs after they're overwritten by copying them to an archive destination. Required for point-in-time recovery and Data Guard.

## Storage

**Q: What is a datafile?**
A: Physical file storing data blocks. One tablespace can have many datafiles (or one big bigfile datafile).

**Q: What is the control file?**
A: Small binary file describing the DB — datafile paths, log switch history, RMAN metadata, current SCN. Multiplex for safety.

**Q: What are online redo logs?**
A: Files LGWR writes to record every change. Cycles through groups (2–4 typically). Filled logs are archived (in archivelog mode).

## Users / Privileges

**Q: Difference between a role and a privilege?**
A: Privilege = an atomic permission (`SELECT ON HR.EMP`, `CREATE TABLE`). Role = a named bundle of privileges/roles that you grant to users.

**Q: What's the difference between GRANT and REVOKE?**
A: GRANT gives, REVOKE takes away. `WITH ADMIN OPTION` (roles) or `WITH GRANT OPTION` (object) lets the grantee re-grant.

**Q: What is SYSDBA?**
A: Privileged connection type — full DB administration, can `STARTUP` / `SHUTDOWN` / recover.

## Basic Tuning

**Q: What is an index?**
A: Sorted data structure (usually B-tree) that speeds up row lookup by indexed columns. Trade-off: slower DML, more space.

**Q: What is a bind variable?**
A: A placeholder in SQL text (`WHERE id = :b1`) instead of a literal. Enables cursor sharing — the same SQL plan is reused for many values.

**Q: What are the two components of SQL execution time?**
A: Parse (hard/soft) + Execute + Fetch. Hard parse is expensive — it involves optimizer work.

**Q: Which view shows currently running SQL?**
A: `V$SESSION` (with `SQL_ID`), joined to `V$SQL` for full text and plan.

## Related

- [Fundamentals](../01-fundamentals/index.md).
- [Intermediate](intermediate.md).
