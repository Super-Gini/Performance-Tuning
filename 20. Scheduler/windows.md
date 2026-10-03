# Scheduler Windows

## Overview

A **Window** is a named time interval that, while open, activates a Resource Manager plan and can raise scheduling events. Jobs whose schedule matches a Window get an implicit priority boost; jobs assigned to consumer groups become subject to the caps of the plan that Window activates.

Windows are how Oracle switches between **daytime** ("OLTP plan": interactive gets 80% of CPU) and **nighttime** ("BATCH plan": ETL group gets 60% of CPU) automatically. In 19c the pre-installed `MAINTENANCE_WINDOW_GROUP` uses this mechanism to run auto-stats, segment advisor, and SQL Tuning Advisor.

## Anatomy

| Attribute         | Purpose                                                 |
| ----------------- | ------------------------------------------------------- |
| `resource_plan`   | Plan activated while window is open.                    |
| `start_date`      | First start time.                                       |
| `repeat_interval` | Calendar syntax (as with jobs).                         |
| `duration`        | INTERVAL — how long the window stays open.              |
| `window_priority` | `HIGH` or `LOW` — resolves overlap with another window. |
| `comments`        | Free text.                                              |

## Pre-installed Windows

```sql
SELECT window_name, resource_plan, repeat_interval, duration, enabled
FROM   dba_scheduler_windows
ORDER  BY window_name;
```

Typical rows:

| Window             | Interval                                     | Duration |
| ------------------ | -------------------------------------------- | -------- |
| `MONDAY_WINDOW`    | `FREQ=WEEKLY;BYDAY=MON;BYHOUR=22;BYMINUTE=0` | 4:00:00  |
| `TUESDAY_WINDOW`   | Tuesday 22:00                                | 4:00:00  |
| `WEDNESDAY_WINDOW` | ...                                          | 4:00:00  |
| `THURSDAY_WINDOW`  | ...                                          | 4:00:00  |
| `FRIDAY_WINDOW`    | ...                                          | 4:00:00  |
| `SATURDAY_WINDOW`  | Saturday 06:00                               | 20:00:00 |
| `SUNDAY_WINDOW`    | Sunday 06:00                                 | 20:00:00 |
| `WEEKNIGHT_WINDOW` | (legacy)                                     | 8:00:00  |
| `WEEKEND_WINDOW`   | (legacy)                                     | 48:00:00 |

All belong to the `MAINTENANCE_WINDOW_GROUP` and default to the `DEFAULT_MAINTENANCE_PLAN`.

## Creating a Custom Window

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_WINDOW(
    window_name     => 'ETL_WINDOW',
    resource_plan   => 'BATCH_PLAN',
    start_date      => TRUNC(SYSDATE)+1 + 1/24,      -- tomorrow 01:00
    repeat_interval => 'FREQ=DAILY; BYHOUR=1',
    duration        => INTERVAL '4' HOUR,
    window_priority => 'HIGH',
    comments        => 'Nightly ETL window');
END;
/
```

Open it manually now:

```sql
EXEC DBMS_SCHEDULER.OPEN_WINDOW('ETL_WINDOW', duration => INTERVAL '30' MINUTE, force => TRUE);
```

Close it early:

```sql
EXEC DBMS_SCHEDULER.CLOSE_WINDOW('ETL_WINDOW');
```

## Window Groups

A **Window Group** is a bundle of windows treated as one. When any member window opens, the group is "open". The pre-installed `MAINTENANCE_WINDOW_GROUP` bundles all seven day-of-week windows so auto-tasks can point at a single group.

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_WINDOW_GROUP('BATCH_WG', 'ETL_WINDOW,BACKUP_WINDOW');
END;
/

-- Add / remove
EXEC DBMS_SCHEDULER.ADD_WINDOW_GROUP_MEMBER   ('BATCH_WG','MAINT_WINDOW');
EXEC DBMS_SCHEDULER.REMOVE_WINDOW_GROUP_MEMBER('BATCH_WG','BACKUP_WINDOW');
```

## Scheduling a Job on a Window

Set the job's `schedule_name` to a window (or window group):

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name       => 'ETL.LOAD_JOB',
    program_name   => 'ETL.LOAD_PRG',
    schedule_name  => 'ETL_WINDOW',
    enabled        => TRUE);
END;
/
```

The job runs each time the window opens.

## Overlap Resolution

If two windows overlap:

- If they have **different priorities**, `HIGH` wins.
- If they have the **same priority**, the currently open one continues; the new one is skipped until next occurrence.

Trace with `DBA_SCHEDULER_WINDOW_LOG`:

```sql
SELECT log_date, window_name, operation
FROM   dba_scheduler_window_log
ORDER  BY log_date DESC
FETCH  FIRST 20 ROWS ONLY;
```

## Diagnostic Queries

```sql
-- Currently active window (if any)
SELECT window_name, actual_start_date
FROM   dba_scheduler_windows
WHERE  active = 'TRUE';

-- Verify resource plan active right now
SELECT name, cpu_managed
FROM   v$rsrc_plan;

-- Upcoming window activations
SELECT window_name, next_start_date, duration, resource_plan
FROM   dba_scheduler_windows
WHERE  enabled = 'TRUE'
ORDER  BY next_start_date;

-- Window log
SELECT log_date, window_name, operation, comments
FROM   dba_scheduler_window_log
WHERE  log_date > SYSDATE - 7
ORDER  BY log_date DESC;
```

## Extending or Shortening an Open Window

```sql
-- Extend the currently-open window by another hour
EXEC DBMS_SCHEDULER.OPEN_WINDOW('ETL_WINDOW', duration => INTERVAL '1' HOUR, force => TRUE);

-- Force close (auto-tasks stop cleanly)
EXEC DBMS_SCHEDULER.CLOSE_WINDOW('ETL_WINDOW');
```

## Common Issues

- **Auto-stats runs into business hours** — Someone extended the maintenance window or the `duration` is too long for the workload. Reduce `duration`.
- **Plan doesn't switch when window opens** — Window is `DISABLED` or `resource_plan` doesn't exist. Check `DBA_SCHEDULER_WINDOWS`.
- **`ORA-27484: cannot open a window that is already open`** — Use `force=>TRUE` on `OPEN_WINDOW`, or close first.
- **Two windows fight for priority** — Set `window_priority` explicitly on both; audit `DBA_SCHEDULER_WINDOW_LOG`.
- **`FORCE => TRUE` runs but plan doesn't stick** — Someone set `RESOURCE_MANAGER_PLAN` via `ALTER SYSTEM` to `FORCE:xxx` — that overrides windows.

## Best Practices

1. Keep the **default maintenance windows** at 4h weeknights, 20h weekends — don't extend without a reason.
2. Create custom windows for **application-specific batch cycles** (ETL, backups, reindexes).
3. Always pair a custom window with a **Resource Plan**; a window without a plan does nothing useful.
4. Use `window_priority=HIGH` on the ETL window and `LOW` on the report window if they overlap on Sunday mornings.
5. Never manually pin `RESOURCE_MANAGER_PLAN='FORCE:XX'` — it disables all window-driven switching.
6. Audit `DBA_SCHEDULER_WINDOW_LOG` after any maintenance run to confirm the window opened and closed as expected.
7. Test window changes with `OPEN_WINDOW(force=>TRUE, duration=>INTERVAL '5' MINUTE)` in a non-prod database first.

## Interview Questions

1. **Q:** What is a Scheduler Window?
   **A:** A named time interval that, while open, activates a Resource Manager plan.

2. **Q:** How is a window group different from a window?
   **A:** A group bundles multiple windows into one schedulable object — like `MAINTENANCE_WINDOW_GROUP` which contains one window per weekday.

3. **Q:** What happens if two windows overlap?
   **A:** Higher-priority window wins. Equal priority — the earlier one keeps running.

4. **Q:** How do you stop auto-stats from running during business hours?
   **A:** Shorten the maintenance window's `duration`, or disable specific auto-tasks with `DBMS_AUTO_TASK_ADMIN.DISABLE`.

5. **Q:** How do you know which resource plan is active right now?
   **A:** `SELECT name FROM v$rsrc_plan;` or `SELECT window_name FROM dba_scheduler_windows WHERE active='TRUE';`.

## References

- Oracle Database Administrator's Guide 19c — Scheduler Windows
- `DBMS_SCHEDULER.CREATE_WINDOW`, `OPEN_WINDOW`, `CLOSE_WINDOW` reference
- MOS Doc ID 786346.1 — Maintenance Windows
