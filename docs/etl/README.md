# ETL table-creation pack (SQL Server → Snowflake)

**Phase:** CREATE TABLE only. Do not implement extract/load/CDC apply in this phase.  
**Consumer:** AI Agent that will implement the ETL tool.  
**Normative spec:** [sqlserver-to-snowflake-create-table-spec.md](./sqlserver-to-snowflake-create-table-spec.md)  
**Machine-readable mapping:** [type-conversion-table.yaml](./type-conversion-table.yaml)

The agent MUST treat the spec as the source of truth for generating Snowflake `CREATE TABLE` DDL from SQL Server metadata. Views are out of typed DDL scope. Unmapped types MUST stop table generation.
