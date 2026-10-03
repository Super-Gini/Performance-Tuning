# Multitenant

The **Multitenant** architecture (12.1+) restructures the Oracle database into a **container database (CDB)** hosting multiple **pluggable databases (PDBs)**. A CDB has one SGA, one set of background processes, and one shared SYSTEM/SYSAUX/UNDO/TEMP; each PDB has its own dictionary, its own tablespaces, and its own users. PDBs plug into a CDB, unplug from one, plug into another — like Docker containers for databases.

In 19c EE, up to **3 free PDBs per CDB** without the Multitenant Option license. Beyond 3 (up to 4096) requires the Multitenant Option.

## Contents

| Page                                                | Purpose                                                   |
| --------------------------------------------------- | --------------------------------------------------------- |
| [CDB](cdb.md)                                       | Container database — architecture, `CDB$ROOT`, `PDB$SEED` |
| [PDB](pdb.md)                                       | Pluggable database — lifecycle, open modes, save state    |
| [Application Containers](application-containers.md) | Application root and application PDBs (12.2+)             |
| [Application PDBs](application-pdbs.md)             | Sharing definitions across sibling PDBs                   |
| [Common Users](common-users.md)                     | `C##` users, roles, privileges spanning all containers    |
| [Local Users](local-users.md)                       | Per-PDB users; the default and recommended                |
| [PDB Clone](pdb-clone.md)                           | Hot clone, cold clone, remote clone                       |
| [PDB Snapshot Clone](pdb-snapshot-clone.md)         | ACFS / ZFS snapshot copy-on-write clones                  |
| [Refreshable PDB](refreshable-pdb.md)               | Periodic sync from source PDB                             |
| [PDB Relocate](pdb-relocate.md)                     | Move a PDB to another CDB with minimal downtime           |
| [Unplug / Plug](unplug-plug.md)                     | Physical move by unplugging and plugging                  |

## Related

- [User Management](../09-user-management/index.md)
- [RMAN](../15-rman/index.md) — backup/restore of CDB and PDBs.
- [Data Guard](../17-data-guard/index.md) — DG at CDB level.
