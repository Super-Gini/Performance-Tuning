# GV$PROCESS

## Purpose

Cluster-wide `V$PROCESS` — one row per Oracle process per instance.

## Extra Column

| Column    | Meaning          |
| --------- | ---------------- |
| `INST_ID` | Instance number. |

## Common Queries

```sql
-- Top PGA consumers cluster-wide
SELECT inst_id, spid, sid, username,
       ROUND(pga_used_mem/1024/1024,2) used_mb,
       ROUND(pga_max_mem/1024/1024,2) max_mb
FROM   gv$process p JOIN gv$session s ON s.paddr = p.addr AND s.inst_id = p.inst_id
WHERE  s.type = 'USER'
ORDER  BY p.pga_used_mem DESC
FETCH  FIRST 20 ROWS ONLY;

-- Background processes on each node
SELECT inst_id, pname, spid FROM gv$process
WHERE  pname IS NOT NULL
ORDER  BY inst_id, pname;
```

## References

- Oracle Database Reference 19c — `GV$PROCESS`
