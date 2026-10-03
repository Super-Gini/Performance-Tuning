# Unified Audit

## Overview

**Unified Auditing** (12c+) consolidates all audit trails — DDL, DML, privileged operations, RMAN, Data Pump, RAS, XS, database vault, FGA — into a single table: `AUDSYS.AUD$UNIFIED`. Policy-driven capture replaces the old `AUDIT ...` statements. Recommended for all 19c+ databases.

## Architecture

```mermaid
flowchart LR
    Session --> Policy[Unified Audit Policies]
    Policy -->|match| Cap[Capture]
    Cap --> Queue[In-memory queue]
    Queue --> Write[Async writer]
    Write --> AudTable[AUDSYS.AUD$UNIFIED]
    AudTable --> View[UNIFIED_AUDIT_TRAIL]
```

## Internal Working

### Enable

Check if Unified Auditing is active:

```sql
SELECT value FROM v$option WHERE parameter = 'Unified Auditing';
```

If `FALSE`, relink Oracle binary:

```bash
$ORACLE_HOME/bin/oracle stop first (shutdown instance)
cd $ORACLE_HOME/rdbms/lib
make -f ins_rdbms.mk uniaud_on ioracle
```

Restart instance. Once ON, cannot disable easily.

### Policies

Ship pre-defined:

- `ORA_LOGON_FAILURES` — captures failed logins.
- `ORA_SECURECONFIG` — captures security-relevant DDL.
- `ORA_ACCOUNT_MGMT` — user/role/profile changes.
- `ORA_DATABASE_PARAMETER` — parameter changes.
- `ORA_CIS_RECOMMENDATIONS` — CIS benchmark set.
- `ORA_STIG_RECOMMENDATIONS` — DoD STIG set.

Enable:

```sql
AUDIT POLICY ORA_LOGON_FAILURES;
AUDIT POLICY ORA_SECURECONFIG;
```

### Custom Policy

```sql
CREATE AUDIT POLICY app_dml_policy
  ACTIONS SELECT, INSERT, UPDATE, DELETE ON hr.employees
  WHEN 'SYS_CONTEXT(''USERENV'', ''SESSION_USER'') NOT IN (''HR_APP'')'
  EVALUATE PER SESSION;

AUDIT POLICY app_dml_policy;
```

Attributes:

- `ACTIONS` — statements to audit
- `WHEN` — conditional (SYS_CONTEXT for user, IP, etc.)
- `EVALUATE PER STATEMENT / SESSION`
- `WHENEVER SUCCESSFUL / NOT SUCCESSFUL`
- `BY <user>` / `EXCEPT <user>`

### Views

| View                             | Purpose               |
| -------------------------------- | --------------------- |
| `UNIFIED_AUDIT_TRAIL`            | All audit records     |
| `AUDIT_UNIFIED_ENABLED_POLICIES` | Enabled policies      |
| `AUDIT_UNIFIED_POLICIES`         | Policy definitions    |
| `AUDIT_UNIFIED_CONTEXTS`         | Context-based filters |

### Storage

`AUDSYS.AUD$UNIFIED` in the `SYSAUX` tablespace by default. Move to dedicated tablespace:

```sql
BEGIN
  DBMS_AUDIT_MGMT.SET_AUDIT_TRAIL_LOCATION(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    audit_trail_location_value => 'AUDIT_TS');
END;
/
```

### Retention and Purge

```sql
-- Enable automatic purge (job)
BEGIN
  DBMS_AUDIT_MGMT.INIT_CLEANUP(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    default_cleanup_interval => 24);   -- hours
END;
/

-- Set retention (last archive timestamp advanced periodically)
BEGIN
  DBMS_AUDIT_MGMT.SET_LAST_ARCHIVE_TIMESTAMP(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    last_archive_time => SYSTIMESTAMP - INTERVAL '90' DAY);
END;
/

-- Create scheduled purge job
BEGIN
  DBMS_AUDIT_MGMT.CREATE_PURGE_JOB(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    audit_trail_purge_interval => 24,
    audit_trail_purge_name => 'PURGE_UNIFIED_AUDIT',
    use_last_arch_timestamp => TRUE);
END;
/
```

### Async Write Mode

Default is **queued write** — records buffered in memory, written by an audit slave process. Fast but risks loss on crash. For strict durability:

```sql
BEGIN
  DBMS_AUDIT_MGMT.SET_AUDIT_TRAIL_PROPERTY(
    audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
    audit_trail_property => DBMS_AUDIT_MGMT.AUDIT_TRAIL_WRITE_MODE,
    audit_trail_property_value => DBMS_AUDIT_MGMT.AUDIT_TRAIL_IMMEDIATE_WRITE);
END;
/
```

## Diagnostic Queries

```sql
-- Is Unified enabled?
SELECT value FROM v$option WHERE parameter = 'Unified Auditing';

-- Enabled policies
SELECT policy_name, enabled_option, entity_name, entity_type
FROM   audit_unified_enabled_policies;

-- Custom policies defined
SELECT policy_name, audit_option, audit_condition
FROM   audit_unified_policies
WHERE  policy_name NOT LIKE 'ORA_%';

-- Recent audit records
SELECT event_timestamp, action_name, dbusername, os_username,
       userhost, client_program_name, object_schema, object_name,
       return_code, sql_text
FROM   unified_audit_trail
WHERE  event_timestamp > SYSTIMESTAMP - INTERVAL '1' HOUR
ORDER  BY event_timestamp DESC
FETCH FIRST 20 ROWS ONLY;

-- Volume per day
SELECT TO_CHAR(event_timestamp, 'YYYY-MM-DD') AS day,
       COUNT(*) AS records
FROM   unified_audit_trail
WHERE  event_timestamp > SYSTIMESTAMP - INTERVAL '30' DAY
GROUP  BY TO_CHAR(event_timestamp, 'YYYY-MM-DD')
ORDER  BY 1 DESC;

-- Audit trail size
SELECT segment_name, ROUND(bytes/1024/1024, 1) AS mb
FROM   dba_segments
WHERE  segment_name LIKE 'AUD$UNIFIED%';
```

## Common Operations

### Enable pre-defined policies

```sql
AUDIT POLICY ora_logon_failures;
AUDIT POLICY ora_secureconfig;
AUDIT POLICY ora_account_mgmt;
```

### Custom policy

```sql
CREATE AUDIT POLICY audit_sys_dba
  ACTIONS ALL ON CURRENT_USER
  ONLY TOPLEVEL;

AUDIT POLICY audit_sys_dba BY SYS;
```

### Disable policy

```sql
NOAUDIT POLICY app_dml_policy;
```

### Drop policy

```sql
DROP AUDIT POLICY app_dml_policy;
```

## Common Issues

- **`AUDSYS.AUD$UNIFIED` in SYSAUX filling** — Move to dedicated tablespace + purge.
- **Records missing after crash (queued write mode)** — Switch to immediate write for critical audits.
- **Policy defined but no records** — Verify `AUDIT POLICY <name>` was run to enable.
- **PDB vs CDB context** — Policies defined at CDB level are common; at PDB level are local.

## Best Practices

1. Enable `ORA_LOGON_FAILURES`, `ORA_SECURECONFIG`, `ORA_ACCOUNT_MGMT`, `ORA_DATABASE_PARAMETER` at minimum.
2. Move `AUD$UNIFIED` out of SYSAUX.
3. Automate purge with `DBMS_AUDIT_MGMT.CREATE_PURGE_JOB` — retain 90 days minimum.
4. Ship to SIEM (Splunk, ELK) — off-site copy.
5. Audit SYS explicitly — SYS operations bypass some other audit rules.
6. Alert on failed logins, ALTER SYSTEM, GRANT operations, DROP USER, etc.
7. Test recovery of audit tablespace.
8. Use `WHEN` conditions to reduce noise (e.g., exclude known app user).
9. Do not audit every SELECT — huge volume.
10. Regularly review `AUDIT_UNIFIED_ENABLED_POLICIES` — nothing "sneak-disabled."

## Interview Questions

1. **Q:** What is Unified Auditing?
   **A:** 12c+ single audit trail (`AUDSYS.AUD$UNIFIED`) with policy-based capture.

2. **Q:** How to enable?
   **A:** Relink Oracle binary with `uniaud_on`.

3. **Q:** Predefined policies?
   **A:** `ORA_LOGON_FAILURES`, `ORA_SECURECONFIG`, `ORA_ACCOUNT_MGMT`, `ORA_DATABASE_PARAMETER`, `ORA_CIS_RECOMMENDATIONS`.

4. **Q:** Where is the audit trail stored?
   **A:** `AUDSYS.AUD$UNIFIED`, default in SYSAUX; recommended to move to dedicated tablespace.

5. **Q:** Async vs immediate write?
   **A:** Async (default): buffered, fast, may lose on crash. Immediate: durable per statement, slower.

6. **Q:** How to purge?
   **A:** `DBMS_AUDIT_MGMT.CREATE_PURGE_JOB` with retention timestamp.

7. **Q:** Can you audit sysdba?
   **A:** Yes — Unified Audit captures SYS actions when policy targets SYS.

## References

- Oracle Database Security Guide 19c — Unified Auditing
- MOS Doc ID 1567006.1 — Unified Auditing Overview
- MOS Doc ID 1362997.1 — Enabling Unified Auditing
