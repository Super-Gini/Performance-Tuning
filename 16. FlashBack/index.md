# Flashback

Oracle's **Flashback** family lets you rewind data — at the query, transaction, table, or database level — without restoring from backup. Different Flashback features use different underlying stores:

- **Flashback Query / Version Query** — reads undo (bounded by `UNDO_RETENTION`).
- **Flashback Table** — reads undo, moves rows back.
- **Flashback Drop / Recycle Bin** — drop metadata retention.
- **Flashback Data Archive (FDA)** — dedicated tablespace, long retention.
- **Flashback Database** — flashback logs in the FRA, whole-DB rewind.

Together they cover accidental change scenarios that once required RMAN restore.

## Contents

| Page                                        | Purpose                            |
| ------------------------------------------- | ---------------------------------- |
| [Flashback Query](flashback-query.md)       | `AS OF TIMESTAMP` / `AS OF SCN`    |
| [Flashback Table](flashback-table.md)       | `FLASHBACK TABLE ... TO TIMESTAMP` |
| [Flashback Database](flashback-database.md) | Whole-DB rewind                    |
| [Recycle Bin](recycle-bin.md)               | Dropped table retention            |

## Related

- [Undo Management](../05-undo/undo-management.md) — powers query-level flashback.
- [FBDA process](../03-instance-architecture/processes/fbda.md) — for Flashback Data Archive.
- [PITR](../15-rman/pitr.md) — heavier alternative when flashback isn't enabled.
