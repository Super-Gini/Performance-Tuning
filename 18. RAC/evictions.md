# Node Evictions

## Overview

A **node eviction** is when CSSD forcibly removes a node from the cluster — usually by rebooting it — because the cluster believes the node is unhealthy. From the surviving cluster's perspective, eviction is a safety measure to prevent split-brain data corruption. From the evicted node's perspective, it's an unplanned reboot.

Every eviction has a root cause. Common ones: interconnect issues, storage issues, CPU starvation, hangcheck, misconfigured firewall.

## Types

- **Voluntary eviction** — Node's own CSSD decides to leave (usually because it lost quorum).
- **Involuntary eviction** — Other CSSDs vote a node out; that node's CSSD receives the notice and reboots.
- **Reboot via IPMI** — CSSD power-cycles a node via IPMI (12c+ preferred).
- **`oclskd` reboot** — Legacy method (11g).

## Common Causes

### 1. Interconnect Failure

- Physical cable cut, switch reboot, NIC firmware bug.
- Symptoms: CSS alert log shows "network heartbeat has failed."
- Prevention: bonded NICs, redundant switches, monitoring.

### 2. Voting Disk Unreachable

- Storage array issue, LUN unmapped, ASM diskgroup dismounted.
- Symptoms: "CSSD unable to access voting disk."
- Prevention: multiple voting disks in HIGH-redundancy ASM.

### 3. CPU Starvation

- Runaway process, misbehaving app.
- CSSD is real-time priority; other processes shouldn't starve it, but a kernel-level lockup can.
- Prevention: monitor `top`, `mpstat`; use `oclumon` for cluster health.

### 4. Memory Pressure

- OOM killer terminates CSS or Oracle.
- Prevention: sized swap, HugePages, no memory oversubscription.

### 5. Kernel Bugs

- Rare but documented.
- Prevention: keep OS kernel and firmware current.

### 6. Time Skew

- Large clock jump (NTP correction) disturbs heartbeats.
- Prevention: chrony or NTP with `slew` mode, not `step`.

## Investigating an Eviction

Immediately after the node reboots:

1. **CRS alert log**: `$ORACLE_BASE/diag/crs/<host>/crs/alert.log`.
2. **CSSD trace**: `$ORACLE_BASE/diag/crs/<host>/crs/trace/ocssd.trc` — has the reason.
3. **System log**: `/var/log/messages` (RHEL) — dmesg, kernel messages.
4. **`oclumon`**: cluster health snapshot from all nodes.
5. **`chactl`**: Cluster Health Advisor output.

### CSSD Trace — Key Messages

- `network heartbeat has failed` → interconnect.
- `disk heartbeat has failed` → voting disk / storage.
- `CSSD reboot` → self-eviction due to lost quorum.
- `Waiting for local network heartbeat` → NIC issue.
- `Reconfiguration is in progress` → cluster is adjusting membership.

## Diagnostic Queries

```bash
# Cluster health during recent period
oclumon showtrail -n <node> -last "01:00:00"
oclumon dumpnodeview -n <node> -last "01:00:00"

# Cluster Health Advisor
chactl query diagnosis -db <db> -start "2026-08-06 14:00:00" -end "2026-08-06 15:00:00"

# CSS misscount and reboot time
crsctl get css misscount
crsctl get css reboottime

# Recent evictions from CRS alert log
grep -i "evict\|reboot" $ORACLE_BASE/diag/crs/$(hostname -s)/crs/alert.log | tail

# Interconnect metrics
oifcfg iflist -p -n
```

## Prevention

1. **Redundant interconnect** — bonded NICs, LACP, redundant switches.
2. **Redundant voting disks** — HIGH-redundancy ASM.
3. **Time sync** — chrony slew mode.
4. **HugePages** — reduces memory pressure.
5. **Monitoring** — proactive alerts for interconnect latency, disk health, CPU load.
6. **Firmware / OS current** — apply RUs and OS updates.
7. **CLU health checks** — `oclumon`, `chactl` in monitoring dashboards.
8. **`priocntl`** on non-Oracle processes to prevent CPU starvation of CSSD.

## Diagnostic Runbook

See [RAC Node Eviction Runbook](../27-runbooks/rac-node-eviction.md) for step-by-step.

Summary:

1. Confirm eviction from CRS alert log.
2. Find the reason in CSSD trace.
3. Correlate with OS logs, interconnect metrics.
4. Identify root cause: network / disk / OS / config.
5. Remediate — patch, tune, reconfigure.
6. Track evictions per month — repeat evictions signal systemic problem.

## Common Issues

- **Repeated evictions on same node** — hardware issue (NIC, cable, disk).
- **Evictions cluster-wide simultaneously** — network storm; switch or interconnect.
- **After patching** — kernel or firmware change; check OS logs.
- **After VM migration** — clock jump post-migration; NTP.

## Best Practices

1. Alert on **any** eviction.
2. Trend analysis — evictions per month per node.
3. Bond interconnect NICs.
4. Keep OS + firmware current.
5. Test failure modes in a lower environment.
6. Do not run non-essential workloads on RAC nodes.
7. Monitor interconnect latency in real time.
8. Cluster Health Advisor (12.2+) — enable and monitor.
9. Isolate root cause before considering `misscount` adjustments.
10. Document each eviction in a post-mortem log.

## Interview Questions

1. **Q:** What is a node eviction?
   **A:** Forced removal of a node from the cluster (usually reboot) by CSSD to prevent split-brain corruption.

2. **Q:** Common causes?
   **A:** Interconnect failure, voting disk issue, CPU starvation, memory pressure, time skew, kernel bugs.

3. **Q:** Where to find the reason?
   **A:** CSSD trace and CRS alert log.

4. **Q:** `misscount`?
   **A:** Missed network heartbeat seconds before eviction. Default 30.

5. **Q:** Prevention?
   **A:** Redundant interconnect + voting disks, time sync, HugePages, current OS/firmware.

6. **Q:** Cluster Health Advisor?
   **A:** 12.2+ ML-based predictor of cluster health issues; queried via `chactl`.

## References

- Oracle Grid Infrastructure Administration 19c
- MOS Doc ID 1050908.1 — Clusterware Troubleshooting
- MOS Doc ID 265769.1 — GI Overview
- MOS Doc ID 265769.1 — Cluster Evictions
- Runbook: [RAC Node Eviction](../27-runbooks/rac-node-eviction.md)
