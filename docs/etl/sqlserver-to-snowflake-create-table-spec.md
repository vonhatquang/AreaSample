# SQL Server → Snowflake CREATE TABLE specification

**Document type:** Normative input for an AI Agent that implements an ETL tool.  
**Phase:** Table creation (DDL generation) only.  
**Status:** Spec. Do not implement extract, load, merge, or CDC apply in this phase.  
**Mapping file:** [`type-conversion-table.yaml`](./type-conversion-table.yaml) (normative, same version).

This document tells the agent **exactly** how to turn SQL Server table metadata into Snowflake `CREATE TABLE` statements. If a rule here conflicts with a guess, follow this document. If a type is not in the conversion table, **stop**.

---

## 0. RFC 2119

MUST, MUST NOT, SHOULD, MAY are binding for the ETL tool.

---

## 1. Goal

Generate Snowflake table DDL that:

1. Matches **Qlik conversion rules** (SQL Server → Replicate internal type → Snowflake).
2. Can be checked against **保全** SQL Server DDL vs Snowflake DDL (no contradiction with the conversion table).
3. Is safe for **system-used tables** and later **Raw CDC** (types are a contract; ALTER is not casual).
4. Fails closed when a data type has **no conversion rule**.

Customer design-time steps encoded here:

| Step | How this spec uses it |
|---|---|
| ① Qlik conversion rules | Default rows in `type-conversion-table.yaml` |
| ② 保全 DDL compare | Resolution priority below; 保全 observed type wins over docs when recorded |
| ③ Conversion table | YAML + this spec. Agent MUST NOT invent extra mappings |

---

## 2. Scope

### 2.1 In scope

- SQL Server **base tables** (`sys.tables` where `type = 'U'` and not `is_ms_shipped`).
- Column data types, length, precision, scale, nullability.
- Primary key column list (for CDC merge key documentation and optional Snowflake PK clause).
- Schema/table/column identifier conversion.
- `CREATE TABLE` DDL files + machine-readable manifest + error report.
- User-defined types **only after** resolving to a SQL Server base type.

### 2.2 Out of scope (MUST NOT generate in this phase)

- `CREATE VIEW` with declared column types (Snowflake infers view types; secondary use is user-only).
- Extract/load/CDC apply jobs, Snowpipe, streams, tasks.
- Stored procedures, functions, triggers, sequences.
- Indexes other than documenting PK columns.
- Foreign keys, CHECK constraints, unique indexes (document only; do not emit unless config `emit_pk_clause` is true for PK).
- Clustering keys, warehouses, grants, masking policies.
- Data backfill.
- `CREATE OR REPLACE TABLE` as the default path (drops data; unsafe for CDC).

### 2.3 Views

If the source object is a view:

- MUST NOT emit a typed Snowflake table for it under this spec.
- MUST list it in `manifest.skipped_views`.
- MUST NOT guess column types. Views are inferred at query time.

---

## 3. Inputs the agent MUST accept

| Input | Required | Description |
|---|---|---|
| `source_metadata` | YES | SQL Server catalog dump: tables, columns, types, PK. JSON or equivalent from `sys` views. |
| `type-conversion-table.yaml` | YES | This pack’s mapping file. |
| `config.yaml` | YES | Names, quoting, PK emission, CDC metadata columns, 保全 overrides. |
| `hozen_compare.json` | NO | Result of 保全 SQL Server DDL vs Snowflake DDL compare. When present, used in type resolution. |
| `overrides` | NO | Approved per-column Snowflake types. May live inside the YAML `overrides` list or config. |

### 3.1 Minimum `source_metadata` fields per column

```json
{
  "database": "AppDB",
  "schema": "dbo",
  "table": "Customer",
  "object_type": "TABLE",
  "column": "CustomerId",
  "ordinal": 1,
  "sqlserver_type": "int",
  "max_length": 4,
  "precision": 10,
  "scale": 0,
  "is_nullable": false,
  "is_identity": true,
  "is_computed": false,
  "computed_definition": null,
  "is_primary_key": true,
  "default_definition": null,
  "udt_name": null,
  "base_type": null
}
```

If `udt_name` is set, `base_type` MUST be filled (SQL Server system type name). If `base_type` is missing → `UNMAPPED_TYPE`.

### 3.2 Required `config.yaml` keys

```yaml
snowflake:
  database: TBD
  schema_map:
    dbo: RAW_DBO          # source schema → Snowflake schema
    default: RAW          # used when source schema not listed
  quote_identifiers: true # true = preserve SQL Server case with quoted ids
  emit_pk_clause: false   # Snowflake PK is informational; default off
  create_mode: if_not_exists   # if_not_exists | create | replace_explicit
include_cdc_metadata_columns: false
fail_on_yellow: false     # if true, yellow types also require override
stop_on_first_error: false
# if false, continue other tables, but process exit MUST be non-zero if any STOP
```

`create_mode`:

- `if_not_exists` — default. `CREATE TABLE IF NOT EXISTS`.
- `create` — `CREATE TABLE` (fail if exists).
- `replace_explicit` — `CREATE OR REPLACE TABLE` only when the operator sets this. MUST NOT be default.

---

## 4. Outputs the agent MUST produce

```
output/
  ddl/<snowflake_schema>/<snowflake_table>.sql
  manifest.json
  conversion_report.json
  errors.json
```

- If `errors.json` contains any object with `severity: stop`, the ETL table-create run MUST be considered **failed**.
- MUST NOT emit a `.sql` file for a table that has any `stop` error.
- MUST still emit report rows for failed tables (so humans can fill the conversion table).

### 4.1 `manifest.json` (required shape)

```json
{
  "spec_version": 1,
  "mapping_file": "type-conversion-table.yaml",
  "tables_succeeded": [],
  "tables_failed": [],
  "skipped_views": [],
  "sql_files": []
}
```

### 4.2 `conversion_report.json` (one row per source column)

Required fields: `database, schema, table, column, sqlserver_type, qlik_type, snowflake_type, confidence, action, resolution_source, notes`.

`resolution_source` MUST be one of: `override` | `hozen_observed_ddl` | `conversion_table` | `unmapped`.

### 4.3 `errors.json`

```json
{
  "code": "UNMAPPED_TYPE",
  "severity": "stop",
  "schema": "dbo",
  "table": "T",
  "column": "C",
  "sqlserver_type": "sql_variant",
  "message": "No conversion rule. CREATE TABLE skipped."
}
```

---

## 5. Hard rules (tables)

These exist because tables are used by **system processing**. Wrong types waste tests and cause runtime errors. After Raw CDC starts, **ALTER is not casual**.

1. **MUST** create types only from the conversion table (plus override / 保全, see §6).
2. **MUST NOT** guess a Snowflake type for an unknown SQL Server type.
3. **MUST NOT** silently map unknown types to `VARCHAR` or `VARIANT` on **tables**.
4. **MUST STOP** table DDL generation if any column is `table_action: stop` or has no row in the conversion table.
5. **MUST NOT** use `CREATE OR REPLACE TABLE` unless `create_mode: replace_explicit`.
6. **MUST NOT** emit Snowflake `IDENTITY` / `AUTOINCREMENT` for SQL Server `IDENTITY` columns. CDC/load supplies the value. Map as a normal `NUMBER` (or mapped type) and preserve nullability.
7. **MUST NOT** convert SQL Server `TIMESTAMP`/`ROWVERSION` to a datetime type. It is `BINARY(8)`.
8. **MUST** preserve column **ordinal order** from SQL Server (CDC/apply and diffs depend on stable order). New CDC columns later are appended, never inserted in the middle by this generator.
9. **MUST** preserve nullability (`NULL` / `NOT NULL`).
10. **MUST NOT** emit `COLLATE` from SQL Server.
11. **MUST NOT** emit filegroup, `TEXTIMAGE_ON`, `SPARSE`, `FILESTREAM`, `ROWGUIDCOL`, or fillfactor.
12. **MUST** skip computed columns that cannot be expressed as a Snowflake column of the computed result’s **base type**. If the computed column is persisted and `base_type` is known, map `base_type` as a normal column **without** the formula (ETL/CDC should load the persisted value). If not persisted and no value will be loaded → `COMPUTED_UNSUPPORTED` stop, unless config `drop_nonpersisted_computed: true` (then omit column and warn).
13. **MUST NOT** change an existing Snowflake column type as part of this generator. This tool **creates** tables. Schema evolution is a later change-management process (§11).

---

## 6. Type resolution algorithm

The agent MUST implement this function for every table column. Pseudocode is normative.

```
function resolve_type(col, config, mapping, hozen, overrides):
  if col.object_type == "VIEW":
      skip as view
      return

  sql_type = normalize(col.sqlserver_type)   # lowercase, map aliases (rowversion→timestamp, text→varchar_max)

  if col.udt_name is not empty:
      sql_type = normalize(col.base_type)
      if sql_type is empty:
          STOP UNMAPPED_TYPE
          return

  # 1) explicit override (approved)
  ov = find_override(overrides, col)
  if ov:
      return Type(snowflake=ov.snowflake, source="override", confidence=ov.confidence or "yellow")

  # 2) 保全 observed Snowflake type for this column
  hz = find_hozen(hozen, col)
  if hz and hz.snowflake_type:
      # MUST still verify it does not contradict mapping without a note
      mapped = lookup(mapping, sql_type)
      if mapped and canonical(mapped.snowflake) != canonical(hz.snowflake_type):
          record contradiction in conversion_report.notes
          # 保全 wins for CREATE TABLE (actual Qlik output)
      return Type(snowflake=hz.snowflake_type, source="hozen_observed_ddl", confidence="yellow")

  # 3) conversion table
  row = lookup(mapping, sql_type)
  if row is missing OR row.table_action == "stop" OR row.snowflake is null:
      STOP UNMAPPED_TYPE
      return

  sf = apply_length_precision(row, col)   # §7
  if apply_length_precision failed:
      STOP TYPE_PARAM_INVALID
      return

  if config.fail_on_yellow and row.confidence == "yellow" and no override:
      STOP YELLOW_REQUIRES_APPROVAL
      return

  return Type(snowflake=sf, source="conversion_table", qlik=row.qlik, confidence=row.confidence)
```

`normalize` aliases MUST include:

| Source token | Canonical `sqlserver` key |
|---|---|
| `rowversion` | `timestamp` |
| `text` | `varchar_max` |
| `ntext` | `nvarchar_max` |
| `image` | `varbinary_max` |
| `varchar` with `max_length = -1` | `varchar_max` |
| `nvarchar` with `max_length = -1` | `nvarchar_max` |
| `varbinary` with `max_length = -1` | `varbinary_max` |
| `sysname` | `sysname` |
| `decimal` | `decimal` |
| `numeric` | `numeric` |

Any type that still does not match a YAML `sqlserver` key or `sqlserver_aliases` entry → **STOP** `UNMAPPED_TYPE`.

There is **no default** “map to TEXT/VARCHAR”. That is forbidden for tables even if other products (e.g. Snowflake Openflow) document it.

---

## 7. Length, precision, scale

After the YAML row is found, the agent MUST fill parameters.

### 7.1 `NUMBER(p,s)` (`decimal` / `numeric`)

- `p = col.precision`, `s = col.scale`.
- If `p` is null or `p < 1` → `TYPE_PARAM_INVALID`.
- If `p > 38` → **STOP** `PRECISION_EXCEEDS_SNOWFLAKE`. Do not emit `VARCHAR` as a hidden fallback on tables.
- Emit `NUMBER(p,s)`.

### 7.2 Integers (`bigint`, `int`, `smallint`, `tinyint`)

- Always `NUMBER(38,0)` (Qlik Snowflake integer rule).
- Ignore SQL Server `precision` display (10 for int, etc.).

### 7.3 `money` / `smallmoney`

- Fixed `NUMBER(19,4)` and `NUMBER(10,4)`. Ignore catalog precision.

### 7.4 `datetime2` / `time` (SQL Server)

- `datetime2`: `TIMESTAMP_NTZ(n)` with `n = col.scale` (fractional seconds). If `n` null, use `7`.
- `time`: **not** Snowflake `TIME`. Qlik uses `STRING(16)` → `VARCHAR(16)`.

### 7.5 Character types

- `char(n)` / `varchar(n)`: `n = col.max_length` (bytes in SQL Server for non-Unicode). Emit `VARCHAR(n)`.
- `nchar(n)` / `nvarchar(n)`: SQL Server `max_length` is **bytes** (usually `2 * n`). Snowflake `VARCHAR` length is **characters**.  
  **MUST** emit `VARCHAR(n_chars)` where `n_chars = max_length / 2` (integer division), except `nvarchar_max`.
- If `n_chars < 1` after division → `TYPE_PARAM_INVALID`.
- Cap: if computed length > 16777216, use `VARCHAR(16777216)`.

### 7.6 Binary types

- `binary(n)` / `varbinary(n)`: `n = col.max_length` bytes, cap `8388608`.
- `varbinary_max` / `image`: `BINARY(8388608)`.
- `timestamp` / `rowversion`: `BINARY(8)` regardless of catalog length.

### 7.7 `uniqueidentifier`

- `VARCHAR(36)` (8-4-4-4-12).

### 7.8 `datetimeoffset`

- Default `VARCHAR(34)` (Qlik STRING). Do not emit `TIMESTAMP_TZ` unless override.

---

## 8. Column attributes (non-type)

| SQL Server | Snowflake CREATE TABLE |
|---|---|
| `NULL` / `NOT NULL` | Same |
| `IDENTITY(n,m)` | No identity. Type from mapping. Value loaded from source |
| `DEFAULT` literal (`0`, `'x'`, numeric) | MAY emit `DEFAULT` literal if it is a simple constant |
| `DEFAULT` with T-SQL (`GETDATE()`, `NEWSEQUENTIALID()`, `USER_NAME()`) | MUST omit default; note `DEFAULT_SKIPPED` (warning, not stop) |
| `COLLATE` | Omit |
| Computed persisted | Column of mapped base type, no `AS` expression |
| Computed non-persisted | Stop unless `drop_nonpersisted_computed: true` |
| Sparse / FILESTREAM | Ignore storage; keep type |
| `UNIQUEIDENTIFIER` ROWGUIDCOL | Ignore flag; keep type |

Column `COMMENT` MUST be set to a short machine-readable string:

```text
src=sqlserver;type=nvarchar;len=100;nullable=true
```

---

## 9. Table-level DDL rules

### 9.1 Naming

- Snowflake database: `config.snowflake.database`.
- Snowflake schema: `config.snowflake.schema_map[source_schema]` or `schema_map.default`.
- Table name: same as SQL Server table name unless `table_name_map` exists.
- If `quote_identifiers: true`: emit `"TableName"` / `"ColumnName"` preserving case.
- If `quote_identifiers: false`: unquoted identifiers (Snowflake stores uppercase). Agent MUST warn that case is lost.

### 9.2 Primary key

- Always record PK columns in `manifest` and in table `COMMENT`.
- If `emit_pk_clause: true`, emit `PRIMARY KEY (cols)` (Snowflake not enforced like SQL Server). Used as documentation for merge keys.
- MUST NOT emit clustered/nonclustered index DDL.

### 9.3 Table COMMENT

JSON, single line:

```json
{"source_db":"AppDB","source_schema":"dbo","source_table":"Customer","pk":["CustomerId"],"generator":"sqlserver-to-snowflake-create-table","spec_version":1}
```

### 9.4 CDC metadata columns

If `include_cdc_metadata_columns: true`, append columns listed in YAML `cdc_metadata_columns.columns` **after** all source columns. If that list is empty while the flag is true → **STOP** `CDC_METADATA_UNDEFINED` (do not invent names).

---

## 10. DDL template (normative)

```sql
CREATE TABLE IF NOT EXISTS <db>.<schema>.<table> (
  <col> <snowflake_type> [NOT NULL|NULL] COMMENT '<col_comment>',
  ...
)
COMMENT = '<table_comment>'
;
```

Rules:

- One file per table.
- Trailing comma MUST NOT appear after the last column.
- `IF NOT EXISTS` omitted only when `create_mode` is `create` or `replace_explicit`.
- `replace_explicit` uses `CREATE OR REPLACE TABLE` and MUST write a warning into `conversion_report` for that table: `REPLACE_DROPS_DATA`.

Worked example (quoted identifiers, PK not emitted):

Source:

```sql
CREATE TABLE dbo.Customer (
  CustomerId INT IDENTITY(1,1) NOT NULL PRIMARY KEY,
  Name       NVARCHAR(100) NOT NULL,
  Active     BIT NULL,
  CreatedAt  DATETIME2(7) NOT NULL,
  RowVer     ROWVERSION NOT NULL
);
```

Generated:

```sql
CREATE TABLE IF NOT EXISTS RAW_DB.RAW_DBO."Customer" (
  "CustomerId" NUMBER(38,0) NOT NULL COMMENT 'src=sqlserver;type=int;identity=true;nullable=false',
  "Name" VARCHAR(100) NOT NULL COMMENT 'src=sqlserver;type=nvarchar;len=100;nullable=false',
  "Active" BOOLEAN NULL COMMENT 'src=sqlserver;type=bit;nullable=true',
  "CreatedAt" TIMESTAMP_NTZ(7) NOT NULL COMMENT 'src=sqlserver;type=datetime2;scale=7;nullable=false',
  "RowVer" BINARY(8) NOT NULL COMMENT 'src=sqlserver;type=timestamp;nullable=false'
)
COMMENT = '{"source_db":"AppDB","source_schema":"dbo","source_table":"Customer","pk":["CustomerId"],"generator":"sqlserver-to-snowflake-create-table","spec_version":1}'
;
```

Worked example — **STOP** (no SQL file):

```sql
CREATE TABLE dbo.Weird (
  Id INT NOT NULL,
  Payload SQL_VARIANT NULL
);
```

`errors.json`: `UNMAPPED_TYPE` on `Payload`. No `Weird.sql`.

---

## 11. CDC and ALTER (contract for later ETL phases)

This phase only **creates** tables. The agent implementing later phases MUST still obey:

| Change on SQL Server | Allowed on Snowflake Raw CDC table |
|---|---|
| New nullable column | Append column (same mapping rules). Do not reorder existing columns |
| Widen `VARCHAR`/`NVARCHAR` length | MAY `ALTER ... SET DATA TYPE VARCHAR(new)` after mapping |
| Narrow length | Forbidden in-place |
| Change type (`INT`→`BIGINT` is still same Snowflake `NUMBER(38,0)`; `INT`→`VARCHAR` is not) | If mapped Snowflake type is **identical**, no DDL. If mapped type **differs**, forbidden in-place |
| `NULL` → `NOT NULL` | Forbidden in-place |
| `datetime` → `datetimeoffset` | Forbidden in-place (NTZ/VARCHAR vs previous type) |
| Drop column | Out of band; do not auto-drop |

Unmapped type discovered **after go-live** (new column): same as create-time — **STOP apply**, do not ALTER in a guessed type.

---

## 12. 保全 compare (customer ②) — input contract

If `hozen_compare.json` is provided, each entry:

```json
{
  "schema": "dbo",
  "table": "Customer",
  "column": "Name",
  "sqlserver_ddl_type": "nvarchar(100)",
  "snowflake_ddl_type": "VARCHAR(100)",
  "matches_conversion_table": true
}
```

Agent MUST:

1. Use `snowflake_ddl_type` when resolving that column (§6 step 2).
2. If `matches_conversion_table` is false, still use 保全 type, and set report note `HOZEN_DIFFERS_FROM_QLIK_DOC`.
3. If 保全 has extra Snowflake columns not in SQL Server: do not drop them; if `include_cdc_metadata_columns` is false, warn `HOZEN_EXTRA_COLUMN`.
4. If 保全 is missing a SQL Server column: STOP `HOZEN_MISSING_COLUMN` for that table (cannot certify CREATE TABLE).

If `hozen_compare.json` is absent, agent MUST note `HOZEN_NOT_PROVIDED` at manifest level. Generation MAY proceed from the conversion table only.

---

## 13. Error codes

| Code | Severity | When |
|---|---|---|
| `CONFIG_INCOMPLETE` | stop | `snowflake.database` empty or `TBD` |
| `UNMAPPED_TYPE` | stop | Type not in YAML, or `table_action: stop` |
| `TYPE_PARAM_INVALID` | stop | Missing/illegal length or precision |
| `PRECISION_EXCEEDS_SNOWFLAKE` | stop | `decimal`/`numeric` p > 38 |
| `YELLOW_REQUIRES_APPROVAL` | stop | Only if `fail_on_yellow: true` |
| `COMPUTED_UNSUPPORTED` | stop | Non-persisted computed, not dropped by config |
| `CDC_METADATA_UNDEFINED` | stop | Flag true, column list empty |
| `HOZEN_MISSING_COLUMN` | stop | Compare file present, column missing on Snowflake |
| `IDENTITY_AS_AUTOINCREMENT` | stop | Generator attempted Snowflake IDENTITY (must not) |
| `DEFAULT_SKIPPED` | warning | T-SQL default omitted |
| `HOZEN_DIFFERS_FROM_QLIK_DOC` | warning | 保全 type ≠ YAML |
| `HOZEN_NOT_PROVIDED` | warning | Compare file absent |
| `HOZEN_EXTRA_COLUMN` | warning | Extra target column |
| `REPLACE_DROPS_DATA` | warning | `replace_explicit` used |
| `VIEW_SKIPPED` | info | View not emitted as table |

Process exit: non-zero if any `stop`.

---

## 14. What the ETL tool MUST implement in the CREATE TABLE phase

Ordered jobs. Later ETL phases (extract/load) are **not** this document.

1. Load config + YAML mapping + optional 保全 + overrides.
2. Read `source_metadata` (or query SQL Server using the inventory SQL in the appendix — query is allowed; writing Snowflake is not required in spec-only mode).
3. Group columns by table; skip views.
4. Resolve every column (§6–§8).
5. If any column STOPs → no SQL for that table; write errors.
6. Else write `CREATE TABLE` file from template §10.
7. Write `manifest.json`, `conversion_report.json`, `errors.json`.
8. Exit non-zero on any stop.

The agent MUST NOT:

- Call Snowflake `ALTER TABLE` in this phase.
- Auto-approve yellow types.
- Use SnowConvert mappings when they disagree with Qlik (example: SnowConvert `DATETIMEOFFSET → TIMESTAMP_TZ`; Qlik `DATETIMEOFFSET → STRING` → `VARCHAR`). **Qlik wins** unless override/保全 says otherwise.

---

## 15. Qlik vs other mappers (do not mix)

| SQL Server | This spec (Qlik) | Do not use instead |
|---|---|---|
| `INT` | `NUMBER(38,0)` | `NUMBER(10,0)` |
| `TIME` | `VARCHAR(16)` | `TIME` |
| `DATETIMEOFFSET` | `VARCHAR(34)` | `TIMESTAMP_TZ` |
| `TIMESTAMP` (rowversion) | `BINARY(8)` | `TIMESTAMP_NTZ` |
| `XML` | `VARCHAR(16777216)` | auto `VARIANT` |
| `SQL_VARIANT` | **STOP** | `VARIANT` or `TEXT` |
| Unknown type | **STOP** | default `TEXT` |

System tables must match what Qlik will load. A “prettier” Snowflake type that Qlik will not write is a defect.

---

## 16. Appendix — SQL Server inventory (optional collector)

The agent MAY collect `source_metadata` with:

```sql
SELECT
  DB_NAME() AS database_name,
  s.name AS schema_name,
  t.name AS table_name,
  'TABLE' AS object_type,
  c.name AS column_name,
  c.column_id AS ordinal,
  ty.name AS sqlserver_type,
  c.max_length,
  c.precision,
  c.scale,
  c.is_nullable,
  c.is_identity,
  c.is_computed,
  cc.definition AS computed_definition,
  CASE WHEN i.is_primary_key = 1 THEN 1 ELSE 0 END AS is_primary_key,
  dc.definition AS default_definition,
  CASE WHEN ty.is_user_defined = 1 THEN ty.name END AS udt_name,
  basety.name AS base_type
FROM sys.tables t
JOIN sys.schemas s ON s.schema_id = t.schema_id
JOIN sys.columns c ON c.object_id = t.object_id
JOIN sys.types ty ON ty.user_type_id = c.user_type_id
JOIN sys.types basety ON basety.user_type_id = ty.system_type_id
LEFT JOIN sys.computed_columns cc
  ON cc.object_id = c.object_id AND cc.column_id = c.column_id
LEFT JOIN sys.default_constraints dc ON dc.object_id = c.default_object_id
LEFT JOIN sys.index_columns ic
  ON ic.object_id = c.object_id AND ic.column_id = c.column_id
LEFT JOIN sys.indexes i
  ON i.object_id = ic.object_id AND i.index_id = ic.index_id AND i.is_primary_key = 1
WHERE t.is_ms_shipped = 0
ORDER BY s.name, t.name, c.column_id;
```

Views MUST be inventoried separately if needed, and only for skip-listing, not for typed CREATE TABLE.

---

## 17. Appendix — config skeleton

```yaml
snowflake:
  database: RAW_DB
  schema_map:
    dbo: RAW_DBO
    default: RAW
  quote_identifiers: true
  emit_pk_clause: false
  create_mode: if_not_exists
include_cdc_metadata_columns: false
fail_on_yellow: false
stop_on_first_error: false
drop_nonpersisted_computed: false
```

Fill `TBD` before a production run. The agent MUST refuse to emit DDL if `snowflake.database` is `TBD` or empty (`CONFIG_INCOMPLETE` stop for the run).

---

## 18. Definition of done (CREATE TABLE phase)

The phase is done when:

1. Every in-scope SQL Server **table** either has a Snowflake `CREATE TABLE` file **or** a `stop` error with a conversion-table gap (never a guessed type).
2. `conversion_report.json` covers every source column.
3. Integers, `TIME`, `DATETIMEOFFSET`, `ROWVERSION` follow **Qlik**, not SnowConvert.
4. No `CREATE OR REPLACE` unless explicitly configured.
5. Views are skipped, not typed.

Implementing load/CDC is a **later** document.
