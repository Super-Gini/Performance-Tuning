# Password File

## Overview

The **password file** (`orapw<SID>` on Unix, `PWD<SID>.ora` on Windows) stores credentials for **remote SYSDBA**, **SYSOPER**, **SYSBACKUP**, **SYSDG**, **SYSKM**, and **SYSRAC** connections. Without it, only local OS-authenticated (`sqlplus / as sysdba`) connections work.

Location: `$ORACLE_HOME/dbs/orapw<SID>` (or `$ORACLE_HOME/database/PWD<SID>.ora` on Windows).

Required for:

- Remote SYSDBA (`sqlplus sys/pwd@service as sysdba`).
- Data Guard broker.
- RMAN over TCP.
- RAC (each instance).

## Architecture

```mermaid
flowchart LR
    Client -->|remote SYSDBA connect| Lsnr[Listener]
    Lsnr --> Inst[Instance]
    Inst -->|verify| PWD[orapw&lt;SID&gt;]
    PWD --> Users[Privileged users:<br/>SYS, SYSBACKUP, ...]
```

## Internal Working

### Creation

```bash
# Linux
orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID \
       password=StrongPassword \
       entries=10 \
       format=12 \
       ignorecase=n \
       force=y
```

Parameters:

- `file` — path (must be `orapw<SID>`).
- `password` — SYS password.
- `entries` — max privileged users (SYS + others granted SYSDBA-like privs).
- `format` — `10`, `12`, `12.2` — hash format version.
- `ignorecase` — password case sensitivity.
- `format=12` — SHA-2 512, recommended default.

### Add Users

Grant SYSDBA to another user:

```sql
GRANT SYSDBA TO backupdba_user;
```

The password file entry is created automatically.

Then remotely:

```bash
sqlplus backupdba_user/pwd@service as sysbackup
```

### List Password File Entries

```sql
SELECT * FROM v$pwfile_users;
```

Columns:

- `USERNAME` — user
- `SYSDBA`, `SYSOPER`, `SYSASM`, `SYSBACKUP`, `SYSDG`, `SYSKM`, `SYSRAC` — privileges granted

### REMOTE_LOGIN_PASSWORDFILE Parameter

| Value       | Meaning                                               |
| ----------- | ----------------------------------------------------- |
| `NONE`      | No password file used; only OS auth allowed           |
| `EXCLUSIVE` | Password file specific to this instance (default)     |
| `SHARED`    | Multiple databases share one password file (uncommon) |

`EXCLUSIVE` is standard.

### Password File Restrictions

- **Case-sensitive by default** (`ignorecase=n`) since 11g.
- Cannot be modified while instance is running (add/change users, sure; but binary format changes need shutdown).
- On RAC with shared ORACLE_HOME, one password file works across all nodes (on ASM: `orapw+ASM`).

### Recreate

If lost or corrupt:

```bash
orapwd file=$ORACLE_HOME/dbs/orapw$ORACLE_SID \
       password=SysPwd entries=10 format=12 force=y

-- Re-grant SYSDBA to non-SYS users
SQL> GRANT SYSDBA TO backupdba_user;
```

## Components

| Component        | Purpose               |
| ---------------- | --------------------- |
| Password file    | Credential store      |
| `orapwd` utility | Creation and reformat |
| SYS password     | Master credential     |

## Important Parameters

| Parameter                   | Purpose                   |
| --------------------------- | ------------------------- |
| `remote_login_passwordfile` | EXCLUSIVE / NONE / SHARED |
| `sec_case_sensitive_logon`  | Deprecated; always TRUE   |

## Important Views

| View             | Purpose                                      |
| ---------------- | -------------------------------------------- |
| `V$PWFILE_USERS` | Users in password file with their privileges |

## Diagnostic Queries

```sql
-- Who has SYSDBA and family?
SELECT username, sysdba, sysoper, sysasm, sysbackup, sysdg, sysrac, sysdba, con_id
FROM   v$pwfile_users;

-- Confirm parameter
SHOW PARAMETER remote_login_passwordfile
```

## Common Operations

### Change SYS password

```sql
ALTER USER sys IDENTIFIED BY "NewStrongPwd";
```

Automatically updates the password file entry.

### Grant SYSBACKUP

```sql
CREATE USER rman_ops IDENTIFIED BY pwd;
GRANT CREATE SESSION TO rman_ops;
GRANT SYSBACKUP TO rman_ops;
```

`V$PWFILE_USERS` should now include `RMAN_OPS` with `SYSBACKUP = TRUE`.

### Revoke

```sql
REVOKE SYSBACKUP FROM rman_ops;
```

Entry removed automatically.

### Move / Copy

For RAC, if not on ASM: copy `orapw<SID>` to each node's `$ORACLE_HOME/dbs/`.

For ASM-stored password file (recommended for RAC):

```bash
srvctl modify database -db prod -pwfile '+DATA/orapwprod'
```

## Common Issues

- **`ORA-01017` for remote SYS** — Password file missing/wrong; recreate with `orapwd`.
- **`ORA-01031: insufficient privileges` on `AS SYSDBA`** — User not in password file OR OS user not in `dba` group for local auth.
- **`ORA-01994: GRANT failed`** — Password file full (`entries` cap).
- **RAC — password file out of sync** — Some nodes have old copy. Copy latest to all nodes or move to ASM.

## Best Practices

1. **Use `format=12`** — SHA-2 512 hashes.
2. **`REMOTE_LOGIN_PASSWORDFILE=EXCLUSIVE`** — the default; leave it.
3. **Store on ASM in RAC** — one file, all nodes share.
4. Set `entries` generously (10–20) — recreating is minor pain but plan ahead.
5. **Separation of duty** — grant SYSBACKUP (RMAN), SYSDG (DG), SYSKM (TDE) instead of SYSDBA.
6. Rotate SYS password quarterly (or per policy).
7. Backup the password file (via OS or ASM copy).
8. Alert on missing or unreadable password file at instance startup.
9. Never `chmod 777` — password file needs to be `oracle:oinstall 0640`.

## Interview Questions

1. **Q:** What is the password file for?
   **A:** Stores credentials for remote SYSDBA, SYSOPER, and family privileges (SYSBACKUP, SYSDG, SYSKM, SYSRAC).

2. **Q:** Where is it?
   **A:** `$ORACLE_HOME/dbs/orapw<SID>` (Linux) or `%ORACLE_HOME%\database\PWD<SID>.ora` (Windows).

3. **Q:** How do you create/recreate?
   **A:** `orapwd file=... password=... entries=10 format=12 force=y`.

4. **Q:** How do you list privileged users?
   **A:** `SELECT * FROM v$pwfile_users;`.

5. **Q:** RAC — one password file per node?
   **A:** Historically yes. Best practice now: store in ASM, all nodes share.

6. **Q:** What does `REMOTE_LOGIN_PASSWORDFILE=NONE` do?
   **A:** Disables password file — only local OS-authenticated `AS SYSDBA` works.

7. **Q:** Difference between SYSDBA and SYSOPER?
   **A:** SYSDBA: full — startup/shutdown, DDL, data access. SYSOPER: startup/shutdown/backup, no data access.

## References

- Oracle Database Administrator's Guide 19c — Password File Authentication
- Oracle Database Security Guide 19c
- MOS Doc ID 185703.1 — Password File Management
