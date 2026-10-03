# Scheduler Programs

## Overview

A **program** is a reusable action definition, independent of any specific run time. It packages a `PLSQL_BLOCK`, `STORED_PROCEDURE`, `EXECUTABLE`, or `EXTERNAL_SCRIPT` along with typed argument metadata. Multiple jobs and chain steps can share the same program — you patch the code in one place and every consumer inherits the change on its next run.

Think of a program as the **"what"** and a job as the **"when + as whom"**. In small setups you can skip programs; in production ETL/maintenance suites they eliminate duplication and provide a natural typing layer for arguments.

## When to Use a Program vs Inline `job_action`

| Pattern                                                 | Use                 |
| ------------------------------------------------------- | ------------------- |
| One-off PL/SQL block called by a single job             | Inline `job_action` |
| Same action wanted on 3 schedules or from 4 chain steps | Program             |
| Action takes typed arguments that must be validated     | Program             |
| External shell script reused across many jobs           | Program             |

## Creating a Program

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_PROGRAM(
    program_name        => 'OPS.GATHER_STATS_PRG',
    program_type        => 'STORED_PROCEDURE',
    program_action      => 'OPS.GATHER_SCHEMA_STATS',
    number_of_arguments => 2,
    enabled             => FALSE,
    comments            => 'Wrapper for schema-level stats');

  DBMS_SCHEDULER.DEFINE_PROGRAM_ARGUMENT(
    program_name      => 'OPS.GATHER_STATS_PRG',
    argument_position => 1,
    argument_name     => 'P_SCHEMA',
    argument_type     => 'VARCHAR2',
    default_value     => 'APP');

  DBMS_SCHEDULER.DEFINE_PROGRAM_ARGUMENT(
    program_name      => 'OPS.GATHER_STATS_PRG',
    argument_position => 2,
    argument_name     => 'P_DEGREE',
    argument_type     => 'NUMBER',
    default_value     => '4');

  DBMS_SCHEDULER.ENABLE('OPS.GATHER_STATS_PRG');
END;
/
```

## Using a Program from a Job

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'OPS.APP_STATS_JOB',
    program_name    => 'OPS.GATHER_STATS_PRG',
    repeat_interval => 'FREQ=DAILY; BYHOUR=1',
    enabled         => FALSE);

  DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE(
    job_name          => 'OPS.APP_STATS_JOB',
    argument_name     => 'P_SCHEMA',
    argument_value    => 'APPCORE');

  DBMS_SCHEDULER.ENABLE('OPS.APP_STATS_JOB');
END;
/
```

## Program Types

| `program_type`     | `program_action`                           |
| ------------------ | ------------------------------------------ |
| `PLSQL_BLOCK`      | Full anonymous block                       |
| `STORED_PROCEDURE` | Fully qualified procedure/function name    |
| `EXECUTABLE`       | Absolute path to OS binary                 |
| `EXTERNAL_SCRIPT`  | Script content stored inside the DB (12c+) |
| `SQL_SCRIPT`       | SQL\*Plus script content (12c+)            |
| `BACKUP_SCRIPT`    | RMAN script content (12c+)                 |

`EXTERNAL_SCRIPT` and `SQL_SCRIPT` are unique — the script body itself is stored in `program_action` (not a path). This eliminates OS filesystem dependencies for scheduled scripts.

## External Script Example

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_PROGRAM(
    program_name   => 'OPS.DF_CHECK_PRG',
    program_type   => 'EXTERNAL_SCRIPT',
    program_action => q'{#!/bin/bash
                         df -h /u01 | awk 'NR==2 {print $5}' > /tmp/u01_usage.txt}',
    enabled        => TRUE);
END;
/
```

Jobs referring to this program need a credential so the DB knows which OS user to run the script as.

## Diagnostic Queries

```sql
-- All programs
SELECT owner, program_name, program_type, enabled, number_of_arguments
FROM   dba_scheduler_programs;

-- Their argument definitions
SELECT program_name, argument_position, argument_name,
       argument_type, default_value
FROM   dba_scheduler_program_args
ORDER  BY program_name, argument_position;

-- Which jobs use which program
SELECT owner, job_name, program_name, state, next_run_date
FROM   dba_scheduler_jobs
WHERE  program_name IS NOT NULL
ORDER  BY program_name;
```

## Changing Program Arguments

```sql
-- Add a new argument
BEGIN
  DBMS_SCHEDULER.DEFINE_PROGRAM_ARGUMENT(
    program_name      => 'OPS.GATHER_STATS_PRG',
    argument_position => 3,
    argument_name     => 'P_ESTIMATE_PERCENT',
    argument_type     => 'NUMBER',
    default_value     => 'DBMS_STATS.AUTO_SAMPLE_SIZE');
END;
/

-- Drop an argument (jobs bound to that position must be updated first)
EXEC DBMS_SCHEDULER.DROP_PROGRAM_ARGUMENT('OPS.GATHER_STATS_PRG', 3);
```

## Common Issues

- **`ORA-27476: object does not exist`** on job create — Program is disabled. Enable it, or use `enabled=>FALSE` on the job and enable after argument binding.
- **`ORA-27453: undefined argument value`** at runtime — Job did not `SET_JOB_ARGUMENT_VALUE` for an argument without a default.
- **Program body changes not picked up** — Programs are compiled on creation; if the referenced procedure changes signature, disable + enable the program.
- **External program hangs** — Check `credential_name` on the job and OS user's shell PATH.

## Best Practices

1. Use programs for **any action referenced by more than one job**.
2. Give every program a **`comments`** string explaining what it does.
3. Prefer **`STORED_PROCEDURE`** over `PLSQL_BLOCK` — keeps logic under source control in the schema.
4. Use `EXTERNAL_SCRIPT` (12c+) so scripts don't drift with OS filesystem changes.
5. Give arguments **default values** where possible to make jobs simpler.
6. Version programs by name suffix (`GATHER_STATS_PRG_V2`) rather than mutating in place, if the application depends on stability.
7. Own programs from a dedicated **schema per team**, not `SYS`.
8. Grant `EXECUTE` on the program only to the job-owning schemas — not `PUBLIC`.

## Interview Questions

1. **Q:** Why use a program?
   **A:** Reuse across jobs/chains, typed arguments, defaults, and centralized action definition.

2. **Q:** Difference between `EXECUTABLE` and `EXTERNAL_SCRIPT`?
   **A:** `EXECUTABLE` runs a file on the OS filesystem; `EXTERNAL_SCRIPT` stores the script body inside the database and is streamed out at runtime.

3. **Q:** Can a program run without a job?
   **A:** No. Programs are definitions; you need a job (or chain step) to actually execute them.

4. **Q:** How do you change a program that's currently in use?
   **A:** Disable it, redefine arguments/action, re-enable. Existing jobs pick up the change on next run.

## References

- Oracle Database Administrator's Guide 19c — Scheduler Programs
- `DBMS_SCHEDULER.CREATE_PROGRAM`, `DEFINE_PROGRAM_ARGUMENT` reference
