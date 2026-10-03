# Exadata Cloud — Monitoring

## Overview

Exadata monitoring spans **DB Server** (compute node), **Storage Cell**, and **cluster-wide** metrics. Most standard Oracle monitoring works plus Exadata-specific views.

## Key Views (DB-Side)

```sql
-- Cells known to this instance
SELECT cell_name, cell_type, status, cell_path FROM v$cell;

-- Cell IO stats
SELECT   name, value FROM v$sysstat
WHERE    name LIKE 'cell%'
ORDER BY value DESC;

-- Cell offload benefit
SELECT   name, value FROM v$sysstat
WHERE    name IN ('cell physical IO bytes eligible for predicate offload',
                  'cell physical IO interconnect bytes',
                  'cell physical IO interconnect bytes returned by smart scan',
                  'cell physical IO bytes saved by storage index');

-- SQL benefit
SELECT   sql_id, executions, io_cell_offload_eligible_bytes/1024/1024 elig_mb,
         io_cell_offload_returned_bytes/1024/1024 returned_mb,
         ROUND(io_cell_offload_returned_bytes*100 /
               NULLIF(io_cell_offload_eligible_bytes,0),1) return_pct
FROM     v$sql
WHERE    io_cell_offload_eligible_bytes > 0
ORDER BY io_cell_offload_eligible_bytes DESC
FETCH FIRST 20 ROWS ONLY;
```

Low `return_pct` = high offload = good.

## Cell-Side Metrics

From a compute node:

```bash
# Push CellCLI command to all cells
dcli -g cellhosts.txt -l celladmin \
     "cellcli -e list metriccurrent \
        attributes name,metricValue,alertState \
        where name like 'CD_IO_.*' \
          and metricValue > 0"
```

Common metric families:

- `CD_IO_*` — cell disk IO.
- `FC_IO_*` — flash cache IO.
- `FD_IO_*` — flash disk IO.
- `DB_IO_*` — per-database IO.
- `IORM_MODE`, `IORM_WAITTIME` — I/O Resource Manager.
- `SMARTIO_*` — Smart Scan metrics.

## Alerts

Cells raise **stateful alerts** on threshold breach:

```bash
dcli -g cellhosts.txt -l celladmin \
     "cellcli -e list alerthistory \
        where alertState = 'open' \
          and severity in ('critical','warning')"
```

Common alert types:

- Physical disk failure.
- Flash card failure.
- HW temperature.
- Cell software crash.
- IORM plan violation.
- Battery / PSU.

## OEM Exadata Plugin

OEM has an Exadata plugin (13.x). Displays:

- Cell topology.
- Per-cell health.
- Smart Scan effectiveness heatmap.
- IORM plan compliance.
- HCC ratios.

## MS API

MS exposes metrics via REST:

```bash
curl -k -u dbmadmin:pw https://cellhost1:2020/monitor/metric_history
```

Returns JSON of metric samples.

## Exachk

Oracle's holistic Exadata health checker. Run periodically:

```bash
$GRID_HOME/suptools/exachk/exachk
```

Produces HTML report with:

- Firmware level compliance.
- Cell software version consistency.
- OS parameter compliance.
- HW health.
- Best-practice violations.

Run monthly minimum. Address `CRITICAL` and `WARNING` findings.

## AWR + ASH on Exadata

Standard AWR reports include Exadata-specific sections:

- Cell IO Summary.
- Cell IO Detail per DG.
- Smart Scan benefit.
- HCC compression rates.

Generate:

```sql
SELECT DBMS_WORKLOAD_REPOSITORY.AWR_REPORT_HTML(
       (SELECT dbid FROM v$database),
       1, &start_snap, &end_snap)
FROM   dual;
```

## IORM Monitoring

```sql
SELECT * FROM v$sysmetric WHERE metric_name LIKE '%IORM%';
```

Or CellCLI:

```
CellCLI> LIST METRICHISTORY WHERE name LIKE 'IORM.*'
```

## Alerts You Want in Your NOC

- Any cell alert = CRITICAL or WARNING.
- Grid disk STATUS != ACTIVE.
- Smart Scan return% > 30% consistently (query issue).
- HCC ratios dropping (data pattern change).
- Cell CPU > 80% sustained.

## Related

- [Overview](overview.md).
- [Storage Cells](storage-cells.md).
- [Smart Scan](smart-scan.md).
- [AWR](../../12-performance-tuning/awr.md).
