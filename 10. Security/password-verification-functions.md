# Password Verification Functions

## Overview

A **Password Verify Function** is a PL/SQL function called by Oracle when a user changes their password. It examines the new password and either allows the change (returns TRUE) or rejects it with a specific error. Attached to a **profile** via `PASSWORD_VERIFY_FUNCTION`.

Every production database should have a non-default verify function enforcing organizational password policy.

## Oracle-Shipped Functions

Available in `$ORACLE_HOME/rdbms/admin/utlpwdmg.sql`:

| Function                        | Notes              |
| ------------------------------- | ------------------ |
| `verify_function`               | 11g default        |
| `verify_function_11G`           | Legacy             |
| `ora12c_verify_function`        | 12c standard       |
| `ora12c_strong_verify_function` | Strict CIS-aligned |
| `ora12c_stig_verify_function`   | DoD STIG           |

Run `utlpwdmg.sql` in each PDB to install.

## Signature

```sql
FUNCTION my_verify(
  username     VARCHAR2,
  password     VARCHAR2,
  old_password VARCHAR2
) RETURN BOOLEAN
```

Returns TRUE to allow, raises exception to reject.

## Example — Strict Policy

```sql
CREATE OR REPLACE FUNCTION corp_pwd_verify(
  username     VARCHAR2,
  password     VARCHAR2,
  old_password VARCHAR2)
RETURN BOOLEAN
AS
  differ NUMBER;
BEGIN
  -- Length check
  IF LENGTH(password) < 14 THEN
    RAISE_APPLICATION_ERROR(-20001, 'Password must be at least 14 characters');
  END IF;

  -- Complexity: must contain each of upper, lower, digit, special
  IF NOT REGEXP_LIKE(password, '[[:upper:]]') THEN
    RAISE_APPLICATION_ERROR(-20002, 'Must contain an uppercase letter');
  END IF;
  IF NOT REGEXP_LIKE(password, '[[:lower:]]') THEN
    RAISE_APPLICATION_ERROR(-20003, 'Must contain a lowercase letter');
  END IF;
  IF NOT REGEXP_LIKE(password, '[[:digit:]]') THEN
    RAISE_APPLICATION_ERROR(-20004, 'Must contain a digit');
  END IF;
  IF NOT REGEXP_LIKE(password, '[!@#$%^&*()_+\-=\[\]{};:''",.<>/?`~]') THEN
    RAISE_APPLICATION_ERROR(-20005, 'Must contain a special character');
  END IF;

  -- Different from username
  IF LOWER(password) = LOWER(username) THEN
    RAISE_APPLICATION_ERROR(-20006, 'Password cannot match username');
  END IF;

  -- Different from old password
  IF old_password IS NOT NULL THEN
    SELECT COUNT(*) INTO differ FROM dual
    WHERE LOWER(password) = LOWER(old_password);
    IF differ > 0 THEN
      RAISE_APPLICATION_ERROR(-20007, 'New password must differ from old');
    END IF;
  END IF;

  -- Blocklist
  IF LOWER(password) IN ('welcome', 'oracle', 'password', 'company', 'summer2026') THEN
    RAISE_APPLICATION_ERROR(-20008, 'Password is on blocklist');
  END IF;

  RETURN TRUE;
END corp_pwd_verify;
/
```

## Apply to Profile

```sql
ALTER PROFILE app_secure LIMIT
  PASSWORD_VERIFY_FUNCTION corp_pwd_verify;
```

## Test

```sql
-- As a user assigned that profile
ALTER USER hr_app IDENTIFIED BY "weak";
-- ORA-20001: Password must be at least 14 characters

ALTER USER hr_app IDENTIFIED BY "StrongPwd12345!";
-- Succeeds
```

## Multitenant Considerations

- Create verify function **inside each PDB** where it will be used.
- Grant `EXECUTE` on it to `PUBLIC` (or specific users). If a common profile references a function, all PDBs must have the function.

## Common Issues

- **`ORA-28221: REPLACE not specified`** — Password change requires old password when function checks it and user isn't SYSDBA.
- **Function invalid** — `ALTER USER ... IDENTIFIED BY` fails until function recompiles. `DBA_OBJECTS.STATUS = INVALID`.
- **Function too strict** — Locks legitimate users out. Have a documented waiver process.

## Best Practices

1. Deploy a **corporate standard** verify function everywhere.
2. Enforce **≥ 14 characters** (NIST 800-63B recommendation).
3. Include a **blocklist** of common weak passwords + org-specific patterns (company name, season+year).
4. Reject username-embedded passwords.
5. Compare against **old password** — no reuse.
6. Test the function in dev; keep a rollback plan.
7. Version-control the function; audit changes.
8. Apply to every non-default profile.
9. Do not attempt full "check against N history" in the function; use `PASSWORD_REUSE_MAX` in profile instead.
10. Consider integrating with organizational identity policy (LDAP + Kerberos push).

## Interview Questions

1. **Q:** What is a password verify function?
   **A:** PL/SQL function that validates password strength when a user changes password. Returns TRUE or raises exception.

2. **Q:** Function signature?
   **A:** `FUNCTION f(username VARCHAR2, password VARCHAR2, old_password VARCHAR2) RETURN BOOLEAN`.

3. **Q:** How to attach to a profile?
   **A:** `ALTER PROFILE p LIMIT PASSWORD_VERIFY_FUNCTION f;`.

4. **Q:** Oracle-shipped functions?
   **A:** `verify_function`, `ora12c_verify_function`, `ora12c_strong_verify_function`, `ora12c_stig_verify_function`.

5. **Q:** In multitenant, where does the function live?
   **A:** Must exist in the container where the profile is defined; typically in each PDB or CDB$ROOT.

## References

- Oracle Database Security Guide 19c — Password Policy
- MOS Doc ID 401207.1 — Password Verify Function
- NIST SP 800-63B — Digital Identity Guidelines
- CIS Benchmark — Oracle Database 19c
