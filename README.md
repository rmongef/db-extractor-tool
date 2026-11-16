# SqlCsvTool - SQL Server table export / import / copy for Revenue Manager

Move selected tables between SQL Server databases in a controlled way, with a desktop window (GUI) or a command line. Built for Revenue Manager (RM) databases, but it works with any SQL Server database.

- Export tables to CSV (with row filters, optional anonymization and optional zip)
- Import CSV files back, in foreign-key-safe order, inside a single transaction
- Copy tables straight from one database to another, with no CSV files
- Check a read-only `SELECT` and work out which related parent/child tables an export needs (`sqlcheck`)
- Dump the complete schema with all primary and foreign keys (`dump_schema`)
- Claude skills that know the RM 9.0.30 database structure and can plan an export for you

> Safety first: import deletes rows in the target tables before loading. Run a dry run first, and never point the tool at a production database you have not backed up.

## Contents

1. [What is in this repository](#what-is-in-this-repository)
2. [Requirements](#requirements)
3. [Quick start](#quick-start)
4. [The window (GUI)](#the-window-gui)
5. [Command line](#command-line)
6. [Configuration](#configuration)
7. [Export, import and copy behaviour](#export-import-and-copy-behaviour)
8. [Dependency-aware filters](#dependency-aware-filters)
9. [SQL check (`sqlcheck.py`)](#sql-check-sqlcheckpy)
10. [Schema dump (`dump_schema.py`)](#schema-dump-dump_schemapy)
11. [Claude skills](#claude-skills)
12. [Build the .exe files](#build-the-exe-files)
13. [Tests](#tests)
14. [Troubleshooting](#troubleshooting)
15. [Limitations](#limitations)
16. [Contributing and security](#contributing-and-security)

## What is in this repository

| File | Purpose |
|---|---|
| `main.py` | Engine and command-line program (export, import, copy, test, update-config) |
| `gui.py` | Tkinter window (`SqlCsvTool.exe` when built) |
| `subset.py` | Builds WHERE filters for related tables from foreign keys |
| `sqlcheck.py` | Validates a read-only `SELECT` and plans the related tables (engine and CLI) |
| `sqlcheck_panel.py` | The "SQL check" panel on the GUI Connection tab |
| `dump_schema.py` | Writes `schema.sql`, `edges.json` and `key-report.txt` from a live database |
| `configuration.json` | Settings and table list (use the sample values; see [Contributing and security](#contributing-and-security)) |
| `TablesGroup.json` | Named groups of tables for quick selection in the GUI |
| `build_exe.bat`, `BUILD-EXE.md`, `*.spec` | Build the portable Windows executables |
| `GUI-Manual.html` | Full user manual for the window |
| `CLI-Help.txt`, `CLI-Help-extra.txt` | Command-line reference |
| `Skill/` | Claude skills: `rm-export-planner`, `rm-sql-expert-structure` |
| `test_*.py` | Offline unit tests |
| `README-Changes.md`, `test-plan.md`, `test-matrix*.md` | Change summary and test documents |

## Requirements

- Windows 10/11 (the executables are Windows-only; the Python sources run wherever `pyodbc` and an ODBC driver run)
- Python 3.12 or newer, with tcl/tk if you want the window (see [dependencies.txt](./dependencies.txt))
- `pyodbc`
- A SQL Server ODBC driver: ODBC Driver 18 or 17 for SQL Server (or the legacy "SQL Server" driver)
- Network access to the SQL Server instances and a login with enough permissions (read for export; read, delete and insert for import)

## Quick start

```powershell
python --version
python -m pip install pyodbc
python -c "import tkinter; print('tk ok')"     # only needed for the window
```

Start the window:

```powershell
python gui.py
```

Or use the command line:

```powershell
# 1. add every table of the source database to the configuration (all start skipped)
python main.py --mode update-config --config configuration.json --server SOURCE-SQL --database SourceDb

# 2. edit configuration.json: set "skip": false and a "where" filter on the tables you want

# 3. read-only check of every included table (filters, row counts)
python main.py --mode test --config configuration.json

# 4. export, validate, import
python main.py --mode export --config configuration.json --csv .\csv
python main.py --mode import --config configuration.json --csv .\csv --dry-run
python main.py --mode import --config configuration.json --csv .\csv
```

SQL logins are prompted at run time; passwords are never written to `configuration.json`. With `"authentication": "windows"` the current Windows identity is used and nothing is prompted.

## The window (GUI)

`gui.py` (or `SqlCsvTool.exe`) has four tabs:

1. **Connection** - server, database, authentication for export and import, plus the **SQL check** panel
2. **Tables** - the table list with Use/Skip, WHERE filter, table groups from `TablesGroup.json`, and **Dependencies / Auto-filter...**
3. **Run** - export, import, copy, test and update-config with a live log
4. **Settings** - null token, delimiter, update-sequence script, version-compare and other options

The complete walk-through, with examples, is in [GUI-Manual.html](./GUI-Manual.html) (open it in a browser).

## Command line

```text
python main.py --mode export|import|update-config|test|copy --config <configuration.json> [options]
```

| Mode | What it does |
|---|---|
| `export` | Writes one CSV per included table to `--csv` |
| `import` | Loads the CSVs from `--csv` into the import database (clears the included tables first) |
| `copy` | Inserts the included tables straight from the export (source) database into the import (target) database. No CSV, nothing deleted. Add `--dry-run` to only check |
| `test` | Read-only check of every included table (filters, row counts). Takes no other options |
| `update-config` | Adds the tables that exist in the database but not in the configuration (as skipped) |

| Option | Meaning |
|---|---|
| `--config <path>` | Required. The configuration file |
| `--csv <path>` | Export output folder or import input folder |
| `--server`, `--database` | Override the configured server / database (required for `update-config`) |
| `--source-server`, `--source-database`, `--target-server`, `--target-database` | `copy` mode only |
| `--dry-run` | Import/copy: validate without changing anything. `update-config`: write `tables_to_add.json` instead of changing the configuration |
| `--anonymize` | Export/copy: scramble the fields listed in `anonymize_fields` |
| `--fixed-values` | Export/copy: write the constants listed in `fixed_values` instead of the database values |
| `--zip` | Export only: also create a `.zip` of the CSV files |

Exit codes: `0` success, `1` error. The full reference, including the two helper programs below, is in [CLI-Help.txt](./CLI-Help.txt).

## Configuration

Settings are read from `configuration.json`. Command-line values override it.

```json
{
  "connection": {
    "authentication": "sql",
    "odbc_driver": "ODBC Driver 18 for SQL Server",
    "encrypt": true,
    "trust_server_certificate": true
  },
  "databases": {
    "export": { "server": "SOURCE-SQL", "database": "SourceDatabase" },
    "import": { "server": "TARGET-SQL", "database": "TargetDatabase" }
  },
  "csv": { "null_token": "__NULL__", "delimiter": ";" },
  "import_null_substitutions": { "dbo.SCRIPT.VERSION_TX": "" },
  "update_sequence": {
    "auto_update_sequence": false,
    "update_sequence_script_path": "C:\\Excelergy\\RevenueManager\\DB\\UpdateSequences.bat",
    "run_in_background": false,
    "foreground_timeout_seconds": 900
  },
  "reference_code_path_review": { "enabled": false, "output_file_name": "reference_code_paths_to_review.csv" },
  "anonymize_fields": [ "CUSTOMER.EMAIL_ADDRESS", "dbo.GUEST.LAST_NAME" ],
  "tables": [
    { "table": "dbo.ExampleTable", "where": "1 = 0", "skip": false },
    { "table": "dbo.PRICE_PROFILE_DETAIL", "where": "1 = 1", "skip": false, "from_date": "20240101" }
  ]
}
```

Key points:

- `tables[].table` is `Name` or `schema.Name`. `where` is an optional export filter; `1 = 0` exports headers only and is the safe default. `skip: true` keeps the table in the file but ignores it in export, import and copy (no export, no validation, no clearing, no load).
- `from_date` (`"YYYYMMDD"`) is supported only for `PRICE_PROFILE_DETAIL` and adds `READ_DT >= from_date`, combined with `where` using `AND`.
- `csv.null_token` is written for SQL `NULL` and converted back on import. `csv.delimiter` is used for both directions (a semicolon is safer than a comma for text data).
- `anonymize_fields` accepts `Table.Column` or `schema.Table.Column`. Anonymization is type-aware (text, binary, GUID, numeric, date, XML), applies at export/copy only, and `NULL` stays `NULL`. A column type that cannot be scrambled safely stops the run with an error.
- `import_null_substitutions` maps `Table.Column` to a replacement used when the CSV holds the null token. If a `NOT NULL` column has no substitution, the destination default is used; character columns with no default get an empty string; anything else stops with the exact table, line and column.
- `update_sequence` optionally runs RM's `UpdateSequences.bat` after a successful import (never during export). `run_in_background`, `foreground_timeout_seconds` and the schema-owner password prompt are explained in the manual. The password can also be supplied through the `SQLCSVTOOL_SCHEMA_PASSWORD` environment variable.
- `reference_code_path_review` writes a CSV of active `REFERENCE_CODE` rows whose meaning looks like an absolute path, so they can be reviewed after import.
- `dependency_links` and `dependency_sql` store the dependency settings edited in the GUI (see below).

## Export, import and copy behaviour

Export:

- Tables are processed in configuration order, one `schema.table.csv` per non-skipped table. A table with zero rows still gets a header-only file.
- A configured table that does not exist in the source is reported and skipped; the run continues.
- Binary values (`binary`, `varbinary`, `image`) are written as `0x`-prefixed hexadecimal and re-imported byte for byte (compressed blobs are not decompressed).
- The CSV folder is created automatically.

Import:

1. Reads the table list and ignores skipped tables.
2. Compares the RM version in the package's `dbo.SYSTEM_PARAMETER.csv` (`PARAM_CD = 'VERS'`) with the target and warns when they differ.
3. Orders tables by foreign keys: parents are loaded first, tables are cleared in reverse order. Self-referencing tables are handled by temporarily disabling and re-checking their constraints. A multi-table cycle stops the import and names the tables.
4. Validates every CSV header. Column order may differ and extra CSV columns are ignored; missing required columns fail validation.
5. Stops before deleting anything if a skipped or unconfigured child table still references a table that would be cleared.
6. Clears and loads in a single transaction. A failed import is rolled back completely.
7. Optionally writes the `REFERENCE_CODE` review file and runs the update-sequence script.

Copy: the same ordering and rules as import, but rows go directly from the source database to the target with no CSV files and no deletes.

`--dry-run` on import or copy validates files, headers and row counts without modifying the database.

## Dependency-aware filters

Exporting one customer or account usually needs the related rows in other tables. In the **Tables** tab, **Dependencies / Auto-filter...** builds a WHERE clause for every other selected table from the foreign keys declared in the database:

- pass 1 (down): tables that reference the root keep only rows that point at rows already kept
- pass 2 (up): tables referenced by any kept table keep only the rows they point to (it never goes back down, so a customer's other accounts are not pulled in)

Only declared foreign keys among the selected tables are used. Relationships that RM stores without foreign keys (rows owned through a class name plus an id, such as CONTACT, INTERACTION, DOCUMENT) can be added by hand as dependency links. The logic lives in `subset.py`.

## SQL check (`sqlcheck.py`)

Type a read-only `SELECT` on one root table, for example `select * from CUSTOMER where CUST_ID = 3020`. The check:

- accepts a single `SELECT` only and rejects anything that writes or runs code
- validates the syntax on the server with `SET PARSEONLY ON` and `sp_describe_first_result_set`; the query is never executed
- with **Check dependencies** ticked, lists related parent and child tables, builds a filter for each, and returns an export plan

In the window it is the panel on the **Connection** tab; **Apply plan to table list** fills the Tables tab (nothing is saved until you click Save). It also runs from the command line:

```powershell
# syntax check only
python sqlcheck.py --config configuration.json --sql "select * from CUSTOMER where CUST_ID = 3020"

# with dependencies, a plan, and a scratch configuration ready to export
python sqlcheck.py --config configuration.json --sql "select * from CUSTOMER where CUST_ID = 3020" ^
    --deps --expand --scope group:Accounts --write-config configuration.sqlcheck.json

# no database reachable: use a saved foreign-key file
python sqlcheck.py --offline --edges-file schema-dump\edges.json --sql "select * from ACCOUNT where ACCT_ID = 1" --deps
```

Useful flags: `--sql` / `--sql-file`, `--deps`, `--depth`, `--expand`, `--scope config|group:<name>|tables:A,B`, `--groups-file`, `--no-children`, `--no-parents`, `--edges-file`, `--dump-edges`, `--write-config`, `--json`. Exit codes: `0` ok, `1` problems found, `2` usage error. Note that the plan always includes tables referencing a kept table, so "no children" only takes effect together with "no parents".

## Schema dump (`dump_schema.py`)

Writes the complete schema of a live database from the catalog views:

```powershell
python dump_schema.py --config configuration.json --server SOURCE-SQL --database SourceDb --out-dir schema-dump
```

| Output | Content |
|---|---|
| `schema.sql` | Tables, primary keys, unique constraints and foreign keys |
| `edges.json` | Foreign-key graph for `sqlcheck.py --edges-file` |
| `key-report.txt` | Counts, tables without a primary key, tables without any foreign key |

`--keys-only` leaves the CREATE TABLE statements out. Review `schema.sql` before sharing it publicly; it describes your database structure.

## Claude skills

The `Skill/` folder holds two skills for Claude:

- `rm-export-planner` - turns a plain-language request into one root-table `SELECT`, runs `sqlcheck.py`, shows the plan for confirmation and prepares a scratch configuration for export, with a dry run before any import. It never asks for passwords.
- `rm-sql-expert-structure` - reference knowledge of the RM 9.0.30 database generated from the install scripts: about 176 tables with columns and comments, declared foreign keys, indexes, check constraints, sequences, naming conventions and the relationships that have no foreign key. It also lists where the install scripts differ from a live RM90_30 database.

Install a skill by adding its folder to Claude as a custom skill.

## Build the .exe files

`build_exe.bat` builds `SqlCsvTool.exe` (window) and `SqlCsvToolCli.exe` (console) into `dist\SqlCsvTool\`, writes `CLI-Help.txt` and copies the manual next to them. Run the program from that folder, not from `build\`.

```powershell
.\build_exe.bat
```

Requirements and details are in [BUILD-EXE.md](./BUILD-EXE.md). The target machine still needs the Microsoft ODBC driver.

## Tests

The unit tests need no database or window:

```powershell
python -m unittest test_sqlcheck_panel test_sqlcheck test_dump_schema
```

`test-plan.md` and the `test-matrix*.md` files describe the manual test scenarios.

## Troubleshooting

| Message | What to do |
|---|---|
| `Missing dependency 'pyodbc'` | `python -m pip install pyodbc` (or use the build virtual environment's Python) |
| No SQL Server ODBC driver found | Install ODBC Driver 17/18, or set `connection.odbc_driver` |
| SQL connection failed | Check driver, server, authentication, credentials and network; the message names server, database and driver |
| `Failed to load Python DLL` when starting the exe | You started an exe from `build\`; use `dist\SqlCsvTool\SqlCsvTool.exe` and rebuild after code changes |
| `ModuleNotFoundError: sqlcheck` | Run the scripts from the project folder, or from a copy that contains all `.py` files |
| CSV header mismatch | The CSV must contain every required destination column |
| Cannot insert NULL into a column | See `import_null_substitutions` and the NULL rules above |
| Foreign key dependency cycle detected | Remove a table from the set, or break the cycle in the target |
| Cannot clear imported tables (non-imported children reference them) | Include the child tables, remove their `skip`, or skip the parent too |
| RM version mismatch warning | Expect schema drift; review `skip`, `import_null_substitutions` and destination differences |
| Malformed numeric/date/time value | The message gives the table, CSV line, column and SQL type; fix the CSV |

## Limitations

- Export order follows the configuration; import order follows foreign keys.
- Only declared foreign keys are used for ordering and dependency discovery. Class-owned RM rows (no foreign key) must be added by hand.
- Import is interactive (credential prompts) unless Windows authentication is used.
- During import the CSV folder must exist; during export it is created.
- The update-sequence batch file may end with an interactive `PAUSE`, which can look like a hang when run unattended.
- `sqlcheck` validates and plans; it never runs the query. The GUI panel's layout should be checked once in a real window after upgrading.

## Contributing and security

- Keep `dependencies.txt` and this README up to date when behaviour changes, and run the tests before opening a pull request.
- Do not commit real server names, database names, passwords, exported CSV files or `schema-dump` output from a customer system. Commit `configuration.json` only with placeholder values; a `.gitignore` that excludes `csv/`, `dist/`, `build/`, `.build-venv/`, `__pycache__/`, `*.bak*` and `schema-dump/` is recommended.
- Report security issues privately to the maintainers rather than in a public issue.

Revenue Manager and Excelergy are trademarks of their respective owners; this project is not affiliated with them.
