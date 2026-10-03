# OEM OMS

## Overview

The **Oracle Management Server (OMS)** is the middle tier of Enterprise Manager — a Weblogic-hosted Java web application. It ingests metric uploads from agents, runs the console UI, orchestrates jobs, evaluates alerting rules, and reads/writes the repository. Running OMS operationally is largely running a Weblogic domain.

## What Runs Inside

An OMS installation is really:

- **Weblogic AdminServer** (`EMGC_ADMINSERVER`) — administration.
- **Managed Server** (`EMGC_OMS1`) — the OEM app.
- **Node Manager** — starts/stops the AdminServer and managed servers.
- **HTTP Server (OHS)** — front door, TLS termination.
- **BI Publisher** (`BIP`) — reports.
- **JVMD Engine** — Java Virtual Machine Diagnostics.
- **ADP Engine** — Application Diagnostics Platform (older).

## Directory Layout

```
$MIDDLEWARE_HOME/         # Weblogic + FMW
  oms/                    # OMS installation
    bin/emctl             # OMS control script
  oracle_common/          # shared FMW libs
$GC_INST/                 # instance state (per-node)
  em/
    OMSLogs/              # $OMS_LOGS
    EMGC_OMS1/logs/       # managed server logs
$AGENT_INST/              # local agent for OMS host (managing itself)
```

Environment on every OMS host:

```bash
export MW_HOME=/u01/oracle/middleware
export OMS_HOME=$MW_HOME/oms
export AGENT_HOME=/u01/oracle/agent
export ORACLE_HOME=$OMS_HOME       # for emctl
export PATH=$OMS_HOME/bin:$AGENT_HOME/bin:$PATH
```

## `emctl` — OMS Control

`emctl` covers both OMS and Agent — different subcommands per install.

### Status

```bash
emctl status oms
emctl status oms -details

# Weblogic side
emctl status oms -component AdminServer
```

Typical output — everything ONLINE:

```
Oracle Enterprise Manager Cloud Control 13c Release 5
...
Console Server Host        : oms1.example.com
HTTP Console Port          : 7802 (secure)
JVMD Engine                : Up
Oracle Management Server Instance Home: /u01/oracle/gc_inst/em/EMGC_OMS1
Oracle Management Server Console URL: https://oms1.example.com:7803/em
Oracle Management Server is Up
Managed Server Host        : oms1.example.com
```

### Start / Stop

```bash
emctl start oms
emctl stop oms
emctl stop oms -all                # stop OHS, Weblogic AdminServer, etc.
emctl stop oms -all -force         # force-kill
emctl start oms -force
```

### Restart Sequence (Full)

```bash
emctl stop oms -all
# check
ps -ef | grep -E 'weblogic|EMGC'
emctl start oms
```

### Log Locations

```bash
# Managed server (application logs)
$GC_INST/em/EMGC_OMS1/logs/EMGC_OMS1.out
$GC_INST/em/EMGC_OMS1/logs/EMGC_OMS1.log
$GC_INST/em/EMGC_OMS1/logs/emoms.log
$GC_INST/em/EMGC_OMS1/logs/emoms.trc

# AdminServer
$GC_INST/user_projects/domains/GCDomain/servers/EMGC_ADMINSERVER/logs/

# OHS
$GC_INST/user_projects/domains/GCDomain/servers/ohs1/logs/
```

`emoms.log` is where OMS application errors show up — most useful.

## Configuring the OMS

### Change Repository Connect String

```bash
emctl config oms -store_repos_details \
    -repos_conndesc "(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=repo.host)(PORT=1521))(CONNECT_DATA=(SERVICE_NAME=EMREP)))" \
    -repos_user sysman
```

### Set OMS Preferred Repository Password

```bash
emctl config oms -change_repos_pwd -old_pwd oldpw -new_pwd newpw
```

### Increase JVM Heap

Managed server heap starts at 1740 MB — bump for busy sites:

```bash
emctl set property -name 'JAVA_EM_MEM_ARGS' \
    -value '-Xms1024m -Xmx4096m -XX:MaxPermSize=1024m'

# For a specific managed server
emctl restart oms
```

Or via Weblogic console → managed server → Server Start → Arguments.

### Change Console Port

```bash
emctl config oms -change_console_port -console_https_port 7802 -console_http_port 7803
```

### Add / Remove OMS Node (Multi-OMS)

Second OMS install starts as fresh:

```bash
# On new host
./em13500_linux64.bin -novalidation SKIP_SW_UPDATE_CHECK=true \
    b_startOMS=false ORACLE_HOSTNAME=oms2.example.com \
    ADDITIONAL_OMS_INSTALL=true
```

Then:

```bash
emctl config oms -store_repos_details ...
emctl config emrep -conn_desc ...
emctl start oms
```

Add to the Server Load Balancer VIP.

## SLB and TLS

Multi-OMS sits behind an SLB (F5, HAProxy, AWS ALB, etc.). Sockets TLS-terminated at the SLB or passed through:

- **Pass-through** — SLB doesn't inspect; OMS's own cert.
- **TLS termination** — SLB has cert, re-encrypts to OMS.

Config:

```bash
emctl config oms -host slb.example.com \
    -console_secure_port 443 -upload_port 4903
```

Rotate TLS cert every year:

```bash
emctl secure oms -host slb.example.com \
    -reg_pwd <registration_pwd> -sysman_pwd <sysman_pwd>
emctl secure agent -host slb.example.com
```

## Backup

An OMS is stateful in three places:

- **Repository DB** — RMAN handles.
- **`$GC_INST`** — instance config, credentials. Filesystem backup.
- **`$OMS_HOME/sysman/config/`** — encrypted secrets. Filesystem backup.

Recommended: nightly RMAN of repo + weekly tar of `$GC_INST` and `$OMS_HOME/sysman/config`.

## Diagnostic Commands

```bash
# Recent traffic
tail -f $GC_INST/em/EMGC_OMS1/logs/emoms.trc

# Repository connectivity
emctl status oms -details -sysman_pwd <pw>

# Weblogic health
$MW_HOME/oracle_common/common/bin/wlst.sh
wls:/> connect('weblogic', 'pw', 't3://oms1:7101')
wls:/> servers()

# Notification thread health
emctl get property -name oracle.sysman.core.notification.numthreads
```

## Common Issues

- **OMS won't start — `oraclehostname` mismatch** — `/etc/hosts` and `ORACLE_HOSTNAME` differ. Fix `/etc/hosts` and re-run `emctl secure oms`.
- **Weblogic AdminServer stuck starting** — Node Manager credential issue. Reset with `resetNMPassword.sh`.
- **`emctl status oms` shows OMS Up but console 503** — Managed server crashed; check `EMGC_OMS1.out`.
- **Agent uploads failing across the fleet** — TLS cert expired. `emctl secure oms` then `emctl secure agent` on all agents.
- **OMS crashes with OOM** — Bump heap; check for a runaway metric extension.
- **Repository connections exhausted** — `sessions` param too small; bump on repo DB.
- **Console slow** — Repository IO saturated, or nightly stats never ran on SYSMAN. Bounce OMS, gather SYSMAN stats.

## Best Practices

1. Two OMS nodes minimum in production.
2. Repository DB on **separate host** with its own storage.
3. Rotate TLS certs annually before expiry — set an OEM notification for cert expiry.
4. Backup `$GC_INST` and `$OMS_HOME/sysman/config` weekly.
5. Monitor OMS heap usage; alert at 85%.
6. Bounce OMS during a change window monthly — Weblogic likes fresh JVMs.
7. Keep managed server logs on a separate mount so they don't fill `$OMS_HOME`.
8. Set `Log Rotation Size` on managed server — default rolls at 5 MB, too small.
9. Test failover: stop one OMS, verify agents/console still work through SLB.
10. Document `sysman` password rotation procedure — it's involved.

## Interview Questions

1. **Q:** What's the difference between OMS and Weblogic AdminServer?
   **A:** OMS is the OEM application; AdminServer is Weblogic's administration server that hosts the OEM Managed Server.

2. **Q:** How do you restart just the OMS app without touching Weblogic AdminServer?
   **A:** `emctl stop oms` then `emctl start oms` — bounces just the managed server.

3. **Q:** How do you scale OMS horizontally?
   **A:** Install another OMS on a second host, register with same repository, put both behind an SLB.

4. **Q:** Where does OMS store its encrypted config secrets?
   **A:** `$OMS_HOME/sysman/config/` — filesystem-encrypted; must be backed up.

5. **Q:** Where do OMS application errors log?
   **A:** `$GC_INST/em/EMGC_OMS1/logs/emoms.log` and `emoms.trc`.

## References

- Oracle Enterprise Manager Cloud Control Administrator's Guide 13c
- MOS Doc ID 2101223.2 — OEM 13.5 Master Note
- MOS Doc ID 1489868.1 — `emctl` reference
- MOS Doc ID 1541126.1 — Multi-OMS setup
