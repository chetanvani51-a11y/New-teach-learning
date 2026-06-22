# Complete Snowflake Roadmap for DevOps Engineers (Basic to Advanced)

## Phase 1: Snowflake Fundamentals

### 1.1 Understanding Snowflake Architecture
- Cloud-based data warehouse (runs on AWS, Azure, GCP)
- Three-layer architecture: Storage, Compute (Virtual Warehouses), Cloud Services
- How Snowflake separates storage and compute (pay independently)
- Multi-cluster shared data architecture
- Micro-partitions and data clustering (how Snowflake stores data internally)

### 1.2 Account Setup & Navigation
- Creating a Snowflake trial account
- Snowsight UI walkthrough (worksheets, databases, warehouses, monitoring)
- Understanding editions (Standard, Enterprise, Business Critical)
- Regions and cloud providers

### 1.3 Core SQL for Snowflake
- CREATE / ALTER / DROP — databases, schemas, tables
- SELECT, WHERE, GROUP BY, ORDER BY, HAVING, LIMIT
- Data types: STRING, NUMBER, FLOAT, BOOLEAN, DATE, TIMESTAMP, VARIANT, ARRAY, OBJECT
- INSERT, UPDATE, DELETE, MERGE
- SHOW and DESCRIBE commands
- USE DATABASE / USE SCHEMA / USE WAREHOUSE

## Phase 2: Data Loading & Unloading

### 2.1 Stages
- Internal stages (user, table, named)
- External stages (S3, Azure Blob, GCS)
- CREATE STAGE, LIST @stage, REMOVE @stage

### 2.2 File Formats
- CSV, JSON, Parquet,
Avro,
ORC
- CREATE FILE FORMAT with options (skip_header,
compression,
field_delimiter)

### 2.3 Bulk Loading
- COPY INTO <table> from stage
- Transformation during load (column reordering,
casting)
- Error handling: ON_ERROR,
VALIDATION_MODE
Loading compressed files.
 
### 2.4 Bulk Unloading 
 - COPY INTO @stage from table 
 - Exporting to different formats 
 
### 2.5 Continuous Loading (Snowpipe) 
 - CREATE PIPE with auto-ingest 
 - SQS notifications (AWS), Event Grid (Azure) 
 - Monitoring pipes: SYSTEM$PIPE_STATUS 
 
## Phase 3: Access Control & Security 
 
### 3.1 Role-Based Access Control (RBAC) 
 - System roles: ACCOUNTADMIN,
sysadmin,
sesecurityadmin,
uSERADMIN,
pUBLIC 
default roles and role hierarchy.
gCREATE ROLE,
grant role,
grant privileges.
pPrinciple of least privilege.
defined.
defined.
defined.
defined.
