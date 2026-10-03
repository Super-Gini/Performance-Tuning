# Installation

Everything from bare-metal prerequisites to applying quarterly patches. This section is organized as the natural flow a DBA follows when standing up a new 19c database on a Linux server.

## Contents

| Page                                                        | Purpose                                                            |
| ----------------------------------------------------------- | ------------------------------------------------------------------ |
| [Installation Prerequisites](installation-prerequisites.md) | OS packages, kernel parameters, user/group setup, storage layout   |
| [Oracle Home](oracle-home.md)                               | What `$ORACLE_HOME` is, inventory structure, gold-image strategy   |
| [GUI Installation](gui-installation.md)                     | Running `runInstaller` interactively — when it makes sense         |
| [Silent Installation](silent-installation.md)               | Non-interactive install for automation                             |
| [Response Files](response-files.md)                         | Response file structure, essential parameters                      |
| [DBCA](dbca.md)                                             | Database Configuration Assistant — creating and dropping databases |
| [OPatch](opatch.md)                                         | The patching engine — `lsinventory`, `apply`, `rollback`           |
| [Datapatch](datapatch.md)                                   | SQL-level component of a patch — required after every RU           |

---

## Recommended Reading Order

1. [Installation Prerequisites](installation-prerequisites.md) — start here.
2. [Oracle Home](oracle-home.md) — understand the target directory structure.
3. [Silent Installation](silent-installation.md) — the production-grade way to install.
4. [Response Files](response-files.md) — inputs to silent install.
5. [DBCA](dbca.md) — create the actual database.
6. [OPatch](opatch.md) → [Datapatch](datapatch.md) — apply the current RU.

## Related Sections

- [Patching](../22-patching/index.md) — quarterly RU workflow, OPatchauto for RAC/GI.
- [Upgrade & Migration](../23-upgrade-migration/index.md) — Autoupgrade, DBUA, manual upgrade.
