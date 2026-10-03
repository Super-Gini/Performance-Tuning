# Allocation Units

## Overview

An **allocation unit (AU)** is the smallest ASM allocation. Files are composed of multiple AUs distributed across all disks in a diskgroup. AU size affects I/O size, striping granularity, and metadata overhead.

Default AU is **1 MB**. Exadata uses **4 MB**. Larger AUs (16 MB, 64 MB) benefit very large databases and reduce metadata overhead but increase minimum file allocation.

## Choosing AU Size

Set at diskgroup creation via `au_size` attribute:

```sql
CREATE DISKGROUP data EXTERNAL REDUNDANCY
  DISK '/dev/asm/data01'
  ATTRIBUTE 'compatible.asm' = '19.0.0',
            'au_size' = '4M';
```

Valid values: 1, 2, 4, 8, 16, 32, 64 (MB).

Cannot change after creation.

## Trade-offs

- **Larger AU** — fewer AUs to track (lower metadata overhead); larger sequential I/O; wasteful for small files.
- **Smaller AU** — more granular allocation; higher metadata overhead per file.

For most OLTP: 1 MB or 4 MB.
For DW / VLDB: 4 MB or 16 MB.
Exadata: 4 MB by default.

## Diagnostic Queries

```sql
-- AU size per diskgroup
SELECT dg.name, a.value AS au_size_mb
FROM   v$asm_diskgroup dg JOIN v$asm_attribute a ON a.group_number = dg.group_number
WHERE  a.name = 'au_size';

-- File extent map (advanced)
SELECT f.file_number, f.type, f.bytes/1024/1024 AS mb,
       COUNT(*) AS extents
FROM   v$asm_file f JOIN v$asm_alias a ON a.file_number = f.file_number
JOIN   v$asm_diskgroup dg ON dg.group_number = f.group_number
WHERE  dg.name = 'DATA'
GROUP  BY f.file_number, f.type, f.bytes
ORDER  BY mb DESC
FETCH FIRST 10 ROWS ONLY;
```

## Best Practices

1. **1 MB or 4 MB** for typical production.
2. Exadata: **4 MB** (default).
3. Very large data warehouses: consider **16 MB**.
4. Match AU across similar diskgroups for consistency.
5. Set at diskgroup creation — can't change later.
6. Don't over-engineer; measure before deviating from defaults.

## Interview Questions

1. **Q:** What is an AU?
   **A:** Allocation Unit — smallest ASM allocation. Default 1 MB.

2. **Q:** How to change?
   **A:** Set at diskgroup creation via `au_size` attribute. Cannot change later.

3. **Q:** Larger AU pros?
   **A:** Less metadata, larger sequential I/O. Cons: wasteful for many small files.

4. **Q:** Exadata default?
   **A:** 4 MB.

## References

- Oracle ASM Administrator's Guide 19c
- MOS Doc ID 1523046.1 — ASM Best Practices
- MOS Doc ID 470435.1 — AU Size Impact
