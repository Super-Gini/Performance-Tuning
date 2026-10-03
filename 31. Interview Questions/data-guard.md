# Data Guard — Interview Questions

**Q: What is Data Guard?**
A: Oracle's disaster recovery / HA solution — maintains a **standby database** by shipping and applying redo from the **primary**. Supports zero-data-loss and near-zero-downtime failover.

**Q: Types of standby?**
A: **Physical Standby** — block-for-block copy of primary, applies redo directly (MRP0). **Logical Standby** — extracts SQL from redo and reapplies (LSP); allows different structure on standby. **Snapshot Standby** — temporarily open read-write for testing, discards changes on switch back.

**Q: Data Guard protection modes?**
A: **MAX PERFORMANCE** — ASYNC, minimal impact, small data loss window. **MAX AVAILABILITY** — SYNC when possible, temporarily ASYNC when standby unreachable, zero data loss guaranteed at commit time. **MAX PROTECTION** — SYNC always; primary halts if standby unreachable.

**Q: SYNC vs ASYNC?**
A: **SYNC** — primary LGWR waits for standby to ACK receipt before committing. Zero data loss but adds latency (network round trip). **ASYNC** — primary doesn't wait; small window of possible loss.

**Q: What is Real-Time Apply?**
A: Standby applies redo as it arrives, before the log switches, using standby redo logs. Reduces apply lag near-zero. Enable: `ALTER DATABASE RECOVER MANAGED STANDBY DATABASE USING CURRENT LOGFILE DISCONNECT`.

**Q: What are standby redo logs?**
A: A separate set of redo logs on the standby that receive redo from the primary. Sized ≥ primary's online redo. Needed for real-time apply and for SYNC transport.

**Q: What is the Data Guard Broker?**
A: A management framework (`dgmgrl`) that automates configuration, monitoring, and role transitions. Uses its own config files on both sides; commands abstract raw `ALTER DATABASE`.

**Q: Switchover vs failover?**
A: **Switchover** — planned, both DBs healthy, roles swap cleanly, zero data loss. **Failover** — unplanned, primary is lost, standby becomes primary; potentially small data loss (ASYNC).

**Q: What is FSFO?**
A: **Fast-Start Failover** — automatic failover coordinated by the Broker + observer process. On sustained primary loss, observer triggers failover to designated standby.

**Q: What is Active Data Guard?**
A: License add-on that lets the physical standby be **open read-only while applying redo**. Reports run against a live standby.

**Q: How do you monitor DG?**
A: `V$DATAGUARD_STATS` (apply/transport lag), `V$MANAGED_STANDBY` (MRP0/RFS state), `V$ARCHIVE_GAP`, `V$DATAGUARD_STATUS`. Broker: `DGMGRL> SHOW CONFIGURATION VERBOSE`.

**Q: How do you fix an archive gap?**
A: `V$ARCHIVE_GAP` shows sequences missing. Fetch from primary and `ALTER DATABASE REGISTER LOGFILE 'path'`. Broker often auto-resolves. RMAN `RECOVER STANDBY DATABASE FROM SERVICE primary` for large gaps.

**Q: What is redo apply parallelism?**
A: MRP0 can spawn slaves: `ALTER DATABASE RECOVER MANAGED STANDBY DATABASE PARALLEL 4 DISCONNECT`. Speeds up apply on multi-CPU standby.

**Q: What is the difference between physical and logical standby?**
A: Physical = block-level identical; standby is exact copy. Logical = redo mined for SQL, reapplied — structure can differ (extra indexes, MVs, different columns).

**Q: What are typical Data Guard use cases?**
A: DR (primary datacenter loss), Active DG reporting, rolling upgrades, migration to new hardware/cloud, testing (snapshot standby).

**Q: Cascaded standby?**
A: A standby that receives redo from another standby, not primary. Useful for regional distribution or bandwidth-limited primary.

**Q: `MAX AVAILABILITY` vs `MAX PROTECTION` — practical choice?**
A: MAX AVAILABILITY is the common production choice — zero data loss when standby is up, degrades gracefully. MAX PROTECTION halts the primary on standby loss — too risky for most workloads.

## Related

- [Data Guard Architecture](../17-data-guard/architecture.md).
- [FSFO](../17-data-guard/fsfo.md).
- [Broker](../17-data-guard/broker.md).
- [Data Guard Lag runbook](../27-runbooks/data-guard-lag.md).
