# Exadata Storage Cells

## What They Are

Storage cells are dedicated 1U servers running Oracle's **Cell Software** (Exadata Storage Server). They aren't generic storage arrays — they run Oracle-specific software that offloads SQL processing.

Each cell has:

- CPUs (used to run Smart Scan).
- RAM (used for cell memory + PMEM cache).
- NVMe SSDs.
- Optional PMEM (persistent memory) tier.
- 100 or 200 Gbps InfiniBand / RoCE ports.

## Software

- **cellsrv** — main daemon; handles I/O requests.
- **RS (Restart Server)** — monitors cellsrv, restarts on crash.
- **MS (Management Server)** — REST/CLI interface.
- **DBMCLI / CELLCLI** — CLI tools.

Access:

```bash
# From a compute node
ssh -i /var/opt/oracle/dbaastools/cellip.ora cellhost1
$ dcli -c c1,c2,c3 "cellcli -e list cell"
```

## Key Features

### Smart Scan

Storage cells filter and project rows before returning to compute — DB gets only the rows it needs.

Example:

```sql
SELECT id, name FROM sales WHERE amount > 10000;
```

Without Smart Scan: entire block returned to compute; compute filters.
With Smart Scan: cell evaluates `amount > 10000`, returns only matching rows.

See [Smart Scan](smart-scan.md).

### Storage Indexes

Automatic per-cell min/max metadata for each 1 MB storage region per column. Cells skip regions where the predicate can't match.

Zero DBA config. Kicks in automatically on Smart Scan.

### Hybrid Columnar Compression (HCC)

Cells decompress on read; compute doesn't do it. Enables 10–50× compression while still allowing SQL access.

```sql
ALTER TABLE sales COMPRESS FOR QUERY HIGH;
```

Levels:

- `QUERY LOW` — LZO-like, ~4×.
- `QUERY HIGH` — Zlib, ~10×.
- `ARCHIVE LOW` — Zlib higher ratio.
- `ARCHIVE HIGH` — BZIP2, ~50×.

### Smart Flash Cache

NVMe / PMEM tier in front of spinning-media (older Exadata; NVMe-only now). Transparent to Oracle.

### IORM (I/O Resource Manager)

Priorities I/O across databases sharing the cells:

```
CellCLI> ALTER IORMPLAN objective=balanced
CellCLI> ALTER IORMPLAN dbplan=(...)
```

## Cell Health

From a compute node:

```bash
dcli -c c1,c2,c3 "cellcli -e list cell attributes name,status,cellversion"
dcli -c c1,c2,c3 "cellcli -e list physicaldisk attributes name,status where status <> 'normal'"
dcli -c c1,c2,c3 "cellcli -e list flashcache detail"
```

Or DB-side:

```sql
SELECT   cell_name, status, versions
FROM     v$cell;

SELECT   name, value
FROM     v$sysstat
WHERE    name LIKE 'cell%';
```

## Failure Modes

- Cell down: ASM handles via NORMAL/HIGH redundancy — no user impact.
- Cell partial (disks failing): cells auto-drop bad disks; ASM rebalances.
- Cell HA: minimum 3 cells (Base X10M) with HIGH ASM redundancy.

## Related

- [Overview](overview.md).
- [Celldisk / Griddisk](celldisk-griddisk.md).
- [Monitoring](monitoring.md).
- [ASM](../../19-asm/asm-architecture.md).
