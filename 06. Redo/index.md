# Redo

Redo is Oracle's durability engine. Every change to the database (INSERT, UPDATE, DELETE, DDL) generates **change vectors** that describe the change; those vectors are buffered in the redo log buffer and flushed by LGWR to online redo logs. On commit, LGWR waits for the flush to succeed — only then does the commit acknowledge to the client. This synchronous write is the mechanism that guarantees committed transactions survive a crash.

## Contents

| Page                                      | Purpose                                                                                   |
| ----------------------------------------- | ----------------------------------------------------------------------------------------- |
| [Redo Architecture](redo-architecture.md) | End-to-end redo flow: change vector → log buffer → LGWR → online log → ARCn → archive log |
| [Commit Processing](commit-processing.md) | The commit path, `log file sync`, group commits, `commit_write` options                   |
| [Checkpoints](checkpoints.md)             | Full, incremental, and thread checkpoints; `fast_start_mttr_target`                       |
| [Log Switches](log-switches.md)           | When switches happen, hardware/config impact                                              |
| [Redo Tuning](redo-tuning.md)             | Sizing, multiplexing, scalable LGWR, storage choice, DG mode impact                       |

## Related

- [LGWR](../03-instance-architecture/processes/lgwr.md) — the redo writer process.
- [ARCn](../03-instance-architecture/processes/arcn.md) — archiver.
- [CKPT](../03-instance-architecture/processes/ckpt.md) — checkpoint.
- [Data Guard](../17-data-guard/index.md) — how redo ships to standby.
- [RMAN](../15-rman/index.md) — media recovery uses redo.
