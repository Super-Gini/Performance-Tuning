# ADR — Automatic Diagnostic Repository

## Overview

The **Automatic Diagnostic Repository (ADR)** is Oracle's unified diagnostic filesystem, introduced in 11g. Before ADR, every product (RDBMS, ASM, listener, tools) wrote traces to its own directory conventions. ADR standardizes: **one root, one command-line tool (`adrci`), one retention policy, one packaging format**.

Every Oracle server product on the host stores its alerts, traces, incidents, and health-monitor findings under `$ORACLE_BASE/diag/<product>/<instance>/`.

## ADR Base and Homes

- **ADR Base** = `$ORACLE_BASE` (usually `/u01/app/oracle`).
- **ADR Home** = per-product-per-instance directory: `diag/rdbms/<dbname>/<sid>/`, `diag/asm/+asm/+ASM1/`, `diag/tnslsnr/<host>/listener/`.

```
$ORACLE_BASE/
└── diag/
    ├── rdbms/prd/PRD1/
    │   ├── alert/
    │   │   └── log.xml
    │   ├── trace/
    │   │   ├── alert_PRD1.log
    │   │   ├── PRD1_ora_1234.trc
    │   │   └── PRD1_ora_1234.trm
    │   ├── incident/
    │   │   └── incdir_100001/
    │   │       ├── PRD1_ora_5678_i100001.trc
    │   │       └── ...
    │   ├── incpkg/
    │   ├── cdump/
    │   ├── hm/
    │   ├── metadata/
    │   ├── stage/
    │   └── sweep/
    ├── asm/+asm/+ASM1/
    ├── tnslsnr/<host>/listener/
    └── clients/user_oracle/host_.../
```

Subdirectories:

| Dir         | Contents                                      |
| ----------- | --------------------------------------------- |
| `alert/`    | XML alert log (`log.xml`).                    |
| `trace/`    | Text alert log + trace files.                 |
| `incident/` | Per-incident subdirs with trace files.        |
| `incpkg/`   | ZIP packages built by `ips generate package`. |
| `cdump/`    | Core dumps (rare).                            |
| `hm/`       | Health Monitor reports.                       |
| `metadata/` | ADR metadata (internal).                      |
| `stage/`    | Staging area for uploads.                     |
| `sweep/`    | For MOS Automatic Diagnostic Extraction.      |

## `adrci` — Command-Line Tool

Interactive:

```bash
adrci
```

Session commands:

```
adrci> show homes                              # list ADR homes
adrci> set homepath diag/rdbms/prd/PRD1        # scope to one home
adrci> show alert -tail 100                    # last 100 alert entries
adrci> show alert -term                        # follow in terminal
adrci> show incident                           # list incidents
adrci> show incident -mode DETAIL -last 1      # detail of latest
adrci> show trace <trace_file>                 # dump a trace file
adrci> show tracefile -t                       # sort by time
adrci> show control                            # retention config
```

Non-interactive from shell:

```bash
adrci exec="show alert -term -tail 50"
adrci exec="show incident -last 5"
```

## Retention Policy

Two policies:

- **`SHORTP_POLICY`** — hours, for **short-life** files: trace, alert, cdump. Default 720 h (30 days).
- **`LONGP_POLICY`** — hours, for **long-life** files: incident, HM. Default 8760 h (365 days).

```bash
adrci> set homepath diag/rdbms/prd/PRD1
adrci> show control
adrci> set control (SHORTP_POLICY = 720)
adrci> set control (LONGP_POLICY  = 8760)
```

## Purging

Automatic purge runs periodically (controlled by `_ADR_AUTO_PURGE`). Manual:

```bash
adrci> purge -age 4320               # older than 6 months (short-lived)
adrci> purge -age 8760 -type incident # older than 1 year (incidents)
adrci> purge -age 720                # entire home
```

## Health Monitor

ADR includes Oracle's **Health Monitor (HM)** — a checker framework that inspects the DB and reports findings.

```sql
-- Available checks
SELECT   name, description
FROM     v$hm_check
ORDER BY name;

-- Run a check
BEGIN
  DBMS_HM.RUN_CHECK('DB Structure Integrity Check');
END;
/

-- View results
SELECT run_id, name, status
FROM   v$hm_run
ORDER  BY start_time DESC;

-- Get the report
SELECT DBMS_HM.GET_RUN_REPORT('&run_id')
FROM   dual;
```

Common checks:

- `DB Structure Integrity Check` — datafile/control-file consistency.
- `Data Block Integrity Check` — verify blocks.
- `Redo Integrity Check` — walk redo streams.
- `Dictionary Integrity Check` — dictionary consistency.
- `Undo Segment Integrity Check`.
- `ASM Allocation Check`.

Runs on-demand or periodically via `MMON`.

## Incident Packaging Service (IPS)

Bundles an incident + related traces + alert log excerpts + config info into a ZIP for MOS:

```bash
adrci> ips create package incident 100001
adrci> ips add file /u01/app/oracle/upgrade.log package 3
adrci> ips generate package 3 in /tmp
```

Produces `IPSPKG_20260806_100001_ORA00600_..._0000.zip` — attach to SR.

## Sweep Directory

If your DB is registered with My Oracle Support's Auto-Service Request (ASR), ADR's `sweep/` directory receives issues auto-uploaded by MOS agents. Rarely touched manually.

## Interfacing with Views

| View                         | Purpose                                          |
| ---------------------------- | ------------------------------------------------ |
| `V$DIAG_INFO`                | ADR paths and settings for current instance.     |
| `V$DIAG_ALERT_EXT`           | External-table view of `log.xml`.                |
| `V$DIAG_INCIDENT`            | Incidents.                                       |
| `V$DIAG_PROBLEM`             | Problems (deduplicated incident families).       |
| `V$DIAG_TRACE_FILE`          | Trace files in current ADR home.                 |
| `V$DIAG_TRACE_FILE_CONTENTS` | Contents (19c+) — join with `V$DIAG_TRACE_FILE`. |
| `V$HM_CHECK`                 | HM checks available.                             |
| `V$HM_RUN`                   | HM run history.                                  |
| `V$HM_FINDING`               | Findings from HM runs.                           |

Example — a single query for recent trace file content:

```sql
SELECT tf.trace_filename, tfc.payload
FROM   v$diag_trace_file          tf
JOIN   v$diag_trace_file_contents tfc USING (adr_home, trace_filename)
WHERE  tfc.timestamp > SYSDATE - 1/24
   AND tfc.payload LIKE '%ORA-01555%';
```

## Common Issues

- **`adrci` reports no homes** — Wrong `$ORACLE_BASE` in the shell.
- **ADR fills disk** — Retention too generous; enable purge, `SHORTP_POLICY` and `LONGP_POLICY` in hours.
- **HM check `Dictionary Integrity Check` returns findings** — Real issue. Open SR.
- **IPS package huge** — Add `-in <dir>` with sufficient space.
- **`ORA-48141 error creating directory`** — Perms on `$ORACLE_BASE/diag`. `chown oracle:oinstall`.
- **`ORA-48122` empty ADR home** — SID mismatch or startup error left ADR uninitialized. Bounce the instance.

## Best Practices

1. Set `$ORACLE_BASE` in every DBA shell profile — nothing works without it.
2. Configure `SHORTP_POLICY` and `LONGP_POLICY` to your retention policy.
3. Run `adrci -> purge -age <hrs>` monthly as a scheduled job.
4. Run `HM DB Structure Integrity Check` weekly.
5. Ship the XML alert log to your SIEM (Splunk / ELK / Sentinel).
6. Use `IPS` to prepare SR uploads — never manually ZIP traces.
7. Keep `/u01/app/oracle/diag` on its own filesystem — one bug shouldn't fill the DB filesystem.
8. Grant read on `diag/` to your monitoring users; don't share `oracle` OS password.
9. Never `rm -rf` under `diag/` while the DB is open.
10. On RAC, verify each node's ADR is separate — a shared filesystem for ADR is not supported.

## Interview Questions

1. **Q:** What is ADR?
   **A:** A unified diagnostic filesystem under `$ORACLE_BASE/diag/` for all Oracle products, with a common CLI (`adrci`) and packaging tool (IPS).

2. **Q:** What are the two purge policies?
   **A:** `SHORTP_POLICY` (short-lived: trace, alert, cdump) and `LONGP_POLICY` (long-lived: incident, HM).

3. **Q:** How do you get a support package to Oracle?
   **A:** `adrci -> ips create package incident <id> -> ips generate package <n> in /tmp` — upload the ZIP.

4. **Q:** What is Health Monitor?
   **A:** ADR's integrity-check framework; runs on-demand and periodically to catch dictionary, redo, undo, block corruptions.

5. **Q:** How do you query the XML alert log via SQL?
   **A:** `SELECT ... FROM V$DIAG_ALERT_EXT WHERE message_text LIKE '%...%'`.

## References

- Oracle Database Administrator's Guide 19c — Managing Diagnostic Data
- MOS Doc ID 422893.1 — Diagnosability Framework
- MOS Doc ID 443529.1 — Trace / ADR management
- MOS Doc ID 738732.1 — ADRCI reference
