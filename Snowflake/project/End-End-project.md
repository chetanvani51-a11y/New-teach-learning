Let's build this properly. I am your guide on this. Every decision has a reason.

---

# The Project — "RetailPulse"

A complete end-to-end data pipeline. Real data, real tools, real architecture.

---

## What We Are Building

A retail analytics platform. Raw sales data from Kaggle → S3 → AWS Glue cleans and loads → Snowflake stores → dbt transforms into business-ready tables → final analytics queries answer real business questions.

When this is done, you can say in any interview — "I built a full ELT pipeline using Glue, Snowflake, and dbt on real retail data." And you can explain every single decision.

---

## Dataset — What to Pick from Kaggle

**Pick this one: "E-Commerce Sales Dataset" by Olist (Brazilian E-Commerce)**

Search on Kaggle: `olist brazilian ecommerce`

Why this dataset specifically:
- It has 9 separate CSV files — multiple sources, just like real life
- It has orders, customers, products, sellers, payments, reviews — complete business picture
- It has dates — so you can practice incremental loads
- It has foreign keys between tables — so joins and relationships make sense
- It is large enough to feel real (100k+ orders) but small enough for free tier
- Real companies use this dataset in interviews to test candidates

Download all 9 CSV files. You will have:
```
olist_orders_dataset.csv
olist_customers_dataset.csv
olist_order_items_dataset.csv
olist_products_dataset.csv
olist_sellers_dataset.csv
olist_order_payments_dataset.csv
olist_order_reviews_dataset.csv
olist_geolocation_dataset.csv
product_category_name_translation.csv
```

---

## Prerequisites — Check These Before Starting

**AWS Account (Free Tier)**
- S3 access — yes, free tier includes it
- AWS Glue — not fully free, but very cheap for small jobs (under $1 for this project)
- IAM — free
- Make sure billing alerts are set — go to AWS Billing → create a budget alert at $5 so no surprises

**Snowflake Account**
- 30-day free trial — $400 credits
- This project will use maybe $2-3 of credits
- You already have this

**dbt Account**
- dbt Cloud free tier — 1 developer seat, free forever
- You already have this

**Your Local Machine**
- Python 3.8+ installed
- AWS CLI installed and configured (`aws configure` with your access key)
- A code editor — VS Code is best
- Git installed
- Basic SQL knowledge — you already have this from the lab

**Kaggle Account**
- Free, just to download the dataset

---

## Project Stages — Complete Breakdown

---

## Stage 1 — Setup Everything ✅

**Step 1.1 — AWS Setup**

Create an S3 bucket. Name it something unique:
```
retailpulse-yourname-raw
```

Inside the bucket, create this folder structure manually:
```
retailpulse-yourname-raw/
  raw/
    orders/
    customers/
    order_items/
    products/
    sellers/
    payments/
    reviews/
    geolocation/
    category_translation/
```

Why separate folders? Because Glue crawlers work per folder. Each folder = one source table. This is the real-world pattern.

**Step 1.2 — Create IAM Role for Glue**

Go to IAM → Roles → Create Role → Choose Glue as the service.

Attach these policies:
- `AmazonS3FullAccess` (for this learning project — in production you'd restrict this)
- `AWSGlueServiceRole`
- `CloudWatchLogsFullAccess`

Name the role: `RetailPulseGlueRole`

Why this matters: every Glue job runs as this role. If the role doesn't have permission, the job fails silently. Setting this right once saves hours of debugging.

**Step 1.3 — Snowflake Setup**

Log into Snowflake. Switch to ACCOUNTADMIN. Run this entire block:

```sql
-- Warehouse
CREATE WAREHOUSE retailpulse_wh
  WAREHOUSE_SIZE = 'XSMALL'
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;

-- Database
CREATE DATABASE retailpulse_db;

-- Schemas
CREATE SCHEMA retailpulse_db.raw;        -- Glue loads here
CREATE SCHEMA retailpulse_db.staging;    -- dbt staging models
CREATE SCHEMA retailpulse_db.marts;      -- dbt final models

-- Role for dbt
CREATE ROLE dbt_role;
GRANT USAGE ON WAREHOUSE retailpulse_wh TO ROLE dbt_role;
GRANT ALL ON DATABASE retailpulse_db TO ROLE dbt_role;
GRANT ALL ON ALL SCHEMAS IN DATABASE retailpulse_db TO ROLE dbt_role;
GRANT ROLE dbt_role TO USER YOUR_USERNAME;  -- replace with your username
```

Why three schemas? Raw is untouched data — Glue owns it. Staging is dbt's first clean layer. Marts is what analysts query. Each layer has one owner, one purpose.

**Step 1.4 — Upload Data to S3**

Download all 9 CSVs from Kaggle. Upload each one to its correct folder:

```
olist_orders_dataset.csv          → raw/orders/
olist_customers_dataset.csv       → raw/customers/
olist_order_items_dataset.csv     → raw/order_items/
olist_products_dataset.csv        → raw/products/
olist_sellers_dataset.csv         → raw/sellers/
olist_order_payments_dataset.csv  → raw/payments/
olist_order_reviews_dataset.csv   → raw/reviews/
olist_geolocation_dataset.csv     → raw/geolocation/
product_category_name_translation.csv → raw/category_translation/
```

You can do this from the S3 console UI — just drag and drop. Or using AWS CLI:
```bash
aws s3 cp olist_orders_dataset.csv s3://retailpulse-yourname-raw/raw/orders/
```

Mark complete when: all 9 files are in S3, each in their own folder.

---

## Stage 2 — AWS Glue — Extract and Load ✅

**What Glue will do:** Read each CSV from S3, do minimal cleaning, load into Snowflake raw schema. No heavy transformation here — that is dbt's job.

**Step 2.1 — Add Snowflake JDBC Driver to S3**

Glue needs a driver to talk to Snowflake. Download the Snowflake JDBC driver:
`snowflake-jdbc-3.13.30.jar`

Get it from: `repo1.maven.org` — search "snowflake jdbc jar"

Upload this JAR to S3:
```
retailpulse-yourname-raw/glue-jars/snowflake-jdbc-3.13.30.jar
```

**Step 2.2 — Store Snowflake Credentials in AWS Secrets Manager**

Never put passwords in code. Go to AWS Secrets Manager → Store a new secret → Other type of secret.

Add these key-value pairs:
```
snowflake_account    →  youraccountid.region.cloud
snowflake_user       →  your_username
snowflake_password   →  your_password
snowflake_database   →  RETAILPULSE_DB
snowflake_schema     →  RAW
snowflake_warehouse  →  RETAILPULSE_WH
```

Name the secret: `retailpulse/snowflake`

**Step 2.3 — Create the Glue Job**

Go to AWS Glue → Jobs → Create Job → Script editor → Python Shell.

Start with the orders table first. Get one working perfectly, then copy the pattern for all others.

```python
import sys
import boto3
import json
import pandas as pd
from awsglue.utils import getResolvedOptions

# Get job parameters
args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_path', 'target_table'])

# Fetch Snowflake credentials from Secrets Manager
secrets_client = boto3.client('secretsmanager', region_name='us-east-1')
secret = secrets_client.get_secret_value(SecretId='retailpulse/snowflake')
creds = json.loads(secret['SecretString'])

# Read CSV from S3
print(f"Reading from {args['source_path']}")
df = pd.read_csv(args['source_path'])

# Basic cleaning — do NOT transform business logic here, that is dbt's job
df.columns = [col.lower().strip() for col in df.columns]  # lowercase column names
df = df.drop_duplicates()                                   # remove exact duplicates
df['_loaded_at'] = pd.Timestamp.now()                       # add audit column

print(f"Rows to load: {len(df)}")

# Connect to Snowflake and load
import snowflake.connector
from snowflake.connector.pandas_tools import write_pandas

conn = snowflake.connector.connect(
    account=creds['snowflake_account'],
    user=creds['snowflake_user'],
    password=creds['snowflake_password'],
    database=creds['snowflake_database'],
    schema=creds['snowflake_schema'],
    warehouse=creds['snowflake_warehouse']
)

# Create table if not exists and load data
write_pandas(
    conn,
    df,
    table_name=args['target_table'].upper(),
    auto_create_table=True,
    overwrite=True
)

conn.close()
print(f"Successfully loaded {len(df)} rows into {args['target_table']}")
```

Job parameters to set (in Glue job → Job details → Job parameters):
```
--source_path      s3://retailpulse-yourname-raw/raw/orders/olist_orders_dataset.csv
--target_table     raw_orders
--additional-python-modules   snowflake-connector-python,pandas
```

The `--additional-python-modules` line tells Glue to install these Python packages automatically. This is the Glue way of doing pip install.

**Step 2.4 — Run and Verify**

Run the job. Check CloudWatch Logs for the output. Then go to Snowflake and verify:

```sql
USE DATABASE retailpulse_db;
USE SCHEMA raw;

SHOW TABLES;
SELECT COUNT(*) FROM raw_orders;
SELECT * FROM raw_orders LIMIT 5;
```

**Step 2.5 — Create Jobs for All 9 Tables**

Copy the same job 8 more times, changing only the parameters:

```
Job Name              source_path                              target_table
orders_load           raw/orders/olist_orders_dataset.csv      raw_orders
customers_load        raw/customers/...                        raw_customers
order_items_load      raw/order_items/...                      raw_order_items
products_load         raw/products/...                         raw_products
sellers_load          raw/sellers/...                          raw_sellers
payments_load         raw/payments/...                         raw_order_payments
reviews_load          raw/reviews/...                          raw_order_reviews
geolocation_load      raw/geolocation/...                      raw_geolocation
category_load         raw/category_translation/...             raw_category_translation
```

**Step 2.6 — Create a Glue Workflow**

Chain all 9 jobs into one workflow that runs in order.

Go to Glue → Workflows → Create Workflow → name it `RetailPulse_Load`

Add trigger → On demand → starts `orders_load`
Add trigger → After `orders_load` succeeds → starts `customers_load`
And so on.

For this project, you can run them in sequence. In production, independent tables would run in parallel — but sequence is fine for learning.

Mark complete when: all 9 raw tables exist in Snowflake with correct row counts. Check each one.

---

## Stage 3 — dbt — Transform and Model ✅

**What dbt will do:** Take raw tables → clean column names and types (staging) → join and apply business logic (intermediate) → build analytics-ready tables (marts).

**Step 3.1 — Connect dbt Cloud to Snowflake**

In dbt Cloud:
- New Project → name it `retailpulse`
- Connection → Snowflake
- Fill in: account, database (`RETAILPULSE_DB`), warehouse (`RETAILPULSE_WH`), role (`dbt_role`)
- Development schema: `staging` (dbt will create models here during development)
- Test connection → should succeed

Connect to a GitHub repo — create a new empty repo called `retailpulse-dbt` and connect it.

**Step 3.2 — Project Structure**

In dbt Cloud's file editor, create this structure:

```
models/
  staging/
    sources.yml
    stg_orders.sql
    stg_customers.sql
    stg_order_items.sql
    stg_products.sql
    stg_sellers.sql
    stg_payments.sql
    stg_reviews.sql
  intermediate/
    int_orders_enriched.sql
  marts/
    fct_orders.sql
    fct_revenue_daily.sql
    dim_customers.sql
    dim_products.sql
```

**Step 3.3 — Define Sources**

Create `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw
    database: retailpulse_db
    schema: raw
    tables:
      - name: raw_orders
        description: "Raw orders data loaded by Glue"
        columns:
          - name: order_id
            description: "Unique order identifier"
      - name: raw_customers
      - name: raw_order_items
      - name: raw_products
      - name: raw_sellers
      - name: raw_order_payments
      - name: raw_order_reviews
      - name: raw_category_translation
```

Why sources.yml? dbt now knows where raw data lives. If Glue stops loading and the table goes stale, dbt can detect it with source freshness checks.

**Step 3.4 — Staging Models**

`models/staging/stg_orders.sql`:

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'raw_orders') }}
),

cleaned AS (
    SELECT
        order_id,
        customer_id,
        order_status,
        CAST(order_purchase_timestamp AS TIMESTAMP)    AS ordered_at,
        CAST(order_approved_at AS TIMESTAMP)           AS approved_at,
        CAST(order_delivered_carrier_date AS TIMESTAMP) AS shipped_at,
        CAST(order_delivered_customer_date AS TIMESTAMP) AS delivered_at,
        CAST(order_estimated_delivery_date AS TIMESTAMP) AS estimated_delivery_at,
        _loaded_at
    FROM source
    WHERE order_id IS NOT NULL
)

SELECT * FROM cleaned
```

`models/staging/stg_customers.sql`:

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'raw_customers') }}
),

cleaned AS (
    SELECT
        customer_id,
        customer_unique_id,
        customer_zip_code_prefix   AS zip_code,
        customer_city              AS city,
        customer_state             AS state
    FROM source
    WHERE customer_id IS NOT NULL
)

SELECT * FROM cleaned
```

`models/staging/stg_order_items.sql`:

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'raw_order_items') }}
),

cleaned AS (
    SELECT
        order_id,
        order_item_id,
        product_id,
        seller_id,
        CAST(shipping_limit_date AS TIMESTAMP)  AS shipping_limit_at,
        CAST(price AS FLOAT)                    AS item_price,
        CAST(freight_value AS FLOAT)            AS freight_value
    FROM source
)

SELECT * FROM cleaned
```

`models/staging/stg_payments.sql`:

```sql
WITH source AS (
    SELECT * FROM {{ source('raw', 'raw_order_payments') }}
),

cleaned AS (
    SELECT
        order_id,
        payment_sequential,
        payment_type,
        payment_installments,
        CAST(payment_value AS FLOAT) AS payment_amount
    FROM source
)

SELECT * FROM cleaned
```

Do the same pattern for stg_products, stg_sellers, stg_reviews — rename columns to clean names, cast types, filter nulls on primary keys.

**Step 3.5 — Intermediate Model**

`models/intermediate/int_orders_enriched.sql`:

```sql
WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
),

customers AS (
    SELECT * FROM {{ ref('stg_customers') }}
),

payments AS (
    SELECT
        order_id,
        SUM(payment_amount) AS total_payment,
        COUNT(DISTINCT payment_type) AS payment_methods_used,
        MAX(payment_installments) AS max_installments
    FROM {{ ref('stg_payments') }}
    GROUP BY order_id
),

items AS (
    SELECT
        order_id,
        COUNT(*) AS item_count,
        SUM(item_price) AS items_subtotal,
        SUM(freight_value) AS total_freight
    FROM {{ ref('stg_order_items') }}
    GROUP BY order_id
)

SELECT
    o.order_id,
    o.customer_id,
    o.order_status,
    o.ordered_at,
    o.approved_at,
    o.delivered_at,
    o.estimated_delivery_at,
    c.city AS customer_city,
    c.state AS customer_state,
    i.item_count,
    i.items_subtotal,
    i.total_freight,
    p.total_payment,
    p.payment_methods_used,
    p.max_installments,
    DATEDIFF('day', o.ordered_at, o.delivered_at) AS delivery_days
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
LEFT JOIN payments p ON o.order_id = p.order_id
LEFT JOIN items i ON o.order_id = i.order_id
```

**Step 3.6 — Mart Models**

`models/marts/fct_orders.sql`:

```sql
{{
  config(
    materialized='table',
    schema='marts'
  )
}}

SELECT
    order_id,
    customer_id,
    order_status,
    ordered_at,
    delivered_at,
    customer_city,
    customer_state,
    item_count,
    items_subtotal,
    total_freight,
    total_payment,
    delivery_days,
    CASE
        WHEN order_status = 'delivered' AND delivery_days <= 7  THEN 'fast'
        WHEN order_status = 'delivered' AND delivery_days <= 14 THEN 'normal'
        WHEN order_status = 'delivered'                         THEN 'slow'
        ELSE 'not_delivered'
    END AS delivery_speed
FROM {{ ref('int_orders_enriched') }}
```

`models/marts/fct_revenue_daily.sql`:

```sql
{{
  config(
    materialized='table',
    schema='marts'
  )
}}

SELECT
    DATE_TRUNC('day', ordered_at)    AS order_date,
    customer_state,
    COUNT(DISTINCT order_id)          AS total_orders,
    SUM(total_payment)                AS gross_revenue,
    SUM(total_freight)                AS total_freight,
    AVG(delivery_days)                AS avg_delivery_days,
    COUNT(DISTINCT customer_id)       AS unique_customers
FROM {{ ref('fct_orders') }}
WHERE order_status = 'delivered'
GROUP BY 1, 2
ORDER BY 1 DESC
```

`models/marts/dim_customers.sql`:

```sql
{{
  config(
    materialized='table',
    schema='marts'
  )
}}

WITH customer_orders AS (
    SELECT
        customer_id,
        COUNT(DISTINCT order_id)    AS total_orders,
        SUM(total_payment)          AS lifetime_value,
        MIN(ordered_at)             AS first_order_at,
        MAX(ordered_at)             AS last_order_at
    FROM {{ ref('fct_orders') }}
    GROUP BY customer_id
)

SELECT
    c.customer_id,
    c.city,
    c.state,
    co.total_orders,
    co.lifetime_value,
    co.first_order_at,
    co.last_order_at,
    DATEDIFF('day', co.first_order_at, co.last_order_at) AS customer_age_days,
    CASE
        WHEN co.lifetime_value >= 1000 THEN 'high_value'
        WHEN co.lifetime_value >= 300  THEN 'mid_value'
        ELSE 'low_value'
    END AS customer_segment
FROM {{ ref('stg_customers') }} c
LEFT JOIN customer_orders co ON c.customer_id = co.customer_id
```

**Step 3.7 — Add Tests**

Create `models/staging/schema.yml`:

```yaml
version: 2

models:
  - name: stg_orders
    description: "Cleaned orders from raw layer"
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - not_null
          - unique
      - name: order_status
        tests:
          - accepted_values:
              values: ['delivered', 'shipped', 'processing', 'canceled',
                       'unavailable', 'invoiced', 'approved', 'created']

  - name: stg_customers
    columns:
      - name: customer_id
        tests:
          - not_null
          - unique

  - name: stg_order_items
    columns:
      - name: order_id
        tests:
          - not_null
      - name: item_price
        tests:
          - not_null

  - name: stg_payments
    columns:
      - name: order_id
        tests:
          - not_null
      - name: payment_amount
        tests:
          - not_null
```

**Step 3.8 — Run dbt**

In dbt Cloud, open the command bar at the bottom:

```bash
# Run all models
dbt run

# Run tests
dbt test

# Run both together
dbt build

# Generate and view documentation
dbt docs generate
dbt docs serve
```

Check the lineage graph in dbt docs — you will see the full DAG from sources through staging through intermediate to marts. This is something most candidates cannot show.

Mark complete when: `dbt build` runs with zero errors, all tests pass, marts tables exist in Snowflake.

---

## Stage 4 — Validate With Business Questions ✅

Now run these queries in Snowflake to prove your pipeline works end to end:

```sql
-- Question 1: Which states generate the most revenue?
SELECT
    customer_state,
    SUM(gross_revenue) AS total_revenue,
    SUM(total_orders) AS total_orders
FROM retailpulse_db.marts.fct_revenue_daily
GROUP BY customer_state
ORDER BY total_revenue DESC
LIMIT 10;

-- Question 2: What percentage of orders are fast vs slow delivery?
SELECT
    delivery_speed,
    COUNT(*) AS orders,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS percentage
FROM retailpulse_db.marts.fct_orders
GROUP BY delivery_speed;

-- Question 3: Who are our highest value customers?
SELECT *
FROM retailpulse_db.marts.dim_customers
WHERE customer_segment = 'high_value'
ORDER BY lifetime_value DESC
LIMIT 20;

-- Question 4: Revenue trend over time
SELECT
    order_date,
    SUM(gross_revenue) AS daily_revenue,
    SUM(total_orders) AS daily_orders
FROM retailpulse_db.marts.fct_revenue_daily
GROUP BY order_date
ORDER BY order_date;
```

If these return meaningful data — your pipeline is working end to end.

---

## Stage 5 — Polish for Portfolio ✅

**Write a README.md** in your GitHub repo. Include:
- What the project does
- Architecture diagram (the flow: S3 → Glue → Snowflake → dbt → Analytics)
- Dataset source
- How to run it
- Sample business questions it answers

**Screenshot the dbt lineage graph** — this is visual proof of your work.

**Screenshot the Snowflake query results** — the business questions above.

**Document one decision you made** — "I chose Python Shell over Spark because data is under 1GB, making Spark unnecessary overhead." This shows thinking, not just execution.

---

## Completion Checklist

```
Stage 1 — Setup
  □ S3 bucket created with folder structure
  □ IAM role created for Glue
  □ Snowflake schemas created (raw, staging, marts)
  □ All 9 CSVs uploaded to S3

Stage 2 — Glue
  □ Snowflake JDBC jar in S3
  □ Secrets Manager configured
  □ All 9 Glue jobs created and tested
  □ All 9 raw tables visible in Snowflake with correct row counts
  □ Glue Workflow created

Stage 3 — dbt
  □ dbt Cloud connected to Snowflake
  □ Sources defined in sources.yml
  □ All staging models running
  □ Intermediate model running
  □ All mart models running
  □ All tests passing
  □ dbt docs generated

Stage 4 — Validation
  □ All 4 business questions return correct data
  □ Row counts make sense end to end

Stage 5 — Portfolio
  □ GitHub repo with README
  □ Lineage graph screenshot
  □ Query result screenshots
```

---

## When You Get Stuck — Where to Look First

Glue job fails → CloudWatch Logs → look for the last printed line before error.

Snowflake permission error → check which role is active, re-run the GRANT statements.

dbt model fails → read the error message fully, it tells you exactly which model and which line.

dbt test fails → go to Snowflake, query that table manually, find the bad rows, understand why.

JDBC connection fails → check Secrets Manager values, check Snowflake account ID format (should be `account.region.cloud`).

---

You have everything you need. Start with Stage 1 today. Come back here when you finish each stage and I will help you with the next one. Every error you hit is part of the learning — do not skip past errors, understand each one.
