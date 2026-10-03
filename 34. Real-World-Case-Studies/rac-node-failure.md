# Case: 3 AM RAC Node Eviction

## Setup

- 4-node 19c RAC on-prem.
- 03:00 UTC: PagerDuty. Node `db02` evicted.
- Cluster is 3-nodes now. Sessions failed over via FCF; no user-visible impact.
- Task: root-cause the eviction so it doesn't recur.

## Immediate Triage (03:15 UTC)

Cluster state on surviving nodes:

```bash
crsctl status resource -t
```

Node `db02` OFFLINE for all resources. Others ONLINE.

Recent CRS activity on any surviving node:

```bash
tail -300 $GRID_HOME/log/db01/crsd/crsd.log
tail -300 $GRID_HOME/log/db01/cssd/ocssd.log
```

CSSD log shows:

```
2026-08-06 02:58:14 [CSSD]... clssnmPollingThread: node db02, member 2,
    missed heartbeat, sending kill.
2026-08-06 02:58:15 [CSSD]... clssnmEvictMasterNode: node db02 evicted.
2026-08-06 02:58:16 [CSSD]... clssnmDoSyncUpdate: process new dhb; new majority (3 members)
2026-08-06 02:58:18 [CSSD]... clssnmReconfig: cluster reconfigured; group id 12345
```

So node `db02` **missed heartbeats** and was voted out by the majority.

## Investigate on `db02`

SSH works — the host itself is up. Only Oracle CRS was killed.

### System State

```bash
uptime
# 03:15 up 15:23  load average: 0.5, 0.4, 0.4
last reboot | head -3
# No recent reboot
```

Host didn't crash.

### Kernel Messages

```bash
dmesg -T | tail -100
```

Nothing suspicious around 02:58 — no OOM kill, no HW error.

### SAR (System Activity)

```bash
sar -u -f /var/log/sa/sa06 | grep -E "^02:5|^03:0"
```

Result:

```
02:55:00    %CPU  %usr  %nice  %sys  %iowait  %idle
02:55:01    98.5  40.2   0     58.3   0        1.5    <-- KERNEL CPU very high
02:56:01    99.2  35.1   0     64.1   0        0.8    <-- 99% sys CPU
02:57:01    99.5  32.0   0     67.5   0        0.5    <-- worse
02:58:01    99.7  30.0   0     69.7   0        0.3    <-- evicted here
02:59:01    45.0  20.0   0     25.0   30       55.0   <-- recovering
```

`%sys` (kernel) up to 69%. CPU near 100%.

### What Was Running?

```bash
sar -q -f /var/log/sa/sa06 | grep -E "^02:5|^03:0"
```

```
TIME       runq-sz  plist-sz  ldavg-1  ldavg-5
02:55:01   8        1450      12.5     9.2
02:56:01   15       1500      18.7     11.0
02:57:01   22       1550      25.4     13.5
02:58:01   28       1600      32.1     16.0    <-- runqueue 28
```

Runqueue depth 28 — many processes waiting for CPU.

### Culprit Process?

```bash
tail -200 /var/log/messages | grep -E "0[2-3]:5"
```

Nothing OS-noted. But look at process accounting if enabled:

```bash
sa -r -m -f /var/account/pacct | head -30
```

Or look at nginx / haproxy / batch job logs — anything scheduled at 02:55?

Look at `crontab` and Scheduler:

```sql
-- On any node
SELECT job_name, log_date, run_duration, cpu_used
FROM   dba_scheduler_job_run_details
WHERE  log_date BETWEEN TIMESTAMP '2026-08-06 02:50:00' AND TIMESTAMP '2026-08-06 03:00:00'
ORDER  BY log_date;
```

Result:

```
JOB_NAME             LOG_DATE                RUN_DURATION   CPU_USED
GATHER_STATS_JOB     2026-08-06 02:00:00      +00 01:15:00   ...
BATCH_ETL_LOAD       2026-08-06 02:55:12      +00 00:03:22   ...
```

`GATHER_STATS_JOB` — auto-stats — was running for 1h 15min, and `BATCH_ETL_LOAD` kicked off at 02:55, adding to the pile.

### Auto Stats + ETL = CPU Saturation

Both trying to parallelize on the same node. Auto stats had 32 parallel slaves. ETL had 8. Plus regular OLTP. On a 32-core box, load average 32+ means every core is oversubscribed.

### Why This Killed CSSD

CSSD (`ocssd.bin`) has to write a heartbeat to the voting disk every second. Under CPU starvation:

1. CSSD gets scheduled less often.
2. Its writes to voting disk get delayed.
3. Peers see missed heartbeats.
4. After CSS misscount (default 30s), majority votes to evict.

## Root Cause

- **Auto stats window overlapped with ETL start**.
- Neither was CPU-constrained.
- Combined load starved CSSD of CPU.
- CSSD missed 30 s of heartbeats.
- Cluster evicted the node.

## Fix

### Immediate

1. **Move CSSD to real-time priority**:

   ```bash
   # Check current
   ps -eo pid,pri,ni,rtprio,comm | grep ocssd
   ```

   Should already be RT (`SCHED_RR`). If not:

   ```bash
   chrt -f -p 99 <pid>
   ```

   Or set via GI config — usually done at install, but verify.

2. **Set `_high_priority_processes` init parameter** on the DB:

   ```sql
   ALTER SYSTEM SET "_high_priority_processes" = 'LMS*|LGWR|VKTM|CKPT' SCOPE=SPFILE;
   ```

3. **Verify voting disk IO isn't the bottleneck**:

   ```bash
   crsctl query css votedisk
   # For each voting disk:
   dd if=<votedisk> of=/dev/null bs=8192 count=1
   ```

### Long-Term

- **Reschedule** auto stats to a low-CPU window (23:00 UTC).
- **Or reduce parallel degree** on auto stats:

  ```sql
  BEGIN
    DBMS_STATS.SET_GLOBAL_PREFS('DEGREE', '4');
  END;
  /
  ```

- **Kernel isolation**: pin CSSD to specific CPUs so it's never starved:

  ```bash
  # /etc/systemd/system/ohas.service.d/override.conf
  [Service]
  CPUAffinity=0 1   # dedicate cores 0-1
  ```

- **Monitor for CSSD scheduling delays** proactively:

  ```bash
  grep "vote acknowledgement time" $GRID_BASE/diag/crs/db02/crs/trace/ocssd.trc | tail -50
  ```

- **Consider `CSS misscount`** bump (careful — may hide real issues):

  ```bash
  crsctl set css misscount 45   # from default 30
  ```

## Lessons Learned

- **RAC evictions are often OS-level resource starvation, not network.**
- **Auto-stats windows** can be brutal on multi-core boxes.
- **CSSD needs guaranteed CPU** — real-time priority + CPU affinity + `_high_priority_processes`.
- **Correlate incidents with Scheduler run history** — `dba_scheduler_job_run_details`.
- **Alert on runqueue depth > 2x cores** — precursor to eviction.

## Related

- [Evictions](../18-rac/evictions.md).
- [Voting Disk](../18-rac/voting-disk.md).
- [RAC Node Eviction runbook](../27-runbooks/rac-node-eviction.md).
- [Clusterware](../18-rac/clusterware.md).
