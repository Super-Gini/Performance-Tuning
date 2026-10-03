# Scheduler Jobs

## Overview

A **Scheduler job** is a runnable unit: an action (PL/SQL block, stored procedure, external executable, or a chain) plus a schedule (one-time or repeating) plus optional metadata (job class, priority, credentials, arguments). Jobs are created with `DBMS_SCHEDULER.CREATE_JOB` and live in the data dictionary as first-class objects. They run under the `cjq0` coordinator, which spawns `Jnnn` slaves — see [CJQ0 process](../03-instance-architecture/processes/cjq0.md).

Every job runs in its own database session (SYS or the job owner), writes to `DBA_SCHEDULER_JOB_RUN_DETAILS`, and can be paused, disabled, or dropped without a bounce.

## Job Types

| `job_type`         | Action content                                         |
| ------------------ | ------------------------------------------------------ |
| `PLSQL_BLOCK`      | Anonymous PL/SQL block                                 |
| `STORED_PROCEDURE` | Fully qualified procedure name                         |
| `EXECUTABLE`       | OS executable (needs credential + external job daemon) |
| `CHAIN`            | Start a chain                                          |
| `SQL_SCRIPT`       | SQL\*Plus script (12c+)                                |
| `BACKUP_SCRIPT`    | RMAN script (12c+)                                     |

## Creating a Simple PL/SQL Job

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name        => 'OPS.PURGE_STAGING_JOB',
    job_type        => 'PLSQL_BLOCK',
    job_action      => q'[BEGIN ops.purge_staging(p_days => 7); END;]',
    start_date      => SYSTIMESTAMP,
    repeat_interval => 'FREQ=DAILY; BYHOUR=2; BYMINUTE=15',
    enabled         => TRUE,
    comments        => 'Nightly staging purge - retains 7 days'
  );
END;
/
```

## Creating an External Job

```sql
-- Credential must exist first
BEGIN
  DBMS_CREDENTIAL.CREATE_CREDENTIAL(
    credential_name => 'BATCH_OS_CRED',
    username        => 'oracle',
    password        => 'redacted');
END;
/

BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name          => 'OPS.EXPORT_CSV_JOB',
    job_type          => 'EXECUTABLE',
    job_action        => '/u01/app/scripts/export_csv.sh',
    number_of_arguments => 1,
    credential_name   => 'BATCH_OS_CRED',
    start_date        => SYSTIMESTAMP,
    repeat_interval   => 'FREQ=HOURLY; BYMINUTE=5',
    enabled           => FALSE);

  DBMS_SCHEDULER.SET_JOB_ARGUMENT_VALUE(
    job_name          => 'OPS.EXPORT_CSV_JOB',
    argument_position => 1,
    argument_value    => 'HOURLY');

  DBMS_SCHEDULER.ENABLE('OPS.EXPORT_CSV_JOB');
END;
/
```

## Calendar Syntax (`repeat_interval`)

The Scheduler has its own calendar DSL — separate from cron.

```
FREQ=YEARLY | MONTHLY | WEEKLY | DAILY | HOURLY | MINUTELY | SECONDLY
INTERVAL=n                       -- every n frequency-units
BYMONTH=1..12
BYMONTHDAY=1..31 | -1..-31       -- negative counts from month end
BYWEEKNO=1..53
BYYEARDAY=1..366
BYDAY=MON,TUE,WED,THU,FRI,SAT,SUN (optionally prefixed 1..5 or -1)
BYHOUR=0..23
BYMINUTE=0..59
BYSECOND=0..59
```

Examples:

```sql
'FREQ=WEEKLY; BYDAY=MON,WED,FRI; BYHOUR=6; BYMINUTE=30'      -- Mon/Wed/Fri 06:30
'FREQ=MONTHLY; BYMONTHDAY=-1; BYHOUR=23'                     -- Last day of month 23:00
'FREQ=YEARLY; BYMONTH=12; BYMONTHDAY=31; BYHOUR=23; BYMINUTE=45'
'FREQ=DAILY; BYHOUR=0,4,8,12,16,20'                          -- Every 4h
```

Preview next runs:

```sql
DECLARE
  d TIMESTAMP WITH TIME ZONE := SYSTIMESTAMP;
  n TIMESTAMP WITH TIME ZONE;
BEGIN
  FOR i IN 1..5 LOOP
    DBMS_SCHEDULER.EVALUATE_CALENDAR_STRING(
      calendar_string => 'FREQ=WEEKLY; BYDAY=MON; BYHOUR=6',
      start_date      => SYSTIMESTAMP,
      return_date_after => d,
      next_run_date   => n);
    DBMS_OUTPUT.PUT_LINE(TO_CHAR(n,'YYYY-MM-DD HH24:MI:SS TZR'));
    d := n;
  END LOOP;
END;
/
```

## Job States

| STATE       | Meaning                                      |
| ----------- | -------------------------------------------- |
| `SCHEDULED` | Waiting for next run.                        |
| `RUNNING`   | Currently executing on a `Jnnn` slave.       |
| `COMPLETED` | One-shot job finished successfully.          |
| `FAILED`    | Last run failed. Retries per `max_failures`. |
| `BROKEN`    | Repeated failures crossed `max_failures`.    |
| `DISABLED`  | Manually or automatically disabled.          |
| `SUCCEEDED` | Terminal success (one-shot).                 |
| `STOPPED`   | Killed with `STOP_JOB`.                      |

## Diagnostic Queries

```sql
-- All jobs and their next run time
SELECT owner, job_name, state, enabled, run_count, failure_count,
       last_start_date, next_run_date
FROM   dba_scheduler_jobs
ORDER  BY next_run_date NULLS LAST;

-- Currently running
SELECT job_name, session_id, running_instance, elapsed_time
FROM   dba_scheduler_running_jobs;

-- Last 7 days of runs
SELECT   log_date, owner, job_name, status,
         run_duration, cpu_used, errors
FROM     dba_scheduler_job_run_details
WHERE    log_date > SYSDATE - 7
ORDER BY log_date DESC;

-- Failed / broken jobs
SELECT owner, job_name, failure_count, last_start_date
FROM   dba_scheduler_jobs
WHERE  state IN ('FAILED','BROKEN');

-- Long-running jobs (still executing over 30 min)
SELECT job_name, session_id, elapsed_time
FROM   dba_scheduler_running_jobs
WHERE  elapsed_time > INTERVAL '30' MINUTE;
```

## Managing Jobs

```sql
-- Run once now
EXEC DBMS_SCHEDULER.RUN_JOB('OPS.PURGE_STAGING_JOB');

-- Disable / enable
EXEC DBMS_SCHEDULER.DISABLE('OPS.PURGE_STAGING_JOB');
EXEC DBMS_SCHEDULER.ENABLE ('OPS.PURGE_STAGING_JOB');

-- Change schedule
EXEC DBMS_SCHEDULER.SET_ATTRIBUTE(
       'OPS.PURGE_STAGING_JOB',
       'repeat_interval',
       'FREQ=DAILY; BYHOUR=3');

-- Kill a running job (must have MANAGE_ANY_QUEUE or ownership)
EXEC DBMS_SCHEDULER.STOP_JOB('OPS.PURGE_STAGING_JOB', force => TRUE);

-- Drop
EXEC DBMS_SCHEDULER.DROP_JOB('OPS.PURGE_STAGING_JOB');
```

## Job Arguments and Metadata

Arguments are named or positional and set with `SET_JOB_ARGUMENT_VALUE`. Useful attributes:

| Attribute              | Purpose                                           |
| ---------------------- | ------------------------------------------------- |
| `max_runs`             | Auto-disable after N runs.                        |
| `max_failures`         | Move to `BROKEN` after N failures.                |
| `max_run_duration`     | INTERVAL — kill runaway.                          |
| `stop_on_window_close` | End when Window closes.                           |
| `job_priority`         | 1 (high) – 5 (low), used when > slaves available. |
| `logging_level`        | `OFF`, `RUNS`, `FULL`.                            |
| `restartable`          | Auto-restart if instance crashes.                 |
| `raise_events`         | Bitmask — emit events for state changes.          |

## Common Issues

- **Job stays SCHEDULED past its start** — `JOB_QUEUE_PROCESSES=0` or CJQ0 not started. Check `SHOW PARAMETER JOB_QUEUE_PROCESSES` (need >= 4).
- **`ORA-27369: job failed with unknown return code`** — External job STDERR / non-zero exit; check `DBA_SCHEDULER_JOB_RUN_DETAILS.errors`.
- **`ORA-27476: job does not exist`** — Case-sensitive names; owner may need to be qualified.
- **Job in BROKEN state** — Reset with `EXEC DBMS_SCHEDULER.ENABLE('...')` after fixing root cause and clearing failures via `SET_ATTRIBUTE('failure_count', 0)`.
- **External job hangs** — Missing credential or external job daemon (`extjob`) not registered.

## Best Practices

1. Use a **dedicated schema** for application jobs; never create scheduler objects as `SYS`.
2. Wrap PL/SQL job actions in `BEGIN ... EXCEPTION ...` with `DBMS_OUTPUT` + logging.
3. Set `max_run_duration` on every job so runaway jobs auto-kill.
4. Assign every job to a **[Job Class](job-classes.md)** so it inherits Resource Manager caps.
5. Use `raise_events => DBMS_SCHEDULER.JOB_FAILED` + queue subscription for pager alerts.
6. Use **[Chains](chains.md)** for multi-step ETL rather than a serial PL/SQL block.
7. Keep `logging_level=RUNS`; only escalate to `FULL` while diagnosing.
8. Prefer **calendar syntax** over cron-like external triggers.
9. Never `RUN_JOB` interactively on a production instance during business hours — it runs synchronously in your session.
10. Purge run history quarterly: `DBMS_SCHEDULER.PURGE_LOG(log_history => 90)`.

## Interview Questions

1. **Q:** How is `DBMS_SCHEDULER` different from `DBMS_JOB`?
   **A:** Scheduler adds programs, chains, job classes, resource plans, external jobs, calendar syntax, and complete run history. `DBMS_JOB` is deprecated and internally routes to Scheduler in 12c+.

2. **Q:** How do you run an OS shell script from the database?
   **A:** Create a credential (`DBMS_CREDENTIAL.CREATE_CREDENTIAL`), then a job of `job_type='EXECUTABLE'` with `credential_name` set.

3. **Q:** How do you throttle a Scheduler workload?
   **A:** Assign the job to a Job Class → mapped to a Consumer Group in a Resource Plan → activated by a Window.

4. **Q:** How do you find why a job failed?
   **A:** Query `DBA_SCHEDULER_JOB_RUN_DETAILS` for `status='FAILED'`, read `errors` and `additional_info`, and open the trace file listed in `output`.

5. **Q:** What is the difference between `RUNNING`, `SCHEDULED`, and `BROKEN`?
   **A:** RUNNING = currently executing on a slave; SCHEDULED = waiting for next run time; BROKEN = failed `max_failures` times consecutively and won't retry until re-enabled.

## References

- Oracle Database Administrator's Guide 19c — chapters on Scheduler
- `DBMS_SCHEDULER` PL/SQL Packages and Types Reference
- MOS Doc ID 1359942.1 — Scheduler troubleshooting
- MOS Doc ID 231790.1 — External Job Setup
