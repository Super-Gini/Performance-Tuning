# Runbook: Tablespace Full

## Symptom

- ORA-01653 / ORA-01654 in alert log.
- Application inserts failing.
- Monitoring: tablespace > 95%.

## Triage

```sql
SELECT   df.tablespace_name,
         ROUND(SUM(df.bytes)/1024/1024/1024, 2) alloc_gb,
         ROUND((SUM(df.bytes)-NVL(SUM(fs.bytes),0))/1024/1024/1024, 2) used_gb,
         ROUND((SUM(df.bytes)-NVL(SUM(fs.bytes),0))*100/SUM(df.bytes), 2) pct_used,
         ROUND(SUM(df.maxbytes)/1024/1024/1024, 2) max_gb,
         MAX(df.autoextensible) autoext
FROM     dba_data_files df
LEFT JOIN (SELECT tablespace_name, SUM(bytes) bytes FROM dba_free_space
           GROUP BY tablespace_name) fs USING (tablespace_name)
GROUP BY df.tablespace_name
HAVING   ROUND((SUM(df.bytes)-NVL(SUM(fs.bytes),0))*100/SUM(df.bytes), 2) > 85
ORDER BY 4 DESC;
```

Filesystem:

```bash
df -h $ORACLE_BASE
```

## Actions (in order of preference)

### 1. Autoextend datafile

```sql
ALTER DATABASE DATAFILE '/u01/oradata/PRD/users01.dbf' AUTOEXTEND ON MAXSIZE 100G;
```

### 2. Extend existing datafile

```sql
ALTER DATABASE DATAFILE '/u01/oradata/PRD/users01.dbf' RESIZE 50G;
```

### 3. Add new datafile

```sql
ALTER TABLESPACE USERS ADD DATAFILE '/u01/oradata/PRD/users02.dbf'
    SIZE 20G AUTOEXTEND ON MAXSIZE 100G;
```

### 4. If filesystem is out of space too

- Move backups/archives off the mount.
- Add storage.
- Emergency: `DELETE ARCHIVELOG UNTIL TIME 'SYSDATE-3'` (backup first!).

### 5. Bigfile conversion (if hitting 32 GB smallfile limit)

```sql
CREATE BIGFILE TABLESPACE USERS_BIG DATAFILE '/u01/oradata/PRD/users_big.dbf'
    SIZE 100G AUTOEXTEND ON;

ALTER TABLE app.orders MOVE ONLINE TABLESPACE USERS_BIG;
ALTER INDEX app.pk_orders REBUILD ONLINE TABLESPACE USERS_BIG;

-- Once empty
DROP TABLESPACE USERS INCLUDING CONTENTS AND DATAFILES;
```

### 6. Find and reclaim wasted space

```sql
-- Segment advisor
BEGIN
  DBMS_ADVISOR.QUICK_TUNE('Segment Advisor', 'my_seg_tune',
    'select_object => ''SEGMENT'', object_type => ''TABLE'',
     schema => ''APP''');
END;
/

-- HWM check
SELECT owner, segment_name, blocks*8192/1024/1024 mb,
       (SELECT COUNT(*) FROM app.orders) rows
FROM   dba_segments WHERE segment_name='ORDERS';
```

Shrink:

```sql
ALTER TABLE app.orders ENABLE ROW MOVEMENT;
ALTER TABLE app.orders SHRINK SPACE CASCADE;
```

## Verification

```sql
SELECT tablespace_name, ROUND((used_percent),2) pct
FROM   dba_tablespace_usage_metrics
ORDER  BY 2 DESC;
```

Application: retry the failing INSERT.

## Post-Mortem

- Why did it fill?
- Autoextend was disabled — why?
- MAXBYTES too low — should be 32 TB (bigfile) or per-file limit.
- Monitoring didn't page early? Adjust thresholds (80/90).
- Growth rate — is the load unusual, or is 6-month runway exhausted?

## Prevention

- Autoextend + sensible MAXBYTES on every datafile.
- Alert at 80% and 90%.
- Bigfile tablespaces for anything > 32 GB per file.
- Growth forecasting (compare `dba_hist_tbspc_space_usage` snapshots).

## Related

- [ORA-01653](../26-errors/ora-01653.md), [ORA-01654](../26-errors/ora-01654.md).
- [Tablespaces](../04-storage/tablespaces.md), [Bigfile Tablespaces](../04-storage/bigfile-tablespaces.md).
