# Type matrix: SQL Server → Qlik Replicate → Snowflake

**Source:** Qlik Replicate Help, May 2026 (default mapping, no task override).  
**Use:** migrate-table document. Column to CREATE on Snowflake = **right column**. Middle column = Qlik internal (not a Snowflake type).

How to read:

```
SQL Server CREATE TABLE  →  Qlik Replicate internal  →  Snowflake CREATE TABLE
```

Official pages:

- SQL Server → Qlik: https://help.qlik.com/en-US/replicate/May2026/Content/Replicate/Main/SQL%20Server/SQLServerDB_source_DataTypes.htm
- Qlik internal types: https://help.qlik.com/en-US/replicate/May2026/Content/Replicate/Main/Endpoints/att_rep_data_types.htm
- Qlik → Snowflake: https://help.qlik.com/en-US/replicate/May2026/Content/Replicate/Main/Snowflake-Target/Snowflake-target-datatypes.htm

Qlik note: on Snowflake, `INT` / `INTEGER` / `BIGINT` / `SMALLINT` / `TINYINT` / `BYTEINT` are always stored as **`NUMBER(38,0)`**.

`TIMESTAMP` on Snowflake below means Qlik’s `TIMESTAMP(precision)` (typically `TIMESTAMP_NTZ(p)`). Confirm `p` on 保全.

---

## Matrix (default)

| SQL Server (`CREATE TABLE`) | Qlik Replicate (internal) | Snowflake (`CREATE TABLE`) | Notes |
|---|---|---|---|
| `BIT` | `BOOL` | `BOOLEAN` | |
| `TINYINT` | `UINT1` | `NUMBER(38,0)` | Qlik UINT1 → BYTEINT; on Snowflake integers = NUMBER(38,0) |
| `SMALLINT` | `INT2` | `NUMBER(38,0)` | |
| `INT` | `INT4` | `NUMBER(38,0)` | |
| `BIGINT` | `INT8` | `NUMBER(38,0)` | |
| `DECIMAL(p,s)` | `NUMERIC` | `NUMBER(p,s)` | If scale 0–37. If `p > 38`, Qlik/Snowflake limit — do not guess; confirm (stop if unclear) |
| `NUMERIC(p,s)` | `NUMERIC` | `NUMBER(p,s)` | Same as DECIMAL |
| `SMALLMONEY` | `NUMERIC(10,4)` | `NUMBER(10,4)` | |
| `MONEY` | `NUMERIC(19,4)` | `NUMBER(19,4)` | |
| `REAL` | `REAL4` | `FLOAT` | Qlik: FLOAT4 (Snowflake alias of FLOAT) |
| `FLOAT` | `REAL8` | `FLOAT` | Qlik: FLOAT8 |
| `DATE` | `DATE` | `DATE` | |
| `TIME` / `TIME(n)` | `STRING(16)` | `VARCHAR(16)` | **Not** Snowflake `TIME` |
| `SMALLDATETIME` | `DATETIME` | `TIMESTAMP(p)` | Confirm `p` on 保全 (often 0) |
| `DATETIME` | `DATETIME` | `TIMESTAMP(p)` | Confirm `p` on 保全 (often 3) |
| `DATETIME2` / `DATETIME2(n)` | `DATETIME` | `TIMESTAMP(n)` | `n` = source fractional precision 0–7 |
| `DATETIMEOFFSET` / `DATETIMEOFFSET(n)` | `STRING` | `VARCHAR(len)` | **Not** `TIMESTAMP_TZ`. Length not fixed in Qlik source table — confirm on 保全 |
| `CHAR(n)` | `STRING` | `VARCHAR(n)` | Length in bytes; cap 16777216 |
| `VARCHAR(n)` | `STRING` | `VARCHAR(n)` | Cap 16777216 |
| `VARCHAR(MAX)` | `CLOB` | `VARCHAR(16777216)` | Needs CLOB enabled on Qlik task |
| `TEXT` | `CLOB` | `VARCHAR(16777216)` | Same as VARCHAR(MAX) |
| `NCHAR(n)` | `WSTRING` | `VARCHAR(...)` | Snowflake VARCHAR is Unicode. Qlik WSTRING length is bytes: 1–21845 → VARCHAR(that length); larger → VARCHAR(16777216). Confirm character vs byte length on 保全 |
| `NVARCHAR(n)` | `WSTRING` | `VARCHAR(...)` | Same WSTRING rule as NCHAR |
| `NVARCHAR(MAX)` | `NCLOB` | `VARCHAR(16777216)` | Qlik: NVARCHAR(16777216). Needs NCLOB enabled |
| `NTEXT` | `NCLOB` | `VARCHAR(16777216)` | Same as NVARCHAR(MAX) |
| `SYSNAME` | `WSTRING` | `VARCHAR(128)` | SYSNAME = NVARCHAR(128) NOT NULL |
| `BINARY(n)` | `BYTES` | `BINARY(n)` | Cap 8388608 |
| `VARBINARY(n)` | `BYTES` | `BINARY(n)` | Cap 8388608 |
| `VARBINARY(MAX)` | `BLOB` | `BINARY(8388608)` | Needs BLOB enabled |
| `IMAGE` | `BLOB` | `BINARY(8388608)` | Same as VARBINARY(MAX) |
| `TIMESTAMP` / `ROWVERSION` | `BYTES` | `BINARY(8)` | **Not a datetime.** 8-byte version stamp |
| `UNIQUEIDENTIFIER` | `STRING` | `VARCHAR(36)` | Length not explicit in Qlik source table; 36 is GUID text. Confirm on 保全 |
| `XML` | `CLOB` | `VARCHAR(16777216)` | If Qlik treats as XML subtype → `VARIANT`. Needs CLOB/NCLOB per Qlik note. Confirm on 保全 |
| `JSON` | `NCLOB` (JSON subtype) | `VARIANT` | JSON subtype → VARIANT on Snowflake |
| `HIERARCHYID` | `VARCHAR(x)` | `VARCHAR(x)` | `x` not specified — confirm on 保全 |
| `GEOMETRY` | `NCLOB` | `VARCHAR(16777216)` | Text, not Snowflake GEOMETRY |
| `GEOGRAPHY` | `NCLOB` | `VARCHAR(16777216)` | Text, not Snowflake GEOGRAPHY |
| `SQL_VARIANT` | **Unsupported** | **Do not CREATE** | Stop. Ask customer |
| `CURSOR` | **Unsupported** | **Do not CREATE** | Stop |
| `TABLE` (table type) | **Unsupported** | **Do not CREATE** | Stop |
| User-defined type (UDT) | Same as **base type** | Same as base type | If base type unknown → stop |

---

## Qlik → Snowflake length rules (from Snowflake target page)

Apply after the SQL Server → Qlik step.

| Qlik internal | Snowflake |
|---|---|
| `STRING` length 1–16777216 | `VARCHAR(length in bytes)` |
| `STRING` length 16777217–2147483647 | `VARCHAR(16777216)` |
| `STRING` subtype JSON or XML | `VARIANT` (XML subtype not with Snowpipe Streaming) |
| `WSTRING` length 1–21845 | `VARCHAR(length in bytes)` |
| `WSTRING` length 21846–2147483647 | `VARCHAR(16777216)` |
| `WSTRING` subtype JSON or XML | `VARIANT` |
| `BYTES` length 1–8388608 | `BINARY(length)` |
| `BYTES` length 8388609–2147483647 | `BINARY(8388608)` |
| `CLOB` | `VARCHAR(16777216)` or `VARIANT` if JSON/XML subtype |
| `NCLOB` | `NVARCHAR(16777216)` (= VARCHAR on Snowflake) or `VARIANT` if JSON/XML subtype |
| `BLOB` | `BINARY(8388608)` |
| `NUMERIC` scale 0–37 | `NUMBER(p,s)` |
| `NUMERIC` scale 38–127 | `NUMBER(length)` |
| `INT1`/`INT2`/`INT4`/`INT8`/`UINT1`… | `NUMBER(38,0)` on Snowflake |

---

## What to put on the migrate-table (CREATE)

Use the **Snowflake** column as the table type.

Keep SQL Server + Qlik columns on the document so nobody “corrects” it to SnowConvert types (e.g. do not change `TIME` → Snowflake `TIME`, or `DATETIMEOFFSET` → `TIMESTAMP_TZ`, unless 保全 + approval say so).

If 保全 DDL differs from this matrix: record 保全 as fact; customer chooses; do not invent a third type.
