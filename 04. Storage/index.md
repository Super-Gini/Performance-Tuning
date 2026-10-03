# Storage

Everything about how Oracle Database organizes data on disk: control files, datafiles, tablespaces, segments, extents, blocks, and the row-level structures within blocks. This section explains **what lives where**, **why it matters**, and **how to diagnose** the space, corruption, and efficiency problems DBAs deal with day to day.

## Contents

### Physical Files

| Page                              | Purpose                              |
| --------------------------------- | ------------------------------------ |
| [Control Files](control-files.md) | The tiny binary map of the database  |
| [Datafiles](datafiles.md)         | Persistent storage for tablespaces   |
| [Redo Logs](redo-logs.md)         | Change vector journal                |
| [Archive Logs](archive-logs.md)   | Persisted historical redo            |
| [Tempfiles](tempfiles.md)         | Backing storage for TEMP tablespaces |

### Logical Storage Hierarchy

| Page                                                    | Purpose                                              |
| ------------------------------------------------------- | ---------------------------------------------------- |
| [Tablespaces](tablespaces.md)                           | SYSTEM, SYSAUX, UNDO, TEMP, USERS, custom            |
| [Bigfile Tablespaces](bigfile-tablespaces.md)           | Single-datafile tablespaces for very large databases |
| [Segments](segments.md)                                 | Table, index, undo, LOB, temporary                   |
| [Extents](extents.md)                                   | Contiguous groups of blocks                          |
| [Segment Space Management](segment-space-management.md) | ASSM vs MSSM, PCTFREE, PCTUSED                       |
| [High Water Mark](high-water-mark.md)                   | HWM, low HWM, deferred segment creation              |

### Block and Row Structure

| Page                                  | Purpose                                          |
| ------------------------------------- | ------------------------------------------------ |
| [Block Structure](block-structure.md) | Header, ITL, row directory, row data, free space |
| [Row Storage](row-storage.md)         | Row piece layout, column storage                 |
| [Row Chaining](row-chaining.md)       | When a row spans multiple blocks                 |
| [Row Migration](row-migration.md)     | When a row moves to a new block                  |
| [LOB Storage](lob-storage.md)         | BasicFiles vs SecureFiles, inline vs out-of-line |

## Related Sections

- [ASM](../19-asm/index.md) — Automatic Storage Management as the physical layer.
- [Undo Management](../05-undo/index.md) — the UNDO tablespace and read consistency.
- [Redo](../06-redo/index.md) — how change vectors flow to redo logs.
