# Underscore (Hidden) Parameters

## Overview

**Underscore parameters** are undocumented Oracle init parameters — their names begin with `_`. They exist for internal use, bug workarounds, and features Oracle doesn't want end-users touching. Setting one without a MOS document backing you can cause silent corruption, poor performance, or unsupportability.

**Rule of thumb**: set an underscore parameter only when:

1. MOS Doc explicitly recommends it, OR
2. Oracle Support in your SR explicitly requests it, AND
3. You've tested in a lower env first, AND
4. You document why in your CMDB / runbook.

## How Many Are There?

19c has 3,500+ underscore parameters. They fall into categories:

- **Bug fixes** — `_fix_control` flags for specific bug numbers.
- **Feature toggles** — `_use_single_log_writer`, `_optimizer_adaptive_features`.
- **Sizing overrides** — `_shared_pool_reserved_min_alloc`.
- **Diagnostic** — `_disable_flashback_archiver`.

## Viewing All Underscore Parameters

Requires access to `X$` tables — typically `SYSDBA`:

```sql
COLUMN name    FORMAT A45
COLUMN value   FORMAT A30
COLUMN dflt    FORMAT A5
COLUMN modified FORMAT A15

SELECT   x.ksppinm  AS name,
         y.ksppstvl AS value,
         DECODE(BITAND(x.ksppiflg/256,1),1,'TRUE','FALSE') AS session_modifiable,
         DECODE(BITAND(x.ksppiflg/65536,3),1,'IMMED',2,'DEFER','FALSE') AS system_modifiable,
         y.ksppstdf AS dflt,
         y.ksppstdvl AS default_value,
         x.ksppdesc AS description
FROM     x$ksppi  x,
         x$ksppcv y
WHERE    x.indx    = y.indx
   AND   x.ksppinm LIKE '\_%' ESCAPE '\'
ORDER BY x.ksppinm;
```

## Non-Default Underscore Parameters

Show only ones changed from default (much more useful):

```sql
SELECT   x.ksppinm  AS name,
         y.ksppstvl AS value,
         y.ksppstdvl AS default_value,
         x.ksppdesc AS description
FROM     x$ksppi  x, x$ksppcv y
WHERE    x.indx = y.indx
   AND   x.ksppinm LIKE '\_%' ESCAPE '\'
   AND   y.ksppstdf = 'FALSE'          -- not at default
ORDER BY x.ksppinm;
```

Sites often have 10–30 non-default underscore params, some legitimate, some legacy cruft. Audit periodically.

## Common Examples

| Parameter                                   | Purpose                                                          |
| ------------------------------------------- | ---------------------------------------------------------------- |
| `_optimizer_adaptive_statistics`            | Formerly `optimizer_adaptive_features`. 19c splits this feature. |
| `_optimizer_use_feedback`                   | Cardinality feedback. Often set FALSE to avoid plan instability. |
| `_use_single_log_writer`                    | 12.2+ single-LGWR path. TRUE default in 19c.                     |
| `_disable_directory_link_check`             | Bypass directory sanity check.                                   |
| `_kgl_hot_object_copies`                    | Reduce KGL contention on hot cursors.                            |
| `_fix_control='<bugno>:0'`                  | Disable a specific optimizer fix.                                |
| `_datafile_write_errors_crash_instance`     | Crash on write failure vs offline datafile.                      |
| `_gc_defer_time`                            | RAC gc delay tuning (11g bug workaround).                        |
| `_high_priority_processes`                  | List of processes to elevate OS priority.                        |
| `_smon_undo_recovery_secs_before_lag_check` | SMON undo recovery pacing.                                       |

## Setting an Underscore Parameter

Quote the name because of the leading underscore:

```sql
-- One-time to test
ALTER SESSION SET "_optimizer_use_feedback" = FALSE;

-- Persist system-wide
ALTER SYSTEM SET "_optimizer_use_feedback" = FALSE SCOPE=BOTH;
```

## Removing / Resetting

```sql
ALTER SYSTEM RESET "_optimizer_use_feedback" SCOPE=BOTH;
```

Some require bounce (`SCOPE=SPFILE`); many are dynamic (`SCOPE=BOTH`).

## fix_control — Disabling Optimizer Fixes

Special mechanism to toggle individual optimizer fixes by bug number:

```sql
-- What fixes are enabled?
SELECT bugno, description, value, sql_feature
FROM   v$system_fix_control
WHERE  is_default = 'FALSE'
ORDER  BY bugno;

-- Disable a specific fix system-wide
ALTER SYSTEM SET "_fix_control" = '32521906:0' SCOPE=BOTH;

-- Enable
ALTER SYSTEM SET "_fix_control" = '32521906:1' SCOPE=BOTH;
```

Use only when MOS or Oracle Support tells you a fix is causing regression.

## Danger List — Do NOT Touch Without Support

- `_use_realfree_heap`
- `_kks_use_mutex_pin`
- `_kgl_bucket_count`
- `_undo_autotune`
- `_serial_direct_read`
- `_optimizer_gather_stats_on_load`

These affect fundamental engine behavior.

## Auditing Your Site

Save monthly snapshots:

```sql
CREATE TABLE dba_history.underscore_params AS
SELECT SYSDATE snap_time, x.ksppinm name, y.ksppstvl value
FROM   x$ksppi x, x$ksppcv y
WHERE  x.indx = y.indx
   AND x.ksppinm LIKE '\_%' ESCAPE '\'
   AND y.ksppstdf = 'FALSE';
```

Diff month-over-month; any change should have a ticket.

## References

- MOS Doc ID 138872.1 — Underscore parameter policy
- MOS Doc ID 1683791.1 — Common tuning params (documented + underscore)
- MOS Doc ID 39627.1 — `X$KSPPI` structure
