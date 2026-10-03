# Resource Consumer Groups (Scheduler View)

## Overview

A **Consumer Group** is a Resource Manager container of sessions/jobs that share CPU, parallel-degree, IO, and active-session caps under an active **Resource Plan**. Every session in the database belongs to exactly one consumer group at a time. The Scheduler uses this mechanism to cap batch and maintenance jobs: **Job Class → Consumer Group → Resource Plan directive → CPU/parallel caps → Window activates**.

This page focuses on the pieces DBAs touch to run scheduled work safely; the full Resource Manager treatment lives in [Resource Manager](../09-user-management/resource-manager.md).

## Where Consumer Groups Come From

- **Pre-installed groups**: `SYS_GROUP`, `OTHER_GROUPS`, `LOW_GROUP`, `BATCH_GROUP` (Exadata), `ETL_GROUP`, `DSS_CRITICAL_GROUP` — inventory varies by version.
- **Custom groups**: created with `DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP`.

```sql
SELECT consumer_group, cpu_mth, mgmt_mth
FROM   dba_rsrc_consumer_groups
ORDER  BY consumer_group;
```

## Creating a Group for Scheduler Use

Resource Manager requires the **pending-area / validate / submit** pattern.

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA;

  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP(
    consumer_group => 'BATCH_GROUP',
    comment        => 'Overnight ETL and maintenance jobs');

  DBMS_RESOURCE_MANAGER.CREATE_PLAN(
    plan    => 'BATCH_PLAN',
    comment => 'Plan active during ETL window');

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan             => 'BATCH_PLAN',
    group_or_subplan => 'BATCH_GROUP',
    mgmt_p1          => 60,
    parallel_degree_limit_p1 => 8,
    active_sess_pool_p1      => 20,
    switch_group     => 'CANCEL_SQL',
    switch_time      => 3600);

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan             => 'BATCH_PLAN',
    group_or_subplan => 'OTHER_GROUPS',
    mgmt_p1          => 40);

  DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA;
  DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA;
END;
/
```

## Wiring Scheduler Job Classes to the Group

```sql
BEGIN
  DBMS_SCHEDULER.CREATE_JOB_CLASS(
    job_class_name          => 'ETL_BATCH_CLASS',
    resource_consumer_group => 'BATCH_GROUP',
    logging_level           => DBMS_SCHEDULER.LOGGING_RUNS);
END;
/
```

Any job assigned to `ETL_BATCH_CLASS` now runs under `BATCH_GROUP` and inherits its 60% CPU minimum, 8 parallel-degree ceiling, and 20 active-session cap while `BATCH_PLAN` is active (activated by a Window).

## Session-Level Group Membership

Sessions land in a group via one of:

1. **Job Class** (Scheduler jobs).
2. **Mapping rules** — by user, service, module, OS user.
3. **Explicit switch** — `DBMS_RESOURCE_MANAGER.SWITCH_CONSUMER_GROUP_FOR_SESS`.

Query current placement:

```sql
SELECT sid, serial#, username, resource_consumer_group
FROM   v$session
WHERE  username IS NOT NULL
ORDER  BY resource_consumer_group;
```

## Automatic Group Switching

Set `switch_group => 'BATCH_GROUP'` and `switch_time => 60` on the interactive-user group to automatically move a session to `BATCH_GROUP` if it runs > 60 s — a soft form of DoS protection for OLTP databases. Common choices for `switch_group`: `LOW_GROUP`, `CANCEL_SQL`, `KILL_SESSION`.

## Diagnostic Queries

```sql
-- All groups and their pending mapping
SELECT consumer_group, comments, category
FROM   dba_rsrc_consumer_groups;

-- Mapping rules
SELECT attribute, value, consumer_group, status
FROM   dba_rsrc_group_mappings;

-- Directives in the active plan
SELECT   group_or_subplan, mgmt_p1, mgmt_p2,
         parallel_degree_limit_p1, active_sess_pool_p1,
         queueing_p1, switch_group, switch_time
FROM     dba_rsrc_plan_directives
WHERE    plan = (SELECT name FROM v$rsrc_plan WHERE is_top_plan='TRUE')
ORDER BY group_or_subplan;

-- Runtime consumption by group
SELECT   name, active_sessions, execution_waiters, requests,
         cpu_wait_time, cpu_waits, consumed_cpu_time,
         yields, active_sess_limit_hits
FROM     v$rsrc_consumer_group
ORDER BY name;
```

## Manual Switch (Ad Hoc)

Force a specific session into a group without changing plans:

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.SWITCH_CONSUMER_GROUP_FOR_SESS(
    session_id     => &sid,
    session_serial => &serial,
    consumer_group => 'LOW_GROUP');
END;
/
```

Useful when a rogue report kicks off during business hours.

## Common Issues

- **Group defined, but caps not enforced** — Plan is not active. Check `v$rsrc_plan`; enable the Window.
- **`ORA-29368: existing consumer group ... referenced in a plan directive`** — Can't drop a group referenced by any plan directive; drop directives first.
- **All jobs still hit `OTHER_GROUPS`** — Job class either doesn't specify `resource_consumer_group` or the class name is misspelled on the job.
- **`ORA-06550`** on `CREATE_PENDING_AREA` — Missing `ADMINISTER_RESOURCE_MANAGER` system privilege.
- **CPU still not throttled** — On many-core machines, `mgmt_p1` is a **minimum share**, not a max. Use `utilization_limit` (11.2+) for hard cap.

## Best Practices

1. Design **at least 3 groups**: `INTERACTIVE_GROUP`, `BATCH_GROUP`, `LOW_GROUP`, plus mandatory `SYS_GROUP` / `OTHER_GROUPS`.
2. Reserve `mgmt_p1=100` for `SYS_GROUP` alone; distribute across others at lower levels.
3. Use `utilization_limit` (hard CPU cap) for cloud databases where noisy-neighbor risk is real.
4. Use `parallel_degree_limit_p1` to prevent one batch job stealing all PX slaves.
5. `active_sess_pool_p1` on `BATCH_GROUP` prevents Scheduler from launching more jobs than the box can handle.
6. Set `switch_group` on report groups so runaway queries downgrade themselves instead of being killed.
7. Only change the plan through a Window; avoid `ALTER SYSTEM SET RESOURCE_MANAGER_PLAN='FORCE:XX'`.

## Interview Questions

1. **Q:** What is a consumer group?
   **A:** A Resource Manager container; a set of sessions that share CPU / parallel / IO caps defined by a plan directive.

2. **Q:** How does a Scheduler job get into a specific consumer group?
   **A:** Via its Job Class — `job_class → resource_consumer_group`.

3. **Q:** What's the difference between `mgmt_p1` and `utilization_limit`?
   **A:** `mgmt_p1` is a percentage share (minimum guarantee); `utilization_limit` is a hard percentage cap on CPU.

4. **Q:** How would you protect OLTP from a runaway report?
   **A:** Put reports in a group with low `mgmt_p1` and `switch_group => 'CANCEL_SQL', switch_time => 300`.

5. **Q:** Why isn't your plan taking effect?
   **A:** No Window has opened it, or someone forced a different plan with `ALTER SYSTEM`.

## References

- Oracle Database Administrator's Guide 19c — Resource Manager
- `DBMS_RESOURCE_MANAGER` PL/SQL reference
- MOS Doc ID 786346.1 — Resource Manager and Scheduler
