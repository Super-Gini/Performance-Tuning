# Oracle Errors (ORA-)

Common Oracle errors — meaning, causes, quick fix, prevention. Every entry follows the same format so you can grep and act.

## Application/Data Errors

| Code      | Message                    | Page                      |
| --------- | -------------------------- | ------------------------- |
| ORA-00001 | Unique constraint violated | [ora-00001](ora-00001.md) |
| ORA-00054 | Resource busy              | [ora-00054](ora-00054.md) |
| ORA-00060 | Deadlock detected          | [ora-00060](ora-00060.md) |
| ORA-01017 | Invalid username/password  | [ora-01017](ora-01017.md) |
| ORA-01031 | Insufficient privileges    | [ora-01031](ora-01031.md) |

## Session/Instance Errors

| Code      | Message                                 | Page                      |
| --------- | --------------------------------------- | ------------------------- |
| ORA-00020 | Maximum number of processes exceeded    | [ora-00020](ora-00020.md) |
| ORA-00205 | Error identifying control file          | [ora-00205](ora-00205.md) |
| ORA-00257 | Archiver error (FRA full)               | [ora-00257](ora-00257.md) |
| ORA-01157 | Cannot identify/lock datafile           | [ora-01157](ora-01157.md) |
| ORA-01219 | Database or pluggable database not open | [ora-01219](ora-01219.md) |

## Space Errors

| Code      | Message                                  | Page                      |
| --------- | ---------------------------------------- | ------------------------- |
| ORA-01652 | Unable to extend TEMP segment            | [ora-01652](ora-01652.md) |
| ORA-01653 | Unable to extend table                   | [ora-01653](ora-01653.md) |
| ORA-01654 | Unable to extend index                   | [ora-01654](ora-01654.md) |
| ORA-04031 | Unable to allocate memory in shared pool | [ora-04031](ora-04031.md) |

## Undo / Consistent Read

| Code      | Message          | Page                      |
| --------- | ---------------- | ------------------------- |
| ORA-01555 | Snapshot too old | [ora-01555](ora-01555.md) |

## Internal Errors

| Code      | Message                              | Page                      |
| --------- | ------------------------------------ | ------------------------- |
| ORA-00600 | Internal error code (bug)            | [ora-00600](ora-00600.md) |
| ORA-07445 | Exception encountered (segfault)     | [ora-07445](ora-07445.md) |
| ORA-03113 | End-of-file on communication channel | [ora-03113](ora-03113.md) |
| ORA-03135 | Connection lost contact              | [ora-03135](ora-03135.md) |

## Network Errors

| Code      | Message                                                     | Page                      |
| --------- | ----------------------------------------------------------- | ------------------------- |
| ORA-12154 | TNS: could not resolve the connect identifier               | [ora-12154](ora-12154.md) |
| ORA-12514 | Listener does not currently know of service                 | [ora-12514](ora-12514.md) |
| ORA-12528 | All appropriate instances are blocking new connections      | [ora-12528](ora-12528.md) |
| ORA-12541 | TNS: no listener                                            | [ora-12541](ora-12541.md) |
| ORA-12545 | Connect failed because target host or object does not exist | [ora-12545](ora-12545.md) |

## Related

- [Alert Log](../24-monitoring/alert-log.md) — where errors surface.
- [Trace Files](../24-monitoring/trace-files.md) — for ORA-00600/07445.
- [Runbooks](../27-runbooks/index.md) — response procedures.
