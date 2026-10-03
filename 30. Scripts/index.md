# Scripts

Reusable SQL scripts for daily DBA work. All are safe to run in production (no DML/DDL) unless noted.

## Contents

| Page                                              | Purpose                                         |
| ------------------------------------------------- | ----------------------------------------------- |
| [Session Monitoring](session-monitoring.md)       | Who's connected, what they're doing             |
| [Lock Monitoring](lock-monitoring.md)             | Blocking, deadlocks, TX/TM waits                |
| [Tablespace Monitoring](tablespace-monitoring.md) | Fill, growth, autoextend health                 |
| [ASM Monitoring](asm-monitoring.md)               | Diskgroup, rebalance, cell health               |
| [RAC Monitoring](rac-monitoring.md)               | GCS/GES stats, service placement, eviction risk |
| [Data Guard Monitoring](data-guard-monitoring.md) | Apply/transport lag, gaps                       |

## Conventions

- **`&` prompts** — SQL\*Plus substitution. Change to bind (`:x`) if using SQLcl / SQL Developer.
- **`fetch first N rows only`** used to bound output — Oracle 12c+.
- **Container awareness** — most work in both non-CDB and CDB. For CDB, prefix `CDB_` (e.g., `CDB_TABLESPACES`) if you want cross-PDB view; use `V$` for session-scoped.
- **Formatting** — assumes `SET LINESIZE 200 PAGESIZE 100`.

## Automation

- Wrap into shell → cron for cadence.
- Feed output to monitoring pipeline (Prometheus text_exporter, CloudWatch metric filter, Splunk, etc.).
- For alerting, wrap in DBMS_SCHEDULER + `raise_events => JOB_FAILED`.

## Related

- [Monitoring](../24-monitoring/index.md).
- [Checklists](../28-checklists/index.md).
- [Runbooks](../27-runbooks/index.md).
