# Redo Transport

## Overview

**Redo transport** is how primary redo reaches the standby. The mode you choose defines your data loss potential and your primary latency. This is one of the highest-impact Data Guard decisions.

## Modes

### SYNC (AFFIRM)

- LGWR ships redo synchronously.
- Primary waits for standby's RFS to write redo to Standby Redo Logs **AND flush to disk** (AFFIRM).
- Zero data loss.
- Highest latency — primary commit blocks on network + standby I/O.
- Required for **Maximum Protection**.

### FASTSYNC (SYNC/NOAFFIRM)

- LGWR ships synchronously.
- Standby acks on **receipt** (before disk flush).
- Near-zero data loss (small race window on standby crash).
- Lower latency than SYNC.
- Allowed in **Maximum Availability** mode (12c+).

### ASYNC (LGWR ASYNC)

- LGWR posts NSA (Network Server Async) which ships redo asynchronously.
- Primary commit doesn't wait.
- Non-zero data loss window.
- Lowest latency — no standby overhead on primary.
- Default for **Maximum Performance**.

## Configuration

Set at `LOG_ARCHIVE_DEST_n` on primary:

```sql
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
  'SERVICE=prod_dr
   ASYNC
   REOPEN=30
   VALID_FOR=(ONLINE_LOGFILE,PRIMARY_ROLE)
   DB_UNIQUE_NAME=prod_dr
   NET_TIMEOUT=30
   MAX_CONNECTIONS=1
   COMPRESSION=DISABLE';
```

Broker equivalent:

```
DGMGRL> EDIT DATABASE 'prod_dr' SET PROPERTY 'LogXptMode' = 'SYNC';
DGMGRL> EDIT CONFIGURATION SET PROTECTION MODE AS MAXAVAILABILITY;
```

## Transport Attributes

| Attribute                 | Purpose                                           |
| ------------------------- | ------------------------------------------------- |
| `SERVICE=name`            | Standby TNS service                               |
| `SYNC / ASYNC / FASTSYNC` | Mode                                              |
| `AFFIRM / NOAFFIRM`       | Disk flush confirmation                           |
| `NET_TIMEOUT=<sec>`       | Network wait timeout                              |
| `MAX_CONNECTIONS=n`       | Parallel redo transport (helps WAN)               |
| `REOPEN=<sec>`            | Retry delay after failure                         |
| `MAX_FAILURE=n`           | Max retries before defer                          |
| `COMPRESSION=ENABLE`      | Compress redo over network (Advanced Compression) |
| `ENCRYPTION` (via TCPS)   | TLS transport                                     |
| `VALID_FOR=(...)`         | Log type + role scope                             |

## Latency Impact — SYNC vs ASYNC

Primary `log file sync` includes:

- Local LGWR flush.
- Network round-trip to standby (SYNC).
- Standby write to SRL (SYNC).
- Standby ack (SYNC/AFFIRM: after flush; NOAFFIRM: after receipt).

Rough numbers on a well-tuned setup:

- Local flush: 0.5–2 ms.
- Same-DC network: +0.2 ms.
- Cross-metro (< 100 km): +2–5 ms.
- Cross-country: +30–80 ms.
- Cross-continent: +100+ ms.

For cross-continent, SYNC is usually impractical — ASYNC or FASTSYNC preferred.

## Redo Streams (12c+)

Multiple slaves can ship redo concurrently: `LGnn` scalable LGWR + `NSAn` slaves. Improves throughput on high-DML systems.

## Diagnostic Queries

```sql
-- On primary — destination status
SELECT dest_id, dest_name, target, transmit_mode, affirm,
       async_blocks, net_timeout, log_sequence, applied_seq,
       error
FROM   v$archive_dest
WHERE  target = 'STANDBY';

-- Transport lag
SELECT name, value, unit FROM v$dataguard_stats
WHERE  name IN ('transport lag','apply lag');

-- RFS activity on standby
SELECT process, status, sequence#, block#, delay_mins
FROM   v$managed_standby
WHERE  process = 'RFS';

-- Physical write throughput
SELECT event, total_waits,
       ROUND(time_waited_micro/1e6, 1) AS sec
FROM   v$system_event
WHERE  event LIKE 'ARCH%' OR event LIKE 'LGWR%' OR event LIKE 'log file sync%';
```

## Common Issues

- **`ORA-16401: archivelog rejected by RFS`** — Duplicate sequence; benign in some restart scenarios.
- **`ORA-16086: standby database does not contain available standby log files`** — Missing SRLs.
- **`ORA-03135: connection lost contact`** — Network drop. Adjust `NET_TIMEOUT` and check LAN.
- **High `log file sync` after enabling SYNC** — Storage or network too slow for chosen mode. Consider FASTSYNC.
- **Standby lag growing under ASYNC** — Network bandwidth insufficient or standby I/O slow.

## Best Practices

1. **SYNC for zero-data-loss requirements** and low-latency networks.
2. **FASTSYNC as middle ground** for < 100 km links.
3. **ASYNC for WAN or high-throughput** where small data loss is acceptable.
4. Configure **Compression** on WAN destinations (Advanced Compression).
5. Encrypt redo transport in transit (TCPS or native encryption).
6. Multiple `MAX_CONNECTIONS` on WAN for parallelism.
7. `NET_TIMEOUT = 30` seconds — bounds primary blocking on network issues.
8. Monitor `V$DATAGUARD_STATS.transport lag`.
9. Have alternate paths — cascading standby, or dual-DG hub.
10. Test transport during peak workload — measure `log file sync` impact.

## Interview Questions

1. **Q:** SYNC vs ASYNC?
   **A:** SYNC waits for standby ack (zero data loss). ASYNC doesn't wait (small loss window).

2. **Q:** FASTSYNC?
   **A:** SYNC/NOAFFIRM — standby acks on receipt not disk flush. Faster than SYNC, near-zero loss.

3. **Q:** Protection modes and transport?
   **A:** Max Protection = SYNC/AFFIRM. Max Availability = SYNC or FASTSYNC. Max Performance = ASYNC.

4. **Q:** Why compression?
   **A:** Reduces network bandwidth for WAN destinations. Advanced Compression Option required.

5. **Q:** How to see transport lag?
   **A:** `V$DATAGUARD_STATS` name='transport lag'.

## References

- Oracle Data Guard Concepts and Administration 19c
- MOS Doc ID 220970.1 — Redo Transport Troubleshooting
- MOS Doc ID 1265700.1 — DG Best Practices
