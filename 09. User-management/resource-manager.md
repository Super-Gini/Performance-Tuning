# Resource Manager

## Overview

**Database Resource Manager** (DBRM) is Oracle's workload prioritization framework. It divides sessions into **consumer groups**, allocates CPU / parallel-server / undo / active-session-count quotas via a **plan**, and enforces the plan at scheduling time. Essential for multi-tenant CDBs, mixed workloads, and any system where noisy neighbors can starve critical work.

Without Resource Manager, all sessions get equal CPU shares. With it, you can guarantee OLTP gets 70% CPU while batch gets the remainder — even under contention.

## Architecture

```mermaid
flowchart LR
    Session[Session] --> Consumer[Consumer Group<br/>OLTP / REPORTS / BATCH]
    Consumer --> Plan[Resource Plan<br/>PROD_HOURS_PLAN]
    Plan --> Directive[Plan Directives<br/>CPU/PX/Undo shares]
    Directive --> Scheduler[CPU / parallel / undo scheduler]
```

## Internal Working

### Building Blocks

- **Consumer Group** — a bucket of sessions.
- **Plan** — collection of directives.
- **Directive** — per-consumer-group allocation.
- **Mapping** — rules deciding which consumer group a session belongs to.
- **Active Plan** — currently enforced plan.

### Auto-managed sessions

Oracle 19c ships with pre-created groups: `SYS_GROUP`, `LOW_GROUP`, `AUTO_TASK_CONSUMER_GROUP` (maintenance), `BATCH_GROUP`, `INTERACTIVE_GROUP`, etc.

### Creating a Plan

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_PENDING_AREA;
END;
/

-- Consumer groups
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('OLTP','OLTP workload');
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('REPORTS','Reporting');
  DBMS_RESOURCE_MANAGER.CREATE_CONSUMER_GROUP('BATCH','Batch');
END;
/

-- Plan
BEGIN
  DBMS_RESOURCE_MANAGER.CREATE_PLAN('DAYTIME_PLAN','Business hours mix');

  -- Directives
  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'DAYTIME_PLAN',
    group_or_subplan => 'SYS_GROUP',
    comment => 'SYS ops',
    mgmt_p1 => 100);   -- Level 1: SYS always has priority

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'DAYTIME_PLAN',
    group_or_subplan => 'OLTP',
    comment => 'OLTP',
    mgmt_p2 => 70);    -- Level 2: OLTP gets 70% of what's left

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'DAYTIME_PLAN',
    group_or_subplan => 'REPORTS',
    comment => 'Reports',
    mgmt_p2 => 20);

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'DAYTIME_PLAN',
    group_or_subplan => 'BATCH',
    comment => 'Batch',
    mgmt_p2 => 10);

  DBMS_RESOURCE_MANAGER.CREATE_PLAN_DIRECTIVE(
    plan => 'DAYTIME_PLAN',
    group_or_subplan => 'OTHER_GROUPS',
    comment => 'Everything else',
    mgmt_p3 => 100);   -- lowest level

  -- Session mapping rules
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(
    attribute => DBMS_RESOURCE_MANAGER.SERVICE_NAME,
    value => 'oltp.corp',
    consumer_group => 'OLTP');
  DBMS_RESOURCE_MANAGER.SET_CONSUMER_GROUP_MAPPING(
    attribute => DBMS_RESOURCE_MANAGER.SERVICE_NAME,
    value => 'reports.corp',
    consumer_group => 'REPORTS');

  DBMS_RESOURCE_MANAGER.VALIDATE_PENDING_AREA;
  DBMS_RESOURCE_MANAGER.SUBMIT_PENDING_AREA;
END;
/

-- Grant switching privilege
BEGIN
  DBMS_RESOURCE_MANAGER_PRIVS.GRANT_SWITCH_CONSUMER_GROUP(
    grantee_name => 'HR_APP',
    consumer_group => 'OLTP',
    grant_option => FALSE);
END;
/

-- Activate
ALTER SYSTEM SET resource_manager_plan = 'DAYTIME_PLAN' SCOPE=BOTH;
```

### Enforcement

- **CPU shares** — Level 1/2/3 hierarchy. A group's CPU share = its `mgmt_p<N>` / total at level N.
- **Parallel servers** — cap on PX slaves per group.
- **Undo quota** — cap on undo blocks.
- **Active sessions** — cap on concurrent active sessions.
- **Automatic switching** — if a session exceeds `switch_time`, `switch_estimated_time`, or `switch_io_megabytes`, it switches groups (e.g., OLTP → BATCH).

### In CDB

- Set CDB plan: allocates shares across PDBs.
- Each PDB can have its own plan for its consumer groups.
- 19c: memory (SGA) partitioning per PDB via `resource_manager_plan` + PDB shares.

### Maintenance Windows

Oracle's built-in scheduler runs a maintenance plan (`DEFAULT_MAINTENANCE_PLAN`) during maintenance windows — automatic stats gathering, Segment Advisor, SQL Tuning Advisor.

## Components

| Component      | Purpose                  |
| -------------- | ------------------------ |
| Consumer group | Session bucket           |
| Plan           | Collection of directives |
| Plan directive | Per-group allocation     |
| Mapping rule   | Session-to-group logic   |
| Pending area   | Staging for changes      |

## Important Parameters

| Parameter                 | Purpose          |
| ------------------------- | ---------------- |
| `resource_manager_plan`   | Active plan      |
| `parallel_servers_target` | PX resource pool |

## Important Views

| View                                | Purpose               |
| ----------------------------------- | --------------------- |
| `DBA_RSRC_PLANS`                    | Plans                 |
| `DBA_RSRC_CONSUMER_GROUPS`          | Groups                |
| `DBA_RSRC_PLAN_DIRECTIVES`          | Directives            |
| `DBA_RSRC_GROUP_MAPPINGS`           | Mapping rules         |
| `V$RSRC_PLAN`                       | Currently active      |
| `V$RSRC_CONSUMER_GROUP`             | Live group statistics |
| `V$SESSION.RESOURCE_CONSUMER_GROUP` | Session's group       |

## Diagnostic Queries

```sql
-- Active plan
SELECT name, is_top_plan FROM v$rsrc_plan;

-- Consumer group utilization
SELECT name,
       active_sessions, execution_waiters,
       requests, cpu_wait_time, cpu_waits,
       queue_length
FROM   v$rsrc_consumer_group
ORDER  BY cpu_wait_time DESC;

-- Session to group mapping
SELECT s.sid, s.username, s.service_name, s.status,
       s.resource_consumer_group
FROM   v$session s
WHERE  type = 'USER'
ORDER  BY s.resource_consumer_group, s.sid;

-- Historical CPU consumption per group
SELECT snap_id, consumer_group_name,
       cpu_consumed_time_delta AS cpu_ms
FROM   dba_hist_rsrc_consumer_group
WHERE  snap_id > (SELECT MAX(snap_id)-24 FROM dba_hist_snapshot)
ORDER  BY snap_id DESC, cpu_ms DESC;
```

## Common Operations

### Deactivate plan

```sql
ALTER SYSTEM SET resource_manager_plan = '' SCOPE=BOTH;
-- or use the special disable value
ALTER SYSTEM SET resource_manager_plan = 'FORCE:' SCOPE=BOTH;
```

### Switch a session's group manually

```sql
BEGIN
  DBMS_RESOURCE_MANAGER.SWITCH_CONSUMER_GROUP_FOR_SESS(
    session_id => 123,
    session_serial => 456,
    consumer_group => 'BATCH');
END;
/
```

## Common Issues

- **No effect** — Plan not activated or `resource_manager_plan` = ''. Also disabled during maintenance windows.
- **`ORA-29347`** — Cannot activate; validation failed.
- **Sessions in wrong group** — Mapping rules incorrect or session came in via service not mapped.
- **Runaway query not switched** — `switch_time` too high, or `MAX_EST_EXEC_TIME` not set.

## Best Practices

1. **Always enable in CDBs.** Isolate PDB resource consumption.
2. **Map by service name.** Have separate services for OLTP, reports, batch, admin.
3. **Set aggressive kill limits** on batch — `MAX_EST_EXEC_TIME`, `switch_estimated_time`.
4. Use `AUTOMATIC_MAINTENANCE_JOB_PLAN` for maintenance windows.
5. Pin SYS_GROUP at Level 1 — never starve admin.
6. Test plan changes in dev — get shares right.
7. Monitor `V$RSRC_CONSUMER_GROUP.CPU_WAIT_TIME` — high on important groups means over-subscribed.
8. In RAC, consistent plan across instances.
9. Alert on unexpected plan deactivation.
10. Combine with **PDB CPU count** limits for CDB isolation.

## Interview Questions

1. **Q:** What does Resource Manager do?
   **A:** Prioritizes workloads by CPU, parallel servers, undo, and active session counts — enforces a plan on the scheduler.

2. **Q:** Consumer group vs plan?
   **A:** Consumer group = bucket of sessions. Plan = policy of shares across groups.

3. **Q:** How is a session mapped to a group?
   **A:** Mapping rules on service, module, user, program, client_id.

4. **Q:** Does Resource Manager kill sessions?
   **A:** It can, via `switch_time` + `switch_group = KILL_SESSION` or `MAX_EST_EXEC_TIME`.

5. **Q:** In multitenant, how do you split CDB CPU across PDBs?
   **A:** CDB plan with shares per PDB; each PDB can have its own child plan.

6. **Q:** How to activate a plan?
   **A:** `ALTER SYSTEM SET resource_manager_plan = 'MY_PLAN' SCOPE=BOTH;`.

## References

- Oracle Database Administrator's Guide 19c — Resource Manager
- Oracle Database Multitenant Guide 19c — CDB Resource Plans
- MOS Doc ID 1339769.1 — Resource Manager Best Practices
- MOS Doc ID 2005945.1 — DBRM in CDB
