# OEM Troubleshooting

## Overview

Troubleshooting OEM comes down to three questions in order:

1. **Where's the problem** — OMS? Repository? Agent? Network?
2. **What's the observable symptom** — console error? metric gap? job failure?
3. **What do the logs say** — always check logs before guessing.

This page is a cookbook of the most common OEM failure modes and how to walk each of them from symptom to fix.

## Layer 1 — OMS Won't Start / Console 503

### Symptom

Browser to `https://oms1:7802/em` returns 503 or connection reset. `emctl status oms` shows OMS down.

### Investigate

```bash
# Full stack view
emctl status oms -details

# Weblogic side
$MW_HOME/oracle_common/common/bin/wlst.sh
wls:/> connect('weblogic', '<pw>', 't3://oms1:7101')
wls:/> servers()
wls:/> exit()

# Log tail
tail -100 $GC_INST/em/EMGC_OMS1/logs/EMGC_OMS1.out
tail -100 $GC_INST/em/EMGC_OMS1/logs/emoms.trc
tail -100 $GC_INST/user_projects/domains/GCDomain/servers/EMGC_ADMINSERVER/logs/EMGC_ADMINSERVER.log
```

### Common Causes

- **Repository unreachable** — repo DB down / listener down. `sqlplus sysman@repo` from OMS host.
- **AdminServer credential mismatch** — `boot.properties` corruption. Recreate via `resetNMPassword.sh`.
- **Node Manager down** — start via `$MW_HOME/oracle_common/common/bin/startNodeManager.sh` then `emctl start oms`.
- **JVM heap OOM** — bump heap; see [OMS](oem-oms.md).
- **Port already in use** — Check `netstat -tulpn | grep 7802`.
- **TLS cert expired** — `openssl x509 -in <cert> -noout -enddate`. `emctl secure oms`.

## Layer 2 — Agents Show UNREACHABLE

### Symptom

Setup → Manage Cloud Control → Agents: color red. Some or all agents.

### Investigate

```bash
# On the agent host
emctl status agent
emctl ping agent   # pings the OMS
emctl status agent -verbose

# On OMS
telnet <agent_host> 3872
curl -k https://<agent_host>:3872/emd/main/
```

### Common Causes

- **Firewall blocking 3872 in / 4903 out** — network team.
- **Clock skew > 3 min** — Kerberos/TLS handshakes fail. Sync NTP.
- **Agent TLS cert expired** — `emctl secure agent -reg_pwd <pw>`.
- **Reverse DNS mismatch** — Agent hostname in OMS doesn't resolve back. Fix `/etc/hosts` on OMS.
- **Agent binary deleted or filesystem full** — `df -h $AGENT_INST`, restart.
- **After OMS restart** — Give agents 10 min to reconnect.

Fleet-wide TLS renewal:

```bash
# On OMS
emctl secure oms -host oms1.example.com -reg_pwd <regpwd> -sysman_pwd <sysmanpwd>

# On each agent (or via Console)
emctl secure agent -reg_pwd <regpwd>
```

## Layer 3 — Metric Collection Errors

### Symptom

A target (usually a DB) shows Metric Collection Error on one or more metrics.

### Investigate

Console: target → Monitoring → Metric Collection Errors.

Agent side:

```bash
grep "Metric Collection Error" $AGENT_INST/sysman/log/emagent.log | tail
grep "Response DEAD:" $AGENT_INST/sysman/log/emagent.log | tail
```

### Common Causes

- **Monitoring credentials wrong** — DB SYSMAN password changed but not updated in OEM. Setup → Targets → Monitoring Config.
- **Listener down on target** — restart listener.
- **Target renamed at OS level** — remove and rediscover target.
- **`ORA-01017` from agent** — Target DB password expired. Set to `NEVER` for the monitoring user.
- **Custom Metric Extension broken** — Check the SQL manually as `SYSMAN`.

## Layer 4 — Console Slow

### Symptom

Pages load 10s+. Search takes forever.

### Investigate

```bash
# OMS side
grep -c "SLOW" $GC_INST/em/EMGC_OMS1/logs/emoms.log

# Repository
sqlplus / as sysdba
```

```sql
-- Sessions from OMS
SELECT COUNT(*), status FROM v$session WHERE username='SYSMAN' GROUP BY status;

-- Top wait events over last hour
SELECT wait_class, event, total_waits, time_waited/100 secs
FROM   v$system_event
WHERE  wait_class NOT IN ('Idle')
ORDER  BY time_waited DESC
FETCH  FIRST 10 ROWS ONLY;

-- SYSMAN table stats freshness
SELECT table_name, last_analyzed, num_rows
FROM   dba_tables
WHERE  owner='SYSMAN'
ORDER  BY num_rows DESC
FETCH  FIRST 20 ROWS ONLY;
```

### Common Causes

- **Stale stats on SYSMAN** — `EXEC DBMS_STATS.GATHER_SCHEMA_STATS('SYSMAN', degree=>8);`.
- **Purge job broken** — repository ballooning; see [Repository](oem-repository.md).
- **Repository DB out of resources** — CPU, IO, undo.
- **OMS heap paging** — GC pauses. Bump heap.
- **Client-side JVM** — Java WebStart / Flash residuals (old versions).

## Layer 5 — Notifications Not Firing

### Symptom

Incident opens in console but no email.

### Investigate

Console: Setup → Notifications → Mail Servers.

```sql
-- Recent notification attempts
SELECT   attempt_time, notification_state, error_message
FROM     sysman.mgmt_notification_msgs
ORDER BY attempt_time DESC
FETCH FIRST 20 ROWS ONLY;
```

### Common Causes

- **SMTP relay refuses connections** — network team; check `notification_state = 'FAILED'`.
- **Notification rules not matching** — check rule conditions.
- **Recipient email empty on user profile**.
- **PL/SQL notification proc raising exception**.

Test:

```
Setup → Notifications → Send Test Email
```

## Layer 6 — Repository Runaway Growth

### Symptom

SYSMAN tablespace 90%+ and growing.

### Investigate

```sql
SELECT   segment_name, ROUND(bytes/1024/1024/1024,2) gb
FROM     dba_segments
WHERE    owner='SYSMAN'
ORDER BY bytes DESC
FETCH FIRST 20 ROWS ONLY;

SELECT job_name, state, failure_count, last_start_date
FROM   dba_scheduler_jobs
WHERE  owner='SYSMAN'
ORDER  BY last_start_date DESC;
```

### Common Causes

- **`EM_MAINT` job broken** — enable and re-run.
- **Retention too long** — Setup → Manage Cloud Control → Repository → Data Retention.
- **A custom Metric Extension emits high-cardinality metrics** — kills the metric tables.
- **Job execution history not purged** — separate purge.

## Layer 7 — Agent Local Cache Growing

### Symptom

`$AGENT_INST/sysman/emd/upload/` filling up.

### Investigate

```bash
du -sh $AGENT_INST/sysman/emd/upload/
ls -la $AGENT_INST/sysman/emd/upload/ | wc -l
```

### Common Causes

- **OMS unreachable** — see Layer 2.
- **Upload throttled** — Bump `emctl set property -name 'oracle.sysman.core.gcagent.upload.uploadInterval' -value '60'`.
- **XML file corruption** — check `emagent.trc`; may need `emctl clearstate agent` (loses data).

## Layer 8 — Jobs Failing

### Symptom

Job execution history shows FAILED.

### Investigate

Console → Enterprise → Job → Activity → click the job.

```sql
SELECT job_name, status, error_message
FROM   sysman.mgmt_job_execution
WHERE  status = 'FAILED'
ORDER  BY end_time DESC
FETCH  FIRST 20 ROWS ONLY;
```

### Common Causes

- **OS credential expired** — Update Preferred Credentials.
- **Script path wrong on target host** — check exec.
- **Target unavailable at run time** — retry.
- **PL/SQL raised** — inspect `error_message`.

## Fleet Commands from OMS Host

```bash
# Bounce all agents (careful)
emcli execute_agent_action -agent_name="host1:3872" -action=stop
emcli execute_agent_action -agent_name="host1:3872" -action=start

# Force upload from all agents
for a in $(emcli get_agents -search="version:13.5*" | awk 'NR>1{print $1}'); do
  emcli upload_agent -agent_name="$a"
done

# List agents behind on upload
emcli get_agents -search="upload_latency:>1200"
```

## Global Log Locations Cheat-Sheet

| Log                                    | Contents              |
| -------------------------------------- | --------------------- |
| `$GC_INST/em/EMGC_OMS1/logs/emoms.log` | OMS app errors        |
| `$GC_INST/em/EMGC_OMS1/logs/emoms.trc` | OMS traffic trace     |
| `EMGC_OMS1.out`                        | Managed server stdout |
| `EMGC_ADMINSERVER.log`                 | AdminServer           |
| `$AGENT_INST/sysman/log/emagent.log`   | Agent main            |
| `$AGENT_INST/sysman/log/gcagent.log`   | Agent Java            |
| `$AGENT_INST/sysman/log/gcagent.err`   | Agent Java errors     |
| Repository alert log                   | Repo DB events        |

## Best Practices

1. Set OEM to monitor **itself** — `oracle_emd`, repo DB, OMS host — from _another_ OEM if possible.
2. Alert on `Data Upload State` and `Available disk space` on every agent.
3. Renew TLS certs proactively — set OEM to alert 60 days before expiry.
4. Bounce OMS monthly during a change window; JVMs benefit.
5. Keep an offline runbook — you can't reach OEM if OEM is the problem.
6. Baseline repository response times; regressions signal stats issues.
7. Blackouts on any target you're actively troubleshooting.
8. Keep repository patch level ≥ OMS patch level.
9. Version-control credentials for scripted OEM automations.
10. Recovery-test the repository backup quarterly.

## Interview Questions

1. **Q:** What's your first check when a whole fleet of agents goes UNREACHABLE?
   **A:** TLS cert expiry on OMS + clock skew across the fleet.

2. **Q:** SYSMAN tablespace at 95% and growing — what do you look at?
   **A:** `EM_MAINT` job state, metric retention settings, top segments in SYSMAN.

3. **Q:** OMS console is slow — what's the fastest fix that often works?
   **A:** Gather stats on SYSMAN.

4. **Q:** How do you diagnose why notifications aren't going out?
   **A:** `sysman.mgmt_notification_msgs` recent rows, `notification_state`, `error_message`. Check SMTP config.

5. **Q:** Agent shows collection error on a target — first place to look?
   **A:** Monitoring credentials on that target (Setup → Targets → Monitoring Config).

## References

- MOS Doc ID 2144385.1 — Agent Troubleshooting
- MOS Doc ID 1518051.1 — Repository maintenance
- MOS Doc ID 2489053.1 — OMS/Agent version matrix
- MOS Doc ID 1541126.1 — OEM HA
