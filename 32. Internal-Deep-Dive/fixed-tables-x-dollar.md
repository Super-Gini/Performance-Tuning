# Fixed Tables (X$)

## Overview

**X$ tables** are Oracle's internal, SGA-mapped data structures exposed as read-only, SQL-queryable rows. They're the raw metadata layer beneath every `V$` and `GV$` view. There are ~1300 X$ tables in 19c. Every non-trivial diagnostic query eventually joins to one of them — often through a V$ synonym, sometimes directly.

This page maps the naming convention, exposes the V$ ↔ X$ relationship, and catalogs the ones a DBA reaches for on hard problems.

## Naming Convention

X$ names follow a **layered abbreviation** convention:

- `X$K` — Kernel-layer prefix (most tables start here).
- `X$KC*` — Kernel Cache.
- `X$KCB*` — Kernel Cache Buffer (buffer cache).
- `X$KG*` — Kernel Generic.
- `X$KGL*` — KGL (Kernel Generic Library — library cache).
- `X$KGH*` — KGH (Kernel Generic Heap — shared pool heap).
- `X$KSM*` — KSM (Kernel Service Memory).
- `X$KKS*` — KKS (Kernel Kompiler SQL — cursor layer).
- `X$KJ*` — KJ (Kernel J — RAC).
- `X$KT*` — KT (Kernel Transaction).
- `X$KTU*` — KTU (Kernel Transaction Undo).
- `X$KCR*` — KCR (Kernel Cache Redo).

You can guess a table's role from the prefix.

## V$ Views Are Built on X$

`V$SESSION` is a **synonym** for `V_$SESSION` which is a **view** over `X$KSUSE`. Show the actual X$ underneath any V$:

```sql
SELECT view_definition
FROM   v$fixed_view_definition
WHERE  view_name = 'V$SESSION';
```

Result reveals column mappings:

```sql
select ksusenum, ksuseser, ksuudlui, ...
from x$ksuse
where ...
```

## Access

X$ access requires `SELECT ANY DICTIONARY` or `SYSDBA`. Some are further restricted to SYS only. Regular users can't query X$.

## The Most-Used X$ Tables — By Domain

### Sessions & Processes

| X$            | V$ mapping    | Purpose                     |
| ------------- | ------------- | --------------------------- |
| `X$KSUSE`     | `V$SESSION`   | User sessions.              |
| `X$KSUPR`     | `V$PROCESS`   | OS processes.               |
| `X$KSUPL`     | `V$PARAMETER` | Init parameters (via `V$`). |
| `X$KSUPGP`    | `V$PROCESS`   | Process group info.         |
| `X$KSUXSINST` | `V$INSTANCE`  | Instance info.              |
| `X$KSUMYSTA`  | `V$MYSTAT`    | Current session stats.      |

### Wait Events

| X$         | V$ mapping             | Purpose                      |
| ---------- | ---------------------- | ---------------------------- |
| `X$KSLED`  | `V$EVENT_NAME`         | All event names and IDs.     |
| `X$KSLEI`  | `V$SYSTEM_EVENT`       | System-wide event counters.  |
| `X$KSLES`  | `V$SESSION_EVENT`      | Per-session event counters.  |
| `X$KSLLW`  | `V$SESSION_WAIT`       | Current waits.               |
| `X$KSLWSC` | `V$SESSION_WAIT_CLASS` | Per-session per-class stats. |

### Buffer Cache

| X$           | V$ mapping                 | Purpose                   |
| ------------ | -------------------------- | ------------------------- |
| `X$BH`       | `V$BH`                     | Buffer header per buffer. |
| `X$KCBWDS`   | `V$BUFFER_POOL_STATISTICS` | Buffer pool stats.        |
| `X$KCBWH`    | `V$WAITSTAT`               | Buffer wait breakdown.    |
| `X$KCBFWAIT` | `V$FILE_HISTOGRAM`         | Per-file IO histogram.    |

### Latches & Mutexes

| X$         | V$ mapping              | Purpose               |
| ---------- | ----------------------- | --------------------- |
| `X$KSLLT`  | `V$LATCH`               | Latch parents.        |
| `X$KSLLTR` | `V$LATCH_CHILDREN`      | Latch children.       |
| `X$KSLLD`  | -                       | Latch names + levels. |
| `X$KSLM`   | `V$MUTEX_SLEEP`         | Mutex sleep stats.    |
| `X$KSLMH`  | `V$MUTEX_SLEEP_HISTORY` | Recent mutex sleeps.  |

### Library Cache / KGL

| X$           | V$ mapping                | Purpose                                |
| ------------ | ------------------------- | -------------------------------------- |
| `X$KGLOB`    | `V$DB_OBJECT_CACHE`       | KGL objects (cursors, packages, etc.). |
| `X$KGLST`    | `V$LIBRARYCACHE`          | Namespace stats.                       |
| `X$KGLLK`    | `V$OPEN_CURSOR` (partial) | KGL locks held.                        |
| `X$KGLPN`    | -                         | KGL pins.                              |
| `X$KGLDP`    | -                         | Dependencies.                          |
| `X$KGLNA`    | -                         | Names of KGL objects (join to KGLOB).  |
| `X$KGLBS`    | -                         | KGL bind values.                       |
| `X$KGLTABLE` | `V$SQLTEXT`               | SQL text.                              |

### Shared Pool Heap

| X$        | V$ mapping  | Purpose                                |
| --------- | ----------- | -------------------------------------- |
| `X$KGHLU` | `V$SGAINFO` | Sub-heap usage.                        |
| `X$KSMSP` | -           | Chunk-by-chunk detail (very detailed). |
| `X$KGHSC` | -           | Sub-heap descriptors.                  |
| `X$KGLRD` | -           | Reload counts.                         |

### Redo & Undo

| X$         | V$ mapping      | Purpose                   |
| ---------- | --------------- | ------------------------- |
| `X$KCRFWS` | `V$LOG`         | Online redo log groups.   |
| `X$KCRRR`  | -               | Redo record types.        |
| `X$KTUXE`  | `V$TRANSACTION` | Active transactions.      |
| `X$KTIFP`  | -               | Transaction IDs.          |
| `X$KTURD`  | -               | Undo Retention Data (RD). |

### RAC / GES / GCS

| X$          | V$ mapping         | Purpose                |
| ----------- | ------------------ | ---------------------- |
| `X$KJICVT`  | `V$GC_ELEMENT`     | Global cache elements. |
| `X$KJIRFT`  | `V$RESOURCE`       | Cluster resources.     |
| `X$KJILKFT` | -                  | Cluster lock info.     |
| `X$KJMDDP`  | `V$GES_STATISTICS` | GES stats.             |

### SGA

| X$         | V$ mapping  | Purpose             |
| ---------- | ----------- | ------------------- |
| `X$KSMSTA` | `V$SGASTAT` | SGA stats by pool.  |
| `X$KSMFSV` | -           | Fixed SGA vars.     |
| `X$KSMSSF` | -           | SGA sub-heap stats. |

### Init Parameters

| X$         | V$ mapping    | Purpose                         |
| ---------- | ------------- | ------------------------------- |
| `X$KSPPI`  | `V$PARAMETER` | Parameter names.                |
| `X$KSPPCV` | -             | Current values.                 |
| `X$KSPPSV` | -             | Startup values (SPFILE values). |

### Data Dictionary Cache

| X$        | V$ mapping                | Purpose               |
| --------- | ------------------------- | --------------------- |
| `X$KQRST` | `V$ROWCACHE`              | Row cache stats.      |
| `X$KQFTA` | `V$FIXED_TABLE`           | List of fixed tables. |
| `X$KQFVI` | `V$FIXED_VIEW_DEFINITION` | View definitions.     |

## Complete Table Listing

```sql
-- Every X$ (v_$fixed_table shows all)
SELECT   name, table_num, obj#
FROM     v$fixed_table
WHERE    name LIKE 'X$%'
ORDER BY name;

-- Column definitions of a specific X$
DESC X$KSUSE
```

Or:

```sql
SELECT col_name, col_type, col_size
FROM   v$indexed_fixed_column
WHERE  table_name = 'X$KSUSE';
```

## Building Your Own Views

Sometimes V$ is missing a column. Query X$ directly and expose it:

```sql
CREATE VIEW dba_priv.session_full_state AS
SELECT s.sid, s.serial#, s.username,
       KSUSEUD user_id,
       KSUSESQI last_call_et,
       KSUSEPRI priority,
       KSUSEQTM last_sql_time
FROM   v$session s JOIN x$ksuse x ON x.ADDR = s.saddr;
```

## Pitfalls

- **X$ tables can change between versions.** Column names/positions aren't stable. Don't rely on them in production monitoring code without version-check.
- **Some X$ have huge row counts.** `X$KGLOB` on a big shared pool might be millions. Filter aggressively.
- **X$ query costs vary.** Some are memory scans (fast); some require latches (can add contention).
- **`X$KSMSP`** — expensive; iterating every shared pool chunk. Use sparingly.
- **`X$BH`** — heavy on big buffer caches.

## Diagnostic Recipes

### List X$ tables in an interesting namespace

```sql
SELECT name, kqftaobj obj_id
FROM   x$kqfta
WHERE  name LIKE 'X$KGL%'
ORDER  BY name;
```

### Column mapping for a V$ view

```sql
SELECT view_definition
FROM   v$fixed_view_definition
WHERE  view_name = 'V$SQL_SHARED_CURSOR';
```

### Explore X$KSUSE columns

```sql
SELECT   colnam, TYP colType, sz
FROM     x$kqfco WHERE table_number =
         (SELECT table_number FROM x$kqfta WHERE name = 'X$KSUSE')
ORDER BY position;
```

## Interview Framing

> "What are X$ tables?"

Read-only, SQL-queryable views over Oracle's internal SGA structures. Every V$ view is defined by joining and filtering X$ tables. Restricted to SYSDBA / SELECT ANY DICTIONARY.

> "Why would you query X$ directly?"

When V$ doesn't expose the column you need, when you want raw structure without V$'s aggregation, or when investigating a bug for MOS SR. Also: some X$ tables have no V$ equivalent (e.g., `X$KGHLU`).

## References

- MOS Doc ID 175982.1 — X$ Tables Reference.
- Julian Dyke's classic writeups (Google "julian dyke x$" for historical detail).

## Related

- [V$SESSION](../25-reference/v-views/v-session.md), [V$SQL](../25-reference/v-views/v-sql.md), etc.
- [Buffer Cache Internals](buffer-cache-internals.md).
- [Library Cache](library-cache.md).
- [KGH & Shared Pool Heap](kgh-shared-pool-heap.md).
- [Latches vs Mutexes](latches-vs-mutexes.md).
