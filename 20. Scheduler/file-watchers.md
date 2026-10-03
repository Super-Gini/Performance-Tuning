# Scheduler File Watchers

## Overview

A **File Watcher** is an event source that fires when a file matching a name pattern appears in a directory (local or remote). Jobs subscribed to the watcher run once per detected file with the file's metadata passed as an argument. It's the database-native replacement for cron-based "loop until file exists, then load" wrappers.

Introduced in Oracle 11.2, file watchers require the **external job daemon** on the OS side and a credential inside the database.

## When to Use

- ETL feed lands as a file → trigger load automatically.
- Backup dump lands in the FRA staging area → trigger validation.
- Vendor sends a "done" sentinel file → run the ingestion chain.

Do **not** use file watchers for very-high-frequency arrivals (> 1/min). The poll interval is coarse and you'll flood the alert log; use OS-level inotify + `DBMS_SCHEDULER.RUN_JOB` instead.

## Setup Steps

1. Ensure the Scheduler agent (`schagent`) is installed and registered on the target host.
2. Create a **credential** the DB uses to authenticate to the OS.
3. `CREATE_FILE_WATCHER` — the definition.
4. `CREATE_JOB` with `event_condition` pointing at the watcher.

### 1. Register the External Agent

```bash
# On the target host (as oracle)
schagent -start
schagent -registerdatabase <db-listener-host> <db-listener-port>
```

### 2. Create a Credential

```sql
BEGIN
  DBMS_CREDENTIAL.CREATE_CREDENTIAL(
    credential_name => 'FEED_OS_CRED',
    username        => 'oracle',
    password        => 'redacted');
END;
/
```

### 3. Create the File Watcher

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_FILE_WATCHER(
    file_watcher_name => 'ORDERS_FEED_WATCHER',
    directory_path    => '/u01/feeds/orders/in',
    file_name         => 'orders_*.csv',
    credential_name   => 'FEED_OS_CRED',
    destination       => NULL,        -- local host
    min_file_size     => 1024,        -- ignore truncated writes
    steady_state_duration => INTERVAL '60' SECOND,
    enabled           => TRUE);
END;
/
```

Parameters:

| Parameter               | Purpose                                                                      |
| ----------------------- | ---------------------------------------------------------------------------- |
| `directory_path`        | OS directory to watch.                                                       |
| `file_name`             | Glob pattern (`*`, `?`).                                                     |
| `credential_name`       | Credential for the OS agent.                                                 |
| `destination`           | NULL = local; otherwise a remote destination registered as a Scheduler dest. |
| `min_file_size`         | Ignore files smaller than N bytes (avoid mid-write).                         |
| `steady_state_duration` | File must exist unchanged this long before firing.                           |

`steady_state_duration` is critical — without it you'll fire on partially written files.

### 4. Job Subscribing to the Watcher

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB(
    job_name          => 'ETL.LOAD_ORDERS_FEED_JOB',
    program_name      => 'ETL.LOAD_ORDERS_FEED_PRG',
    event_condition   => 'tab.user_data.file_name LIKE ''orders_%.csv''',
    queue_spec        => 'SYS.SCHEDULER_FILEWATCHER_Q, ORDERS_FEED_WATCHER',
    enabled           => TRUE);
END;
/
```

The event message provides file metadata:

- `file_name` — full path
- `directory_path`
- `actual_file_name`
- `destination`
- `file_size`
- `file_timestamp`

Program receives an argument of type `SYS.SCHEDULER_FILEWATCHER_RESULT`:

```sql
CREATE OR REPLACE PROCEDURE etl.load_orders_feed(
  p_evt IN SYS.SCHEDULER_FILEWATCHER_RESULT)
IS
BEGIN
  INSERT INTO etl.feed_log(feed_name, file_path, ts)
  VALUES ('ORDERS', p_evt.file_name, SYSTIMESTAMP);

  -- ... load logic that reads p_evt.file_name via external table ...
  COMMIT;
END;
/
```

Wire the argument into the program:

```sql
BEGIN
  DBMS_SCHEDULER.DEFINE_METADATA_ARGUMENT(
    program_name      => 'ETL.LOAD_ORDERS_FEED_PRG',
    metadata_attribute=> 'EVENT_MESSAGE',
    argument_position => 1);
END;
/
```

## Diagnostic Queries

```sql
-- All file watchers
SELECT owner, file_watcher_name, directory_path, file_name,
       destination, credential_name, enabled
FROM   dba_scheduler_file_watchers;

-- Jobs subscribing to a watcher
SELECT owner, job_name, event_condition, queue_spec, enabled
FROM   dba_scheduler_jobs
WHERE  queue_spec LIKE '%FILEWATCHER%';

-- File-arrival events recently
SELECT owner, job_name, log_date, status, additional_info
FROM   dba_scheduler_job_run_details
WHERE  job_name = 'LOAD_ORDERS_FEED_JOB'
ORDER  BY log_date DESC
FETCH  FIRST 10 ROWS ONLY;
```

## Testing Without Waiting

Manually publish an event as if a file was seen:

```sql
DECLARE
  msg SYS.SCHEDULER_FILEWATCHER_RESULT;
BEGIN
  msg := SYS.SCHEDULER_FILEWATCHER_RESULT(
           destination     => NULL,
           directory_path  => '/u01/feeds/orders/in',
           actual_file_name=> 'orders_20260806_001.csv',
           file_size       => 4096,
           file_timestamp  => SYSTIMESTAMP,
           ts              => SYSTIMESTAMP);

  DBMS_SCHEDULER.PUBLISH_EVENT('ORDERS_FEED_WATCHER', msg);
END;
/
```

## Common Issues

- **Job never fires** — Agent not registered on host, or credential wrong. Check `$AGENT_HOME/log/*.log`.
- **`ORA-27492`** — File watcher enabled but `_scheduler_file_watcher_interval` is 0. Default is 600 s; leave it alone.
- **Multiple firings per file** — `steady_state_duration` too short; file is still being written. Increase to 60–120 s.
- **Files silently ignored** — `min_file_size` set too high or `file_name` pattern doesn't match.
- **Job fires but file is gone** — Another consumer (or the OS `find -mmin` cleanup) picked it up first. Ensure exclusive processing.

## Best Practices

1. Always set `steady_state_duration` ≥ 60 s to avoid firing on partial writes.
2. Have the loader **move the file to `/done/`** or `/error/` immediately — never let it sit where it could re-fire.
3. Use `min_file_size` = expected minimum feed size to filter empty markers.
4. Watch a **sentinel file** (`orders_20260806.done`) rather than the data file, when the source can produce one.
5. One watcher per feed — no wildcard multi-purpose watchers.
6. Log every firing to a persistent feed_log table for audit.
7. Set the **subscriber job's** `max_run_duration` so a stuck load doesn't leave the queue backed up.
8. Test end-to-end with `PUBLISH_EVENT` before wiring to the real OS.

## Interview Questions

1. **Q:** What is a Scheduler File Watcher?
   **A:** An event source that fires when a file matching a pattern appears steadily in a directory, triggering subscribed jobs.

2. **Q:** Why do you set `steady_state_duration`?
   **A:** So the watcher doesn't fire while the file is still being written.

3. **Q:** How does a job receive the file name?
   **A:** The subscribing program takes a `SYS.SCHEDULER_FILEWATCHER_RESULT` argument via `DEFINE_METADATA_ARGUMENT(EVENT_MESSAGE)`.

4. **Q:** Can a watcher watch a remote host?
   **A:** Yes — install the Scheduler agent on that host, register with the DB, then set `destination` to the registered remote name.

5. **Q:** How do you test without waiting for a real file?
   **A:** `DBMS_SCHEDULER.PUBLISH_EVENT` a fake `SCHEDULER_FILEWATCHER_RESULT`.

## References

- Oracle Database Administrator's Guide 19c — File Watchers
- MOS Doc ID 1953605.1 — File Watcher Setup
- MOS Doc ID 1592751.1 — Remote Scheduler Agent
