/*--------------------------------------------------------------------------------------------------

PROJECT: AWS Pricing Analytics Platform using Snowflake

AUTHOR: Chetan Vani

# PROJECT OVERVIEW

AWS provides hundreds of cloud services and pricing varies across regions. As a DevOps Engineer,
it is important to understand service costs, compare pricing between regions, and identify
cost optimization opportunities.

In this project, AWS Pricing CSV data is loaded into Snowflake and processed using a modern
data architecture consisting of:

1. RAW Layer       -> Stores source data exactly as received
2. CURATED Layer   -> Stores cleaned and trusted data
3. REPORTING Layer -> Provides business-friendly views

# OBJECTIVES

✔ Create Snowflake Database Architecture
✔ Create Internal Stage for File Storage
✔ Upload AWS Pricing Dataset
✔ Load CSV Data using COPY INTO
✔ Build RAW Layer
✔ Build CURATED Layer
✔ Build REPORTING Layer
✔ Generate Business Insights using SQL Analytics

# ARCHITECTURE

AWS Pricing CSV File
|
v
Snowflake Internal Stage (AWSDATA)
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
Business Analytics & Reporting

--------------------------------------------------------------------------------------------------*/

/*--------------------------------------------------------------------------------------------------
STEP 1 : VERIFY CURRENT SNOWFLAKE CONTEXT

Purpose:

* Verify current role
* Verify warehouse
* Verify database
* Verify schema

Expected Result:
ACCOUNTADMIN
COMPUTE_WH
--------------------------------------------------------------------------------------------------*/

SELECT CURRENT_ROLE(),
CURRENT_WAREHOUSE(),
CURRENT_DATABASE(),
CURRENT_SCHEMA();

/*--------------------------------------------------------------------------------------------------
STEP 2 : CREATE DATABASE

Purpose:

* Create dedicated database for AWS Pricing Analytics Project

Expected Result:
Database successfully created
--------------------------------------------------------------------------------------------------*/

CREATE DATABASE AWS_COST_DB;

/*--------------------------------------------------------------------------------------------------
STEP 3 : USE DATABASE

Purpose:

* Switch context to AWS_COST_DB
  --------------------------------------------------------------------------------------------------*/

USE DATABASE AWS_COST_DB;

/*--------------------------------------------------------------------------------------------------
STEP 4 : CREATE SCHEMAS

Purpose:

## RAW Schema

Stores original source data exactly as received.

## CURATED Schema

Stores cleaned and business-ready datasets.

## REPORTING Schema

Stores reporting views used by analysts and dashboards.
--------------------------------------------------------------------------------------------------*/

CREATE SCHEMA RAW;

CREATE SCHEMA CURATED;

CREATE SCHEMA REPORTING;

/*--------------------------------------------------------------------------------------------------
STEP 5 : VERIFY SCHEMAS

Purpose:

* Verify schema creation

Expected Result:
RAW
CURATED
REPORTING
--------------------------------------------------------------------------------------------------*/

SHOW SCHEMAS;

/*--------------------------------------------------------------------------------------------------
STEP 6 : SWITCH TO RAW SCHEMA

Purpose:

* RAW schema will contain stages and source tables
  --------------------------------------------------------------------------------------------------*/

USE SCHEMA RAW;

/*--------------------------------------------------------------------------------------------------
STEP 7 : CREATE INTERNAL STAGE

Purpose:

* Stage acts as temporary storage for files before loading into Snowflake tables.

Real World:
Files usually arrive from:

* AWS S3
* Azure Blob
* Google Cloud Storage

In this project:
CSV files are uploaded directly to Snowflake Internal Stage.
--------------------------------------------------------------------------------------------------*/

CREATE STAGE AWS_PRICING_STAGE;

/*--------------------------------------------------------------------------------------------------
STEP 8 : VERIFY STAGE

Expected Result:
AWS_PRICING_STAGE
--------------------------------------------------------------------------------------------------*/

SHOW STAGES;

/*--------------------------------------------------------------------------------------------------
STEP 9 : VERIFY UPLOADED FILES

Purpose:

* Check files uploaded into stage

Expected Result:
AuroraDSQL.csv
OpsWorks.csv
AppFlow.csv
etc.
--------------------------------------------------------------------------------------------------*/

LIST @AWSDATA;

/*--------------------------------------------------------------------------------------------------
STEP 10 : CREATE FILE FORMAT

Purpose:
Tell Snowflake how to read CSV files.

Configuration:
SKIP_HEADER = 1
Ignore first row

FIELD_OPTIONALLY_ENCLOSED_BY = '"'
Handle quoted values correctly
--------------------------------------------------------------------------------------------------*/

CREATE OR REPLACE FILE FORMAT AWS_CSV_FORMAT
TYPE = CSV
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"';

/*--------------------------------------------------------------------------------------------------
STEP 11 : CREATE RAW TABLE

Purpose:
Store Aurora DSQL pricing data exactly as received.

Table:
RAW_AURORA_DSQL_PRICING
--------------------------------------------------------------------------------------------------*/

CREATE OR REPLACE TABLE RAW_AURORA_DSQL_PRICING
(
SKU STRING,
OfferTermCode STRING,
RateCode STRING,
TermType STRING,
PriceDescription STRING,
EffectiveDate STRING,
StartingRange STRING,
EndingRange STRING,
Unit STRING,
PricePerUnit STRING,
Currency STRING,
RelatedTo STRING,
Product_Family STRING,
serviceCode STRING,
Location STRING,
Location_Type STRING,
usageType STRING,
operation STRING,
Region_Code STRING,
serviceName STRING
);

/*--------------------------------------------------------------------------------------------------
STEP 12 : VERIFY TABLE

Expected Result:
RAW_AURORA_DSQL_PRICING
--------------------------------------------------------------------------------------------------*/

SHOW TABLES;

/*--------------------------------------------------------------------------------------------------
STEP 13 : LOAD DATA INTO RAW TABLE

Purpose:
COPY INTO is Snowflake's bulk data ingestion command.

Flow:
Stage
->
Table

Expected Result:
40 rows loaded
--------------------------------------------------------------------------------------------------*/

COPY INTO RAW_AURORA_DSQL_PRICING
FROM @AWSDATA/AuroraDSQL.csv
FILE_FORMAT = AWS_CSV_FORMAT;

/*--------------------------------------------------------------------------------------------------
STEP 14 : VALIDATE DATA LOAD

Expected Result:
40 rows
--------------------------------------------------------------------------------------------------*/

SELECT COUNT(*)
FROM RAW_AURORA_DSQL_PRICING;

/*--------------------------------------------------------------------------------------------------
STEP 15 : REVIEW RAW DATA

Purpose:
Validate data loaded correctly.
--------------------------------------------------------------------------------------------------*/

SELECT *
FROM RAW_AURORA_DSQL_PRICING
LIMIT 10;

/*--------------------------------------------------------------------------------------------------
STEP 16 : CREATE CURATED LAYER

Purpose:
Move trusted data into CURATED schema.

Why?
Business users should not directly query RAW data.
--------------------------------------------------------------------------------------------------*/

USE SCHEMA CURATED;

CREATE OR REPLACE TABLE AURORA_DSQL_PRICING AS
SELECT *
FROM AWS_COST_DB.RAW.RAW_AURORA_DSQL_PRICING;

/*--------------------------------------------------------------------------------------------------
STEP 17 : VALIDATE CURATED DATA

Expected Result:
40 rows
--------------------------------------------------------------------------------------------------*/

SELECT COUNT(*)
FROM AURORA_DSQL_PRICING;

/*--------------------------------------------------------------------------------------------------
STEP 18 : CREATE REPORTING VIEW

Purpose:
Provide simplified business-friendly dataset.

Used By:

* Analysts
* Dashboards
* Reporting Tools
  --------------------------------------------------------------------------------------------------*/

USE SCHEMA REPORTING;

CREATE OR REPLACE VIEW COST_ANALYSIS_VIEW AS
SELECT
Location,
serviceName,
PricePerUnit,
Currency,
Unit
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING;

/*--------------------------------------------------------------------------------------------------
STEP 19 : VALIDATE REPORTING VIEW

Purpose:
Preview reporting data
--------------------------------------------------------------------------------------------------*/

SELECT *
FROM COST_ANALYSIS_VIEW
LIMIT 10;

/*--------------------------------------------------------------------------------------------------
DEMO QUERY 1

Business Question:
Which AWS Regions have the highest Aurora DSQL pricing?

Expected Insight:
Frankfurt, London, Paris, Hong Kong
are among the most expensive regions.
--------------------------------------------------------------------------------------------------*/

SELECT
Location,
MAX(PricePerUnit) AS COST
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
GROUP BY Location
ORDER BY COST DESC;

/*--------------------------------------------------------------------------------------------------
DEMO QUERY 2

Business Question:
How many pricing records exist per AWS Region?

Purpose:
Understand regional coverage.
--------------------------------------------------------------------------------------------------*/

SELECT
Region_Code,
COUNT(*) AS RECORDS
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
GROUP BY Region_Code;

/*--------------------------------------------------------------------------------------------------
DEMO QUERY 3

Business Question:
Compare pricing across all AWS Regions.

Purpose:
Identify low-cost and high-cost deployment regions.
--------------------------------------------------------------------------------------------------*/

SELECT
Location,
PricePerUnit,
Currency
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
ORDER BY PricePerUnit DESC;

/*--------------------------------------------------------------------------------------------------
STEP 20 : CREATE SUMMARY REPORTING VIEW

Purpose:
Provide aggregated cost summary.

Useful For:
Executive Reporting
Dashboarding
Cost Analysis
--------------------------------------------------------------------------------------------------*/

USE SCHEMA REPORTING;

CREATE OR REPLACE VIEW REGION_COST_SUMMARY AS
SELECT
LOCATION,
MAX(PRICEPERUNIT) AS MAX_COST,
MIN(PRICEPERUNIT) AS MIN_COST,
COUNT(*) AS RECORD_COUNT
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
GROUP BY LOCATION;

/*--------------------------------------------------------------------------------------------------
STEP 21 : REVIEW COST SUMMARY

Business Insight:
Identify most expensive and least expensive regions.
--------------------------------------------------------------------------------------------------*/

SELECT *
FROM REGION_COST_SUMMARY
ORDER BY MAX_COST DESC;

/*--------------------------------------------------------------------------------------------------
STEP 22 : VERIFY PROJECT OBJECTS

Expected Views:
COST_ANALYSIS_VIEW
REGION_COST_SUMMARY

Expected Tables:
AURORA_DSQL_PRICING
RAW_AURORA_DSQL_PRICING
--------------------------------------------------------------------------------------------------*/

SHOW VIEWS;

SHOW TABLES IN SCHEMA AWS_COST_DB.CURATED;

/*--------------------------------------------------------------------------------------------------

# PROJECT COMPLETION SUMMARY

Database Created               ✔
Schemas Created                ✔
Internal Stage Created         ✔
CSV Uploaded                   ✔
File Format Created            ✔
Raw Table Created              ✔
COPY INTO Executed             ✔
Raw Data Validated             ✔
Curated Layer Created          ✔
Reporting View Created         ✔
Analytics Queries Executed     ✔
Business Insights Generated    ✔

FINAL STATUS: PROJECT SUCCESSFULLY COMPLETED

--------------------------------------------------------------------------------------------------*/
