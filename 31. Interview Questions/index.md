# Interview Questions

Interview banks by level and topic. Q&A format — question first, then a two-to-three-sentence answer with the key concept and (where relevant) the follow-up interviewers dig into.

## Contents

| Page                                        | Level / Topic                                       |
| ------------------------------------------- | --------------------------------------------------- |
| [Beginner](beginner.md)                     | Fundamentals: architecture, backup, basic tuning    |
| [Intermediate](intermediate.md)             | Undo, redo, plans, ASH/AWR                          |
| [Advanced](advanced.md)                     | Internals, latches/mutexes, corruption, deep tuning |
| [ASM](asm.md)                               | ASM instance, diskgroups, rebalance                 |
| [RAC](rac.md)                               | Cache fusion, evictions, services                   |
| [Data Guard](data-guard.md)                 | Modes, roles, broker                                |
| [Performance Tuning](performance-tuning.md) | Wait events, SQL, memory, IO                        |

## Tips

- Speak in terms of **wait classes / views**, not vague words.
- Real numbers: buffer cache hit ratio 95% → talk about _cache misses per second_ and their wait event.
- Prefer the modern answer (Data Pump over exp, AutoUpgrade over DBUA).
- If asked "how would you do X" — always cover the safety net (backup, restore point).
- Admit "I'd check MOS Doc ID …" for ORA-00600 and other rare errors — better than guessing.

## Related

- [Real-World Case Studies](../34-real-world-case-studies/index.md) — practice the "walk me through" ones.
- [Runbooks](../27-runbooks/index.md) — the practical framing.
