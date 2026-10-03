# Real-World Case Studies

Composite investigations drawn from real DBA incidents. Each follows a **symptom → hypothesis → evidence → root cause → fix → lessons learned** structure. Detail levels vary; the goal is to show the reasoning, not just the outcome.

## Contents

| Page                                                        | Story                                      |
| ----------------------------------------------------------- | ------------------------------------------ |
| [AWR Analysis](awr-analysis.md)                             | Reading an AWR report end-to-end           |
| [ASH Analysis](ash-analysis.md)                             | Diagnosing a spike using ASH samples       |
| [Blocking Session Analysis](blocking-session-analysis.md)   | 15-min hang across app tier                |
| [Data Guard Lag](data-guard-lag.md)                         | Standby fell 30 minutes behind             |
| [DMS Migration Issues](dms-migration-issues.md)             | AWS DMS silent data mismatch               |
| [EXPDP Performance](expdp-performance.md)                   | 12-hour dump made 3-hour                   |
| [ORA-600 Investigation](ora-600-investigation.md)           | Recurring internal error walked to bug fix |
| [RAC Node Failure](rac-node-failure.md)                     | 3 AM node eviction post-mortem             |
| [RMAN Recovery](rman-recovery.md)                           | Restore across incarnations                |
| [Tablespace Growth Analysis](tablespace-growth-analysis.md) | Unplanned 400 GB/month growth              |

## How to Read These

- Not a substitute for [Runbooks](../27-runbooks/index.md).
- The **evidence** sections are the practice — try to guess the root cause before reading the answer.
- Real numbers used; your environment will differ.

## Related

- [Runbooks](../27-runbooks/index.md).
- [Interview Questions](../31-interview-questions/index.md).
- [Errors](../26-errors/index.md).
