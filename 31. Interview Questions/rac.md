# RAC — Interview Questions

**Q: What is Oracle RAC?**
A: **Real Application Clusters** — multiple instances on different servers sharing a single database. Provides HA (survives instance/node failure) and scale-out. Requires shared storage (ASM) and cluster interconnect.

**Q: What is Cache Fusion?**
A: RAC's mechanism to share cache between instances over the interconnect. When instance B needs a block that instance A has, LMS (Global Cache Server) ships it — no disk round trip.

**Q: What is GCS?**
A: **Global Cache Service** — manages block-level coordination (which instance owns each block). Runs `LMS0..N` processes.

**Q: What is GES?**
A: **Global Enqueue Service** — manages non-cache resources (dictionary locks, library cache locks). Runs `LMD`, `LCK0`, `LMON`.

**Q: What is a voting disk?**
A: A shared file (on ASM) that clusterware uses to determine cluster membership. Every node writes heartbeat; majority sees each other = cluster is healthy. Minimum 3 (odd) for HA.

**Q: What is OCR?**
A: **Oracle Cluster Registry** — file storing cluster resource configuration (databases, services, listeners). Multiplexed. Managed by `ocrconfig`.

**Q: What's the difference between GI and Clusterware?**
A: **Grid Infrastructure** = install package including Clusterware + ASM. **Clusterware** = the cluster management software (CRSD, CSSD, OHASD) alone.

**Q: Name the main clusterware daemons.**
A: `OHASD` (Oracle High Availability Services — root of it all), `CSSD` (Cluster Sync — heartbeats, voting), `CRSD` (Cluster Ready Services — resource management), `EVMD` (Event Manager).

**Q: What causes a node eviction?**
A: CSSD miscount timeout — usually interconnect failure or resource starvation (CPU / IO). Also voting disk loss.

**Q: What is SCAN Listener?**
A: **Single Client Access Name** — cluster-level virtual IP name (typically 3 IPs). Clients connect to `scan-vip.example.com`; DNS returns rotating IP; SCAN listener redirects to a local node listener based on load.

**Q: What is Fast Application Notification (FAN)?**
A: A pub-sub event stream from clusterware to interested clients (JDBC / ODP.NET). Clients receive service-up/down events instantly, bypassing TCP timeouts. Enables Fast Connection Failover.

**Q: What is Fast Connection Failover (FCF)?**
A: Client-side feature — when RAC instance fails, JDBC pool receives FAN event, discards affected connections, opens new ones on surviving instances. Zero TCP timeout.

**Q: Difference between preferred and available service?**
A: **Preferred** — service normally runs on this instance. **Available** — accepts service if preferred goes down. Configure with `srvctl add service`.

**Q: What is TAF?**
A: **Transparent Application Failover** — client-side session failover. On instance loss, client session moves to another instance (with SELECT-level continuity or full session reconnect depending on config).

**Q: What is Application Continuity (12c+)?**
A: Newer than TAF — enables the DB to replay in-flight transactions on failover, transparent to app. Requires app to enable it (JDBC UCP with `ReplayInitiationTimeout`).

**Q: Cache fusion causes `gc buffer busy` — what to do?**
A: Application-level fixes: partition the hot table by hash so different instances hit different partitions, use sequences with `NOORDER CACHE` (larger cache), reduce cross-instance concurrent DML on hot rows.

**Q: How would you add a new node to a running cluster?**
A: `addnode.sh` in GI (silent or interactive), then verify with `cluvfy`, add DB instance via `dbca` or `srvctl add instance`, register services.

**Q: In RAC how do you kill a session on another instance?**
A: `ALTER SYSTEM KILL SESSION 'sid,serial#,@inst_id' IMMEDIATE;`

**Q: What files are on shared storage vs local?**
A: **Shared** — datafiles, control files, redo (usually), OCR, voting. **Local** — `$ORACLE_HOME` binaries (can be shared via ACFS but typically local), alert log, trace files, `oratab`.

## Related

- [RAC](../18-rac/index.md).
- [Cache Fusion](../18-rac/cache-fusion.md).
- [Evictions](../18-rac/evictions.md).
- [SCAN Listener](../18-rac/scan-listener.md).
