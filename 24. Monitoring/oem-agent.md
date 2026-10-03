# OEM Agent

## Overview

The **OEM Agent** is the small daemon that runs on every managed host and pushes metric/event data up to the OMS. In OEM 13c it's a self-contained install (~700 MB) — one per host, discovers all targets on that host, uploads on a schedule, and executes jobs on the OMS's behalf. Zero DB downtime to install, upgrade, or bounce.

## What an Agent Does

- Discovers targets: databases, listeners, ASM, cluster, host, WLS, etc.
- Collects metrics on the schedule for each target type.
- Uploads to OMS every few minutes (batch).
- Executes remote jobs (RMAN backups, patching, custom shell/SQL).
- Receives commands (blackouts, threshold changes) via HTTPS callbacks.
- Runs Compliance / Configuration collections.
- Local caching if OMS is unreachable — replays when back.

## Files & Layout

```
$AGENT_BASE/                    # e.g. /u01/oracle/agent
  agent_13.5.0.0.0/            # binaries
    bin/emctl                   # agent control
    plugins/                    # per-target plugins
  agent_inst/                   # this agent's state
    bin/emctl                   # instance emctl
    sysman/emd/                 # metric cache
    sysman/log/                 # logs
    sysman/config/              # emd.properties, gcaparams.properties
```

Set environment:

```bash
export AGENT_HOME=/u01/oracle/agent/agent_13.5.0.0.0
export AGENT_INST=/u01/oracle/agent/agent_inst
export PATH=$AGENT_INST/bin:$PATH
```

## `emctl` — Agent Control

Status:

```bash
emctl status agent
```

Typical output:

```
Oracle Enterprise Manager Cloud Control 13c Release 5
Copyright ...
Agent Version                : 13.5.0.19.0
OMS Version                  : 13.5.0.0.0
Agent Home                   : /u01/oracle/agent/agent_13.5.0.0.0
Agent Log Directory          : /u01/oracle/agent/agent_inst/sysman/log
Agent Binaries               : /u01/oracle/agent/agent_13.5.0.0.0/agent_inst
Core JAR Location            : /u01/oracle/agent/agent_13.5.0.0.0/jlib
Agent Process ID             : 12345
Parent Process ID            : 12300
Agent URL                    : https://host1:3872/emd/main/
Repository URL               : https://oms1:4903/empbs/upload
Started at                   : 2026-08-06 03:14:00
Last Reload                  : (none)
Last successful upload       : 2026-08-06 08:35:00
Last attempted upload        : 2026-08-06 08:35:00
Total Megabytes of Data Uploaded  : 8.15
Number of XML files pending upload : 0
Size of XML files pending upload   : 0.00
Available disk space on upload filesystem : 76.02%
Data Upload State            : SUCCEEDED
Agent is Running and Ready
```

Fields to alert on: `Last successful upload`, `Available disk space`, `XML files pending upload`, `Data Upload State`.

Start/stop:

```bash
emctl start agent
emctl stop agent
emctl reload agent
emctl upload agent      # force upload
emctl clearstate agent  # clear queued metrics
```

## Deployment

New host = fresh agent. Two methods:

### 1. From OMS Console

Setup → Add Target → Add Targets Manually → Install Agent on Host. Fills in:

- Hostname / IP
- OS platform (Linux 64, Solaris, Windows)
- Agent base directory
- Credentials (root or sudo)

OMS transfers a bootstrap tarball via SSH, expands into `$AGENT_BASE`, runs `root.sh`, registers with OMS.

### 2. Silent (`agentDeploy.sh`)

For automation and firewalled hosts:

```bash
# On any host with network to OMS
curl -k -o agentimage.zip \
     https://oms1.example.com:4903/em/install/getAgentImage?platform=linux-x64

# Unzip on target host
unzip agentimage.zip

# Deploy
./agentDeploy.sh \
    AGENT_BASE_DIR=/u01/oracle/agent \
    ORACLE_HOSTNAME=host1.example.com \
    OMS_HOST=oms1.example.com \
    EM_UPLOAD_PORT=4903 \
    AGENT_REGISTRATION_PASSWORD=<regpwd>
```

## Discovery

Once the agent starts, run auto-discovery from OMS:

Setup → Add Target → Configure Auto Discovery → run on selected host. OEM finds:

- Databases (via `oratab`, `srvctl`, `ps`).
- Listeners (`lsnrctl status` output).
- ASM instance.
- Cluster / GI home.
- Weblogic domains.
- HTTP Server instances.

Promote discovered targets to managed targets from the console.

## Blackouts

Prevent alerts during maintenance:

```bash
# Immediate 2-hour blackout on all targets on this host
emctl start blackout MY_PATCH_2H_BO -nodelevel

# Named blackout for a specific DB
emctl start blackout DB_PATCH_BO \
    dbstart_dbtarget_1=oracle_database

# Stop
emctl stop blackout MY_PATCH_2H_BO
emctl status blackout
```

Blackouts also from OMS console: Targets → Blackouts → Create — scheduled windows.

## Upgrading the Agent

From the OMS Console → Setup → Manage Cloud Control → Upgrade Agents. Select agents, cycle through wizard. Under the hood, OMS ships the new agent binary to each host and swaps.

Silent:

```bash
$OMS_HOME/bin/emctl_gcagent_deploy -upgrade -hostList host1,host2 -sysmanpwd <pw>
```

## Adding/Removing Targets Manually

```bash
# Discovery for one target type
emctl config agent addtarget /u01/oracle/agent/targets_dbdisc.xml

# List
emctl config agent listtargets
```

## Logs

```
$AGENT_INST/sysman/log/
├── emagent.log     <- start here
├── emagent.trc     <- deeper trace
├── gcagent.log
├── gcagent.err     <- Java errors
├── gcagent_errors.log
├── emctl.log       <- agent commands' output
└── ...
```

Search for the last upload attempt:

```bash
grep "Beginning Upload" $AGENT_INST/sysman/log/gcagent.log | tail
```

Search for connectivity issues:

```bash
grep -E "IOException|timeout|SSLHandshakeException" $AGENT_INST/sysman/log/gcagent.err | tail
```

## Health Metrics About Agents

```sql
-- From the repository DB (SYSMAN)
SELECT target_name, target_type, avail_status, avg_upload_latency
FROM   sysman.mgmt_targets t
JOIN   sysman.mgmt_current_availability a ON t.target_guid = a.target_guid
WHERE  target_type = 'oracle_emd';
```

Or console: Setup → Manage Cloud Control → Agents.

## Cache Management

If the OMS was down for a long time, the agent queues metrics locally in `$AGENT_INST/sysman/emd/upload/`. When OMS returns, the agent uploads catch-up. If the queue grows too large:

```bash
emctl upload agent           # nudge
emctl clearstate agent       # last resort — discards queue
```

Use `clearstate` only in emergencies — you lose metric history.

## Common Issues

- **`Agent UNREACHABLE`** — OMS can't reach agent's HTTPS port (default 3872). Firewall / SELinux.
- **`Upload Manager (mm-svcs): Failed to upload`** — TLS cert or clock skew. Sync NTP; re-secure agent.
- **`Available disk space on upload filesystem`** low — Agent home partition filling; upload backlog. Investigate why OMS is slow to accept.
- **`Data Upload State: FAILED`** — Multiple root causes; check `gcagent.err`.
- **Agent won't start — `emd.properties parse error`** — Config edited by hand; restore from backup.
- **Discovery finds nothing** — Wrong `oratab` or agent OS user has no read on the DB installation. Add to `oinstall` group.
- **Duplicate targets show up** — RAC misdiscovered per node. Delete duplicates from console.

## Best Practices

1. One agent per host — never share $AGENT_HOME across hosts.
2. Keep agent OS user separate (`oemagent`) with sudo rules — avoid running as `oracle`.
3. Monitor agent health from OMS itself: `oracle_emd` target availability.
4. Set an alert for `Available disk space` < 20% on agent home.
5. Rotate agent TLS certs every 12 months.
6. Blackout targets **before** any change window.
7. Upgrade agents within 60 days of an OMS upgrade.
8. Version-control your agent-deployment automation (Ansible / SaltStack).
9. Keep `AGENT_HOME` and `AGENT_INST` on same filesystem — required.
10. On RAC, each node has its own agent — never share the agent binary.

## Interview Questions

1. **Q:** How does an agent communicate with OMS?
   **A:** HTTPS uploads on OMS's upload port (4903 default), signed with the agent's certificate.

2. **Q:** What's the difference between AGENT_HOME and AGENT_INST?
   **A:** `AGENT_HOME` = binaries. `AGENT_INST` = per-agent state, config, metric cache.

3. **Q:** How do you silently deploy an agent?
   **A:** `agentDeploy.sh` with `AGENT_BASE_DIR`, `OMS_HOST`, `AGENT_REGISTRATION_PASSWORD`, etc.

4. **Q:** OMS was down 6 hours — does the agent lose data?
   **A:** No — the agent caches in `AGENT_INST/sysman/emd/upload/` and drains on reconnect.

5. **Q:** How do you blackout a target from CLI?
   **A:** `emctl start blackout NAME <target=type>[,...]` on the host.

## References

- Oracle Enterprise Manager Cloud Control Basic Installation Guide 13c
- MOS Doc ID 1360083.1 — Agent Deployment
- MOS Doc ID 2144385.1 — Agent Troubleshooting
- MOS Doc ID 1541126.1 — OEM HA (agent side)
