# Auditing

## Overview

**Auditing** records what happens in the database — successful logins, DDL, privilege changes, and specific SQL statements. Oracle has two audit frameworks:

1. **Traditional auditing** (pre-12c) — `AUDIT` statement writes to `SYS.AUD$` or OS files. Still supported but deprecated.
2. **Unified Auditing** (12c+) — single audit trail in `AUDSYS.AUD$UNIFIED` with policy-driven capture. Recommended.

Every production database should have some audit trail — regulatory compliance, forensics, and detection of insider threats depend on it.

## Which Framework?

- **New databases**: Unified Auditing (see [Unified Audit](unified-audit.md)).
- **Migrating from older**: Convert traditional to unified. Mixed mode (Unified + traditional) is supported but confusing — pick one.
- **Enable Unified**: relink oracle binary with `uniaud_on`:

```bash
cd $ORACLE_HOME/rdbms/lib
make -f ins_rdbms.mk uniaud_on ioracle
```

Once enabled, cannot disable without another relink.

## Traditional Auditing (Deprecated)

### Enable

Set `audit_trail`:

| Value          | Destination            |
| -------------- | ---------------------- |
| `NONE`         | Off                    |
| `DB`           | `SYS.AUD$`             |
| `DB,EXTENDED`  | + SQL text + bind vars |
| `OS`           | OS file                |
| `XML`          | OS XML file            |
| `XML,EXTENDED` | + SQL text             |

```sql
ALTER SYSTEM SET audit_trail = 'DB,EXTENDED' SCOPE=SPFILE;
-- restart
```

### Audit Statements

```sql
-- Session (login/logout)
AUDIT SESSION;
AUDIT SESSION WHENEVER NOT SUCCESSFUL;

-- Privileged
AUDIT SYSTEM AUDIT;
AUDIT ALTER SYSTEM;

-- Object
AUDIT SELECT ON hr.employees;
AUDIT INSERT, UPDATE, DELETE ON hr.employees BY ACCESS;

-- Statements
AUDIT DROP ANY TABLE;
AUDIT ALTER USER;

-- Fine-grained (row-level)
BEGIN
  DBMS_FGA.ADD_POLICY(
    object_schema => 'HR',
    object_name => 'EMPLOYEES',
    policy_name => 'hr_salary_reads',
    audit_column => 'SALARY',
    audit_condition => 'DEPARTMENT_ID = 90',
    statement_types => 'SELECT');
END;
/
```

### View Audit Data

| View                     | Purpose            |
| ------------------------ | ------------------ |
| `DBA_AUDIT_TRAIL`        | All audited events |
| `DBA_AUDIT_SESSION`      | Login/logout       |
| `DBA_AUDIT_STATEMENT`    | Statement audit    |
| `DBA_AUDIT_OBJECT`       | Object access      |
| `DBA_FGA_AUDIT_TRAIL`    | FGA                |
| `DBA_COMMON_AUDIT_TRAIL` | Combined           |

### Cleanup

Regular archive + purge:

```sql
BEGIN
  DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD,
    use_last_arch_timestamp => TRUE);
END;
/
```

## Unified Auditing (12c+)

See [Unified Audit](unified-audit.md) — the recommended path.

## Common Issues

- **SYS.AUD$ growing** — Not purged. Set up `DBMS_AUDIT_MGMT` retention.
- **AUD$ in SYSTEM tablespace** — Move to dedicated:
  ```sql
  BEGIN
    DBMS_AUDIT_MGMT.SET_AUDIT_TRAIL_LOCATION(
      audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD,
      audit_trail_location_value => 'AUDIT_TS');
  END;
  /
  ```
- **Audit not capturing** — Verify `audit_trail` parameter and audit statements are in effect (`DBA_STMT_AUDIT_OPTS`).
- **Performance impact** — Audit adds row inserts; keep policies focused.

## Best Practices

1. **Use Unified Auditing** for new databases.
2. Audit at minimum: session (both success and failure), all DDL, all privilege grants, direct SYS access.
3. Move audit tables out of SYSTEM tablespace.
4. Schedule automatic purge (retain 90 days online, archive to compliance storage).
5. Ship audit trail to a SIEM (Splunk, ELK) — DBA cannot tamper with off-site copy.
6. Alert on suspicious patterns (many failed logins, unusual DDL).
7. Test recovery of audit tablespace — audit trail is precious.
8. Do not audit high-frequency operations (SELECT on every row) without cause — huge volume.

## Interview Questions

1. **Q:** Difference between traditional and unified auditing?
   **A:** Traditional: `SYS.AUD$` + OS files, `AUDIT` statement. Unified (12c+): `AUDSYS.AUD$UNIFIED`, policy-based. Unified is recommended.

2. **Q:** How do you enable Unified Auditing?
   **A:** Relink Oracle binary with `uniaud_on`. Cannot easily disable.

3. **Q:** What is FGA?
   **A:** Fine-Grained Auditing — row-level audit conditions via `DBMS_FGA.ADD_POLICY`.

4. **Q:** How to purge old audit data?
   **A:** `DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL` with archive timestamp.

5. **Q:** Where should AUD$ live?
   **A:** Not SYSTEM — move to a dedicated tablespace via `DBMS_AUDIT_MGMT.SET_AUDIT_TRAIL_LOCATION`.

## References

- Oracle Database Security Guide 19c — Auditing
- MOS Doc ID 731908.1 — Audit Management
- MOS Doc ID 1362997.1 — Unified Audit
