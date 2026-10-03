# Oracle Home

## Overview

The **Oracle Home** (`$ORACLE_HOME`) is the directory containing the Oracle Database software binaries, libraries, configuration files, and the OPatch inventory for a single installed version + RU level. Every `oracle` executable, every `.so` library, and every default `network/admin` and `dbs` file lives under one Oracle Home. The concept is central to patching, upgrades, and running multiple database versions on one server.

An Oracle Home is:

- **Version-locked** — 19c binaries live in a 19c home; you never mix major versions in one home.
- **RU-locked (softly)** — Applying a Release Update advances _this_ home's binaries. Rolling back moves it back.
- **Owned by an OS user** — usually `oracle`, sometimes `grid` (for Grid Infrastructure).
- **Tracked by an inventory** — `oraInventory` (central) plus `$ORACLE_HOME/inventory` (local).

## Architecture

```mermaid
flowchart TB
    subgraph Server["Database Server"]
        subgraph GridHome["Grid Home (grid user)"]
            GRIDBIN[/u01/app/19.0.0/grid<br/>Grid Infrastructure + ASM]
        end
        subgraph DBHome1["Database Home #1 (oracle user)"]
            DB1BIN[/u01/app/oracle/product/19.0.0/dbhome_1<br/>19.20 RU]
            DB1DBS[$OH/dbs<br/>SPFILE, PFILE, PWD file]
            DB1NET[$OH/network/admin<br/>listener.ora, sqlnet.ora, tnsnames.ora]
        end
        subgraph DBHome2["Database Home #2 (oracle user)"]
            DB2BIN[/u01/app/oracle/product/19.0.0/dbhome_2<br/>19.21 RU — parallel for upgrades]
        end
        INV[/u01/app/oraInventory<br/>Central Inventory]
    end

    INV -.tracks.-> GRIDBIN
    INV -.tracks.-> DB1BIN
    INV -.tracks.-> DB2BIN
```

## Internal Working

### What lives in `$ORACLE_HOME`

| Subdirectory               | Contents                                                                                                                      |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `bin`                      | `sqlplus`, `rman`, `oracle` executable, `lsnrctl`, `dbca`, `netca`                                                            |
| `lib`                      | Shared libraries (`libclntsh.so`)                                                                                             |
| `dbs`                      | Server parameter file (`spfile<SID>.ora`), init file (`init<SID>.ora`), password file (`orapw<SID>`), external password store |
| `network/admin`            | `listener.ora`, `tnsnames.ora`, `sqlnet.ora`                                                                                  |
| `rdbms/admin`              | Catalog scripts, upgrade scripts (`catupgrd.sql`, `utlrp.sql`)                                                                |
| `sqlplus/admin/glogin.sql` | Global SQL\*Plus login script                                                                                                 |
| `OPatch`                   | The OPatch utility itself                                                                                                     |
| `.patch_storage`           | Prior binaries kept for rollback                                                                                              |
| `inventory`                | Local inventory for this home                                                                                                 |
| `oc4j`, `jdk`, `perl`      | Bundled runtimes                                                                                                              |

### `oraInventory` (Central Inventory)

Path is stored in `/etc/oraInst.loc`:

```
inventory_loc=/u01/app/oraInventory
inst_group=oinstall
```

The central inventory tracks **every Oracle Home on the server** — database homes, Grid Home, client homes, and older versions. Each home has a subdirectory under `oraInventory/ContentsXML/` with its GUID and location.

### Local Inventory (`$ORACLE_HOME/inventory`)

Tracks components installed _within_ this home and every applied patch. `opatch lsinventory -detail` reads from here.

### Gold Image Strategy

For consistency across many servers, install once, apply patches, then **create a gold image**:

```bash
$ORACLE_HOME/runInstaller -silent -createGoldImage \
    -destinationLocation /software/goldimages/db1920_19c.zip
```

On each target server:

```bash
unzip db1920_19c.zip -d $ORACLE_HOME
cd $ORACLE_HOME
./runInstaller -silent -responseFile /software/db1920.rsp
```

This is faster than repeatedly patching and eliminates drift.

## Components

| Component      | Purpose                                                          |
| -------------- | ---------------------------------------------------------------- |
| `$ORACLE_BASE` | Root of Oracle configuration (`/u01/app/oracle`)                 |
| `$ORACLE_HOME` | Software home (`$ORACLE_BASE/product/19.0.0/dbhome_1`)           |
| `$ORACLE_SID`  | Selects the current instance                                     |
| `$TNS_ADMIN`   | Overrides `network/admin` location (optional)                    |
| `oratab`       | `/etc/oratab` (or `/var/opt/oracle/oratab`) lists SIDs and homes |

Typical `/etc/oratab`:

```
+ASM:/u01/app/19.0.0/grid:N
prod01:/u01/app/oracle/product/19.0.0/dbhome_1:Y
prod02:/u01/app/oracle/product/19.0.0/dbhome_2:Y
```

The trailing `Y`/`N` controls whether `dbstart` auto-starts the database.

### Read-Only Oracle Home (19c+)

Oracle 19c introduces read-only Oracle Home: state (dbs, network/admin, log, audit) can be relocated outside `$ORACLE_HOME` so binaries are truly read-only. Enabled with `roohctl -enable`. Recommended for gold-image workflows.

After `roohctl -enable`:

- `dbs/`, `network/admin/`, `log/`, `audit/` move to `$ORACLE_BASE/homes/<home_name>/`.
- `$ORACLE_HOME` is remounted read-only in principle (Oracle-supported).
- Simplifies patching (binaries versus config are cleanly separated).

## Important Parameters

Not applicable — see the operating system variables:

| Variable          | Purpose                                  |
| ----------------- | ---------------------------------------- |
| `ORACLE_HOME`     | Full path to the Oracle Home             |
| `ORACLE_BASE`     | Parent directory of `product/<version>`  |
| `ORACLE_SID`      | Instance identifier                      |
| `PATH`            | Must include `$ORACLE_HOME/bin`          |
| `LD_LIBRARY_PATH` | Must include `$ORACLE_HOME/lib`          |
| `TNS_ADMIN`       | Optional override for TNS files location |
| `NLS_LANG`        | Client character set and language        |

## Important Views

Post-install:

```sql
-- Confirm this instance's Oracle Home
SHOW PARAMETER background_dump_dest;    -- prior to ADR
SELECT sys_context('userenv','oracle_home') FROM dual;

-- 19c+
SELECT sys_context('userenv','oracle_home_name') FROM dual;
```

## Diagnostic Queries

```bash
# List all Oracle Homes registered in the central inventory
cat /etc/oraInst.loc
cat /u01/app/oraInventory/ContentsXML/inventory.xml

# List patches applied to the current home
$ORACLE_HOME/OPatch/opatch lsinventory

# Current environment
echo "ORACLE_HOME=$ORACLE_HOME"
echo "ORACLE_SID=$ORACLE_SID"
which sqlplus
ldd $ORACLE_HOME/bin/oracle | head
```

```sql
-- What home does this instance run from?
SELECT distinct sys_context('userenv','oracle_home') FROM dual;

-- What is the software version?
SELECT banner_full FROM v$version;
```

## Common Issues

- **`ORACLE_HOME` mismatch after upgrade** — Instance running from old home. Restart with correct environment.
- **`libclntsh.so: cannot open shared object file`** — `LD_LIBRARY_PATH` missing `$ORACLE_HOME/lib`.
- **Two homes with same version** — Central inventory tracks them separately (different GUIDs). Confusing but supported.
- **`ORACLE_HOME` on NFS** — Supported but requires specific mount options; performance suffers. Local disk preferred.
- **Detaching a home fails** — Try `runInstaller -detachHome` from within the home; if that fails, edit `inventory.xml` manually (last resort).
- **Multiple databases sharing a home** — Fully supported. Patching updates _all_ databases running from that home simultaneously.

## Troubleshooting

1. Wrong version reported by `sqlplus`: check `PATH`. `which sqlplus` should point to the intended home.
2. Instance won't start: verify `ORACLE_SID` and that a matching `spfile<SID>.ora` or `init<SID>.ora` exists in `$ORACLE_HOME/dbs`.
3. `oraInventory` corruption: back up `inventory.xml` before any manual edit. Rebuild with `attachHome`.
4. Patching fails with "home not registered": run `runInstaller -attachHome ORACLE_HOME=... ORACLE_HOME_NAME=...`.

## Best Practices

1. Follow OFA. `$ORACLE_HOME = $ORACLE_BASE/product/<version>/dbhome_<n>`.
2. **One database version per home**. Never mix 12c and 19c binaries.
3. Use `oraenv` (from `/usr/local/bin/oraenv`) and `oratab` to set environment consistently.
4. Gold-image your patched homes; deploy identical binaries across all servers of the same tier.
5. Enable read-only Oracle Home for cleaner patching.
6. Patch **out-of-place**: install the new RU into a _new_ home, then move databases across, rather than in-place patching. Recovery from a failed patch is much faster.
7. Never remove `.patch_storage` — you lose the ability to roll back patches.

## Interview Questions

1. **Q:** What is `$ORACLE_HOME`?
   **A:** The directory containing the Oracle Database software binaries, libraries, and configuration for a specific version.

2. **Q:** What is `$ORACLE_BASE`?
   **A:** The parent of `product/<version>` — the root of Oracle's diagnostic and configuration hierarchy.

3. **Q:** Can two databases share an `$ORACLE_HOME`?
   **A:** Yes — very common. All share the same binaries and patch level.

4. **Q:** What is a read-only Oracle Home?
   **A:** A 19c+ configuration where state (`dbs`, `network/admin`, `log`, `audit`) moves out of `$ORACLE_HOME` into `$ORACLE_BASE/homes/<home_name>/`, allowing the binary directory to be treated as immutable.

5. **Q:** Why patch out-of-place?
   **A:** To keep the pre-patch home intact for immediate rollback and to reduce downtime — the new home is patched offline, then databases are switched during a brief outage.

6. **Q:** Where is the central inventory?
   **A:** Path is in `/etc/oraInst.loc`; typically `/u01/app/oraInventory`.

## References

- Oracle Database Installation Guide 19c
- Oracle Database Administrator's Guide 19c — Managing Oracle Software
- MOS Doc ID 2298666.1 — Read-Only Oracle Home in 19c
- MOS Doc ID 2419319.1 — Creating and Deploying a Gold Image
- MOS Doc ID 555.1 — Latest Release Update
