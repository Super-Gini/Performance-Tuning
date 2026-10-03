# GV$SQL

## Purpose

Cluster-wide `V$SQL` — the same SQL cursor may be loaded on multiple RAC instances. Aggregate on `SQL_ID` to see total workload.

## Extra Column

| Column    | Meaning          |
| --------- | ---------------- |
| `INST_ID` | Instance number. |

## Common Queries

```sql
-- Top SQL cluster-wide by elapsed time
SELECT   sql_id,
         SUM(executions) execs,
         ROUND(SUM(elapsed_time)/1e6,2) elapsed_secs,
         ROUND(SUM(elapsed_time)/GREATEST(SUM(executions),1)/1e6,3) sec_per_exec,
         ROUND(SUM(cpu_time)/1e6,2) cpu_secs
FROM     gv$sql
GROUP BY sql_id
ORDER BY elapsed_secs DESC
FETCH FIRST 10 ROWS ONLY;

-- Where is this SQL parsed
SELECT inst_id, sql_id, child_number, plan_hash_value, executions
FROM   gv$sql
WHERE  sql_id = '&sql_id'
ORDER  BY inst_id, child_number;

-- SQL running only on some instances
SELECT sql_id, COUNT(DISTINCT inst_id) instances, SUM(executions) execs
FROM   gv$sql
GROUP  BY sql_id
HAVING COUNT(DISTINCT inst_id) < (SELECT COUNT(*) FROM gv$instance)
ORDER  BY 2, 3 DESC;
```

## References

- Oracle Database Reference 19c — `GV$SQL`
- [SQL Monitor](../../12-performance-tuning/sql-monitor.md)
