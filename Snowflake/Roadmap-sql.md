# SQL Roadmap for DevOps Engineers on Snowflake

## Level 1: Foundation (Essential)
- `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`
- `CREATE`/`DROP`/`ALTER` for databases, schemas, tables
- `SHOW` commands (databases, schemas, tables, stages, warehouses)
- `DESCRIBE` (tables, integrations, stages)
- `USE DATABASE` / `USE SCHEMA`
- Basic data types (`STRING`, `NUMBER`, `TIMESTAMP`, `VARIANT`)

## Level 2: Data Loading & Staging
- `CREATE STAGE` (internal and external)
- `CREATE FILE FORMAT` (CSV, JSON, Parquet)
- `COPY INTO` (bulk loading from stages)
- `LIST @stage` (verify staged files)
- `PUT / GET` (local file operations)
- Storage integrations (`CREATE STORAGE INTEGRATION`)

## Level 3: Access Control & Security
- `CREATE ROLE` / `GRANT` / `REVOKE`
- Create User
- Role hierarchy and privilege management
- Network policies
- Show grants on/to objects (`SHOW GRANTS ON / TO`) 

## Level 4: Automation & Infrastructure
- Create task / create stream (scheduling, CDC)
- Create pipe (Snowpipe for auto-ingest)
- Create procedure (stored procedures)
- Resource monitors (`CREATE RESOURCE MONITOR`) 
- Warehouse management (`CREATE/ALTER WAREHOUSE`, scaling policies)

## Level 5: Monitoring & Cost Control
- ACCOUNT_USAGE views (query history, warehouse metering, storage)
- INFORMATION_SCHEMA queries
- Credit consumption tracking
- Query profiling (`QUERY_HISTORY`, `EXPLAIN`) 

## Level 6: Advanced (As Needed)
- Dynamic tables, materialized views
- Data sharing (`CREATE SHARE`) 
- Streams + tasks for ELT pipelines
- VARIANT/JSON handling (`PARSE_JSON`, lateral flatten)
- Time travel (`AT / BEFORE`) 

---
**Key takeaway for DevOps:** You don't need to master analytical SQL (window functions, complex joins, CTEs). Focus on DDL (creating/managing objects), data loading, access control, and monitoring — that covers 90% of DevOps use cases on Snowflake.

Your current project already covers Levels 1–2 well. Want me to help extend it to cover any of the other levels, or shall we go back to fixing the integration error?
