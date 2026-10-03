# Runbook: RAC Node Eviction

## Symptom

- One RAC node kicked out.
- CRS logs say "instance evicted".
- Sessions on that node get FCF-redirected (if configured).
- `crsctl status resource -t` shows resources OFFLINE on evicted node.

## Triage

On any surviving node:

```bash
crsctl status resource -t
crsctl check cluster -all
srvctl status database -db PRD
srvctl status instance -db PRD -instance PRD1

# Recent CRS activity
tail -300 $GRID_HOME/log/<host>/crsd/crsd.log
tail -300 $GRID_HOME/log/<host>/cssd/ocssd.log
tail -300 $GRID_HOME/log/<host>/alertPRDDB.log
```

On the evicted node (once SSH works):

```bash
# Kernel side
dmesg | tail -200
sar -u -f /var/log/sa/sa$(date +%d) | tail -20
sar -q -f /var/log/sa/sa$(date +%d) | tail -20    # runqueue length
sar -B -f /var/log/sa/sa$(date +%d) | tail -20    # paging

# Was there a reboot?
last reboot | head -5
uptime

# Interconnect health
ping <other_node_priv_ip>
```

## Common Causes

### Interconnect failure

Private network died:

```bash
ip link show
ethtool <priv_iface>
```

Look for CRC errors, packet drops, up/down flapping.

### CSSD miscount timeout

`ocssd.log` says "network I/O missed heartbeat":

```
[    CSSD][...]clssnmPollingThread: node <host> at 90% heartbeat fatal, removal in 3.720 secs
```

Node was preserved until vote timed out.

### High CPU / IO starvation

CSSD couldn't heartbeat because of resource starvation:

```
[    CSSD][...]clssnmvSchedDiskThreads: KGSK IO to voting disk exceeds threshold
```

### Voting disk unavailable

Check voting:

```bash
crsctl query css votedisk
```

If < majority visible, eviction is inevitable.

## Actions

### 1. Verify surviving cluster is healthy

```bash
crsctl status resource -t
srvctl status database -db PRD
```

Application traffic should have shifted to surviving nodes.

### 2. Investigate the evicted node

```bash
# CRS logs
grep -iE "reboot|evict|kill|panic" $GRID_HOME/log/<host>/cssd/ocssd.log | tail -30

# Alert log
grep -E "ORA-|shutdown|evict" $ORACLE_BASE/diag/rdbms/prd/PRD1/trace/alert_PRD1.log | tail -30

# OS
grep -iE "kernel|hardware|panic" /var/log/messages | tail -30
```

### 3. Rejoin the node

Once root cause fixed (network, resource, storage):

```bash
# On the evicted node
crsctl start crs

# Monitor
crsctl status resource -t
```

CRS should bring up the local resources, ASM, DB instance.

### 4. Verify RAC health

```sql
-- On any instance
SELECT inst_id, instance_name, host_name, status FROM gv$instance;

-- Reconfig events
SELECT   reconfig_hrs, reconfig#, event, reason
FROM     v$rac_reconfiguration
ORDER BY reconfig# DESC
FETCH FIRST 10 ROWS ONLY;
```

## Post-Mortem

- Root cause: HW, network, resource starvation, bug?
- Detection: how long from problem to eviction?
- Application impact: FCF worked?
- Any data loss? (should be zero — RAC ACID guarantees).
- Need to raise `CSS misscount` or fix real issue?

## Prevention

- Redundant interconnect (bonded / HAIP).
- Voting disks on redundant storage (3+ disks).
- Resource monitoring — no runaway processes on cluster nodes.
- Kernel + firmware current.
- Regular CRS log audit.

## Related

- [Evictions](../18-rac/evictions.md).
- [Clusterware](../18-rac/clusterware.md).
- [Voting Disk](../18-rac/voting-disk.md).
- [RAC Node Failure case study](../34-real-world-case-studies/rac-node-failure.md).
