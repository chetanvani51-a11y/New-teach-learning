# Snowflake for DevOps Engineers

## Complete Learning Notes & Project Revision Guide

---

# Table of Contents

1. What is Snowflake?
2. Why Snowflake Exists
3. Traditional Database vs Snowflake
4. Snowflake Architecture
5. Storage, Compute and Cloud Services
6. Snowflake Objects
7. Database → Schema → Table Hierarchy
8. Stages
9. File Formats
10. Data Loading
11. COPY INTO
12. Data Warehouse Layers
13. Project Walkthrough
14. Analytics Queries
15. DevOps Responsibilities
16. Data Analytics Responsibilities
17. Real World Architecture
18. Common Interview Questions
19. Revision Checklist
20. Next Learning Goals

---

# What is Snowflake?

Snowflake is a Cloud Data Platform used to:

* Store Data
* Process Data
* Analyze Data
* Share Data
* Build Reports
* Run Data Warehouses

Think of Snowflake as:

> "A cloud-based SQL database designed specifically for analytics and large-scale data processing."

---

# Why Snowflake Exists

Before Snowflake:

Companies used:

* Oracle
* SQL Server
* MySQL
* PostgreSQL
* Hadoop

Problem:

As data grew:

* Storage increased
* Compute increased
* Cost increased

Everything was tightly coupled.

If you needed:

```text
More Storage
```

You also had to buy:

```text
More CPU
More Memory
```

Even if you didn't need them.

This was expensive.

---

# Traditional Database Problem

Example:

```text
Server
├── CPU
├── RAM
└── Storage
```

All together.

If storage becomes full:

```text
Need bigger server
```

Even when CPU is idle.

Waste of money.

---

# Snowflake Solution

Snowflake separates:

```text
Storage
Compute
Services
```

independently.

---

# Snowflake Architecture

```text
                +-------------------+
                |  Cloud Services   |
                +---------+---------+
                          |
      ------------------------------------------
      |                    |                   |
      |                    |                   |
+------------+      +------------+      +------------+
| Warehouse  |      | Warehouse  |      | Warehouse  |
| Compute A  |      | Compute B  |      | Compute C  |
+------------+      +------------+      +------------+
      |
      |
      |
+----------------------------------------------+
|            Central Storage Layer             |
+----------------------------------------------+
```

---

# Very Simple Understanding

Imagine Netflix.

Storage:

```text
Movies
Series
Videos
```

Compute:

```text
Users watching videos
```

Services:

```text
Login
Permissions
Metadata
```

All separated.

This is exactly how Snowflake works.

---

# Snowflake Layers Explained

## Storage Layer

Stores:

```text
Tables
Files
CSV
JSON
Parquet
Data
```

Usually backed by:

```text
AWS S3
Azure Blob
Google Cloud Storage
```

You don't manage this directly.

Snowflake manages it.

---

## Compute Layer

Called:

```text
Virtual Warehouse
```

Responsible for:

```text
SELECT
INSERT
UPDATE
DELETE
COPY INTO
Transformations
```

---

## Cloud Services Layer

Handles:

```text
Authentication
Metadata
Query Planning
Optimization
Access Control
Security
```

---

# Why Multiple Warehouses?

Example:

```text
Finance Team
Analytics Team
Data Engineering Team
```

All running queries.

Without separation:

```text
Queries fight for CPU
```

Performance issues.

Snowflake solves this:

```text
Warehouse A
Warehouse B
Warehouse C
```

Independent compute.

---

# Snowflake Object Hierarchy

```text
Account
 └── Database
      └── Schema
           └── Tables
           └── Views
           └── Stages
           └── File Formats
```

---

# Database

Container for business data.

Example:

```sql
CREATE DATABASE AWS_COST_DB;
```

---

# Schema

Logical folder inside database.

Example:

```sql
CREATE SCHEMA RAW;
```

---

# Table

Stores records.

Example:

```sql
CREATE TABLE EMPLOYEE;
```

---

# View

Virtual table.

Stores SQL logic.

Does not store data.

Example:

```sql
CREATE VIEW COST_ANALYSIS_VIEW;
```

---

# Stage

Most Important Concept

Stage = Temporary File Landing Zone

Used before loading data.

```text
CSV
JSON
Parquet
Avro
```

arrives here first.

---

# Stage Flow

```text
CSV File
     |
     v
  Stage
     |
     v
 COPY INTO
     |
     v
 Table
```

---

# Types of Stages

## Internal Stage

Managed by Snowflake

Example:

```text
AWSDATA
```

Files uploaded directly.

---

## External Stage

Connected to:

```text
AWS S3
Azure Blob
Google Cloud Storage
```

Example:

```text
S3 Bucket
   |
   v
Snowflake
```

---

# File Format

Tells Snowflake how to read files.

Example:

```sql
CREATE FILE FORMAT AWS_CSV_FORMAT
TYPE = CSV;
```

---

# Why File Format?

Snowflake must know:

```text
Delimiter
Header
Quotes
Encoding
```

otherwise data loads incorrectly.

---

# COPY INTO

Most commonly used loading command.

```sql
COPY INTO TABLE_NAME
FROM @STAGE;
```

Purpose:

```text
Load file into table
```

---

# Project We Built

Project Name:

```text
AWS Pricing Analytics Platform
```

Goal:

Analyze AWS Aurora DSQL pricing across regions.

---

# Project Architecture

```text
AuroraDSQL.csv
        |
        v
    AWSDATA
      Stage
        |
        v
 COPY INTO
        |
        v
RAW_AURORA_DSQL_PRICING
        |
        v
CURATED.AURORA_DSQL_PRICING
        |
        v
REPORTING.COST_ANALYSIS_VIEW
        |
        v
Analytics Queries
```

---

# Why RAW Layer?

Store source data exactly as received.

No modifications.

Benefits:

```text
Audit
Recovery
Reprocessing
```

---

# Why CURATED Layer?

Store trusted data.

Business-ready.

Benefits:

```text
Cleaner queries
Better governance
Reusable datasets
```

---

# Why REPORTING Layer?

Used by:

```text
Power BI
Tableau
Analysts
Managers
Dashboards
```

---

# Data Warehouse Layer Design

```text
RAW
 |
 v
CURATED
 |
 v
REPORTING
```

Industry standard pattern.

---

# Analytics We Performed

Finding:

```text
Most expensive regions
```

Example:

```text
Frankfurt
Paris
London
Hong Kong
```

---

Finding:

```text
Cheapest regions
```

Example:

```text
N. Virginia
Ohio
Oregon
```

---

# DevOps Engineer Responsibilities

Usually DevOps does NOT:

```text
Build reports
Analyze business metrics
```

Usually DevOps DOES:

```text
Create Snowflake resources
Manage Roles
Manage Warehouses
Manage Stages
Integrate S3
Automate pipelines
Manage Security
CI/CD
Infrastructure
```

---

# What DevOps Controls

```text
Users
Roles
Permissions
Warehouses
Networking
Stages
Storage Integrations
Automation
Monitoring
```

---

# What Data Analysts Control

```text
SQL Queries
Reports
Dashboards
Business KPIs
Insights
```

---

# What Data Engineers Control

```text
ETL Pipelines
Transformations
Data Quality
Data Models
Ingestion
```

---

# ETL Explained

ETL means:

```text
Extract
Transform
Load
```

Example:

```text
AWS Pricing CSV
      |
      v
Extract
      |
      v
Clean Data
      |
      v
Load Snowflake
```

---

# Does Snowflake Provide ETL?

Yes.

Using:

```text
SQL
Tasks
Streams
Snowpipe
Dynamic Tables
Snowpark
```

But many companies also use:

```text
Airflow
Glue
Informatica
Talend
dbt
Matillion
```

---

# Real Production Architecture

```text
AWS Pricing API
        |
        v
      S3
        |
        v
   Snowpipe
        |
        v
      RAW
        |
        v
    CURATED
        |
        v
   REPORTING
        |
        v
Power BI / Tableau
```

---

# Common Interview Questions

## What is Snowflake?

Cloud-native data warehouse platform.

---

## Why Snowflake?

Separates storage and compute.

---

## What is a Warehouse?

Compute engine used to execute queries.

---

## What is a Stage?

Temporary storage location for files.

---

## What is COPY INTO?

Bulk loading command.

---

## Why RAW Layer?

Store source data unchanged.

---

## Why CURATED Layer?

Store cleaned business-ready data.

---

## Why Reporting Layer?

Simplify analytics.

---

## Internal Stage vs External Stage?

Internal:

```text
Managed by Snowflake
```

External:

```text
Connected to S3/Azure/GCS
```

---

# Revision Checklist

Can I explain:

□ Snowflake Architecture

□ Storage Layer

□ Compute Layer

□ Cloud Services Layer

□ Virtual Warehouse

□ Database

□ Schema

□ Table

□ View

□ Stage

□ File Format

□ COPY INTO

□ RAW Layer

□ CURATED Layer

□ REPORTING Layer

□ Internal Stage

□ External Stage

□ ETL

□ Snowpipe

□ DevOps Role

□ Analytics Role

---

# Next Learning Goals

After this project learn:

1. Snowflake Warehouses
2. Roles and RBAC
3. Snowpipe
4. Streams
5. Tasks
6. Dynamic Tables
7. Time Travel
8. Zero Copy Cloning
9. Data Sharing
10. Snowpark
11. Cost Optimization
12. Performance Tuning
13. S3 Integration
14. Terraform for Snowflake
15. CI/CD for Snowflake

---

# Final Summary

Snowflake separates:

```text
Storage
Compute
Services
```

Data loading process:

```text
File
  |
  v
Stage
  |
  v
COPY INTO
  |
  v
Table
```

Data warehouse architecture:

```text
RAW
 |
 v
CURATED
 |
 v
REPORTING
```

DevOps focuses on:

```text
Infrastructure
Security
Automation
Integrations
```

Analytics teams focus on:

```text
Business insights
Reports
Dashboards
```

This project successfully demonstrated a complete Snowflake data ingestion and analytics workflow using AWS pricing datasets.
