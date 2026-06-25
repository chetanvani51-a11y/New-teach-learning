Bilkul! Seedha yahan explain karta hoon — simple Hinglish mein, koi file nahi.

---

# Snowflake SQL Worksheet — Complete Hinglish Explanation

---

## PROJECT KYA HAI? (Pehle Samajhte Hain)

Hum AWS ka pricing data (CSV file) Snowflake database mein load kar rahe hain. Phir usse 3 layers mein process karke business insights nikalenge — jaise "kaunsa AWS region sasta hai, kaunsa mahanga."

**Architecture flow:**
CSV File → Snowflake Stage → RAW Table → CURATED Table → REPORTING View → Business Queries

---

## STEP 1 — Context Verify Karo

```sql
SELECT CURRENT_ROLE(),
CURRENT_WAREHOUSE(),
CURRENT_DATABASE(),
CURRENT_SCHEMA();
```

**Kya karta hai:** Yeh check karta hai ki aap Snowflake mein sahi jagah ho ya nahi.

- `CURRENT_ROLE()` — tumhara permission level kya hai (jaise ACCOUNTADMIN)
- `CURRENT_WAREHOUSE()` — kaunsa compute engine chal raha hai (jaise COMPUTE_WH)
- `CURRENT_DATABASE()` — kaunsi database selected hai
- `CURRENT_SCHEMA()` — kaunsa schema selected hai

**Kyun karte hain:** Socho GPS check karne jaisa — pehle apni location confirm karo, tabhi aage badhho. Agar galat role ya warehouse ho to commands fail ho sakti hain ya galat jagah kaam ho sakta hai.

---

## STEP 2 — Database Banao

```sql
CREATE DATABASE AWS_COST_DB;
```

**Kya karta hai:** Ek bilkul nayi, khali database banata hai sirf is project ke liye.

**Kyun karte hain:** Socho ek nayi filing cabinet kharidi jisme sirf AWS pricing ke documents rakhe jayenge. Agar sab kuch ek hi database mein daalo to bahut confusion hoga. Alag database = clean isolation.

---

## STEP 3 — Database Use Karo

```sql
USE DATABASE AWS_COST_DB;
```

**Kya karta hai:** Snowflake ko bolta hai — "ab mera saara kaam isi database ke andar hoga."

**Kyun karte hain:** Bina `USE` ke Snowflake ko pata nahi chalega ki tumhara kaam kahan hona chahiye. Yeh context set karna hai.

---

## STEP 4 — Teen Schemas Banao

```sql
CREATE SCHEMA RAW;
CREATE SCHEMA CURATED;
CREATE SCHEMA REPORTING;
```

**Kya karta hai:** Database ke andar teen alag folders banate hain.

- `RAW` — bilkul original data, jaisa CSV se aaya waisa, koi changes nahi
- `CURATED` — saaf aur trusted data, business use ke liye ready
- `REPORTING` — analysts aur dashboards ke liye simplified views

**Kyun karte hain:** Yeh "Medallion Architecture" hai — Bronze se Silver se Gold. Isse data quality track hoti hai. Business users galat ya raw data nahi dekhte. Har layer ka ek clear kaam hota hai.

---

## STEP 5 — Schemas Verify Karo

```sql
SHOW SCHEMAS;
```

**Kya karta hai:** Database mein jo bhi schemas hain unki list dikhata hai.

**Kyun karte hain:** Confirm karna hai ki RAW, CURATED, REPORTING — teeno ban gaye. Ek verification step hai.

---

## STEP 6 — RAW Schema Switch Karo

```sql
USE SCHEMA RAW;
```

**Kya karta hai:** Context ko RAW folder mein le jaata hai.

**Kyun karte hain:** Stage aur raw table RAW schema mein banana hai, isliye pehle wahan jaana zaroori hai.

---

## STEP 7 — Internal Stage Banao

```sql
CREATE STAGE AWS_PRICING_STAGE;
```

**Kya karta hai:** Ek temporary holding area banata hai jahan CSV files rakhte hain table mein daalne se pehle.

**Kyun karte hain:** Snowflake mein directly file se table mein load nahi kar sakte. Stage ek intermediate dock jaisa hai — maal pehle dock pe aata hai, phir warehouse ke andar jaata hai. Real world mein yeh S3, Azure Blob ya Google Cloud Storage se bhi connected ho sakti hai. Is project mein hum internal stage use kar rahe hain.

---

## STEP 8 — Stage Verify Karo

```sql
SHOW STAGES;
```

**Kya karta hai:** Saari stages ki list dikhata hai.

**Kyun karte hain:** Confirm karna ki `AWS_PRICING_STAGE` ban gayi.

---

## STEP 9 — Files Dekho Jo Stage Mein Hain

```sql
LIST @AWSDATA;
```

**Kya karta hai:** Stage ke andar jo CSV files upload ki hain unki list dikhata hai. `@` symbol batata hai yeh stage ka reference hai.

**Kyun karte hain:** Load karne se pehle confirm karna ki files actually wahan hain ya nahi. Agar file nahi hai to `COPY INTO` fail ho jaayega. Yeh Linux ka `ls` ya Windows ka `dir` command jaisa hai.

---

## STEP 10 — File Format Banao

```sql
CREATE OR REPLACE FILE FORMAT AWS_CSV_FORMAT
TYPE = CSV
SKIP_HEADER = 1
FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

**Kya karta hai:** Snowflake ko batata hai ki CSV file kaise padhni hai.

- `TYPE = CSV` — yeh comma-separated file hai
- `SKIP_HEADER = 1` — pehli row header hai (column names), use data mein mat daalo
- `FIELD_OPTIONALLY_ENCLOSED_BY = '"'` — kuch CSV fields double quotes mein hote hain, unhe sahi se handle karo

**Kyun karte hain:** Bina file format ke Snowflake nahi samjhega ki CSV ka pehla row data hai ya header. Galat format = garbled data. Yeh Snowflake ko "reading instructions" dena hai.

---

## STEP 11 — RAW Table Banao

```sql
CREATE OR REPLACE TABLE RAW_AURORA_DSQL_PRICING
(
  SKU STRING,
  OfferTermCode STRING,
  RateCode STRING,
  TermType STRING,
  PriceDescription STRING,
  EffectiveDate STRING,
  PricePerUnit STRING,
  Currency STRING,
  Location STRING,
  serviceName STRING
  -- ...aur 10 aur columns
);
```

**Kya karta hai:** Ek nayi table banata hai jisme CSV ka data store hoga. `CREATE OR REPLACE` matlab agar pehle se table hai to usse replace karo.

**Kyun sab columns STRING hain:** RAW layer ka kaam hai data exactly preserve karna, jaisa CSV mein tha. Agar hum `INTEGER` ya `FLOAT` daalen aur CSV mein koi invalid value ho to load fail ho jaayega. String mein sab kuch aa jaata hai, cleaning baad mein CURATED layer mein karenge.

---

## STEP 12 — Table Verify Karo

```sql
SHOW TABLES;
```

**Kya karta hai:** Current schema mein saari tables list karta hai.

**Kyun karte hain:** Confirm karna ki `RAW_AURORA_DSQL_PRICING` ban gayi.

---

## STEP 13 — Data Load Karo (COPY INTO)

```sql
COPY INTO RAW_AURORA_DSQL_PRICING
FROM @AWSDATA/AuroraDSQL.csv
FILE_FORMAT = AWS_CSV_FORMAT;
```

**Kya karta hai:** Stage mein rakhi CSV file ka data table mein copy karta hai. Yeh Snowflake ka bulk data loading command hai — bahut fast hota hai.

**Kyun karte hain:** Yeh poore pipeline ka sabse important step hai. Stage se table mein data aata hai isi se. `FILE_FORMAT` batata hai ki file kaise padhni hai.

Ek important baat — `COPY INTO` by default idempotent hai, matlab same file dobara run karo to duplicate rows nahi banengi.

---

## STEP 14 — Count Validate Karo

```sql
SELECT COUNT(*)
FROM RAW_AURORA_DSQL_PRICING;
```

**Kya karta hai:** Table mein total kitni rows hain yeh count karta hai.

**Kyun karte hain:** Load ke baad confirm karna zaroori hai ki data actually aaya ya nahi. 40 rows expect hain. Agar 0 aaye to koi issue hai. Yeh "receipt check karo" wali step hai.

---

## STEP 15 — Data Preview Karo

```sql
SELECT *
FROM RAW_AURORA_DSQL_PRICING
LIMIT 10;
```

**Kya karta hai:** Table ki pehli 10 rows dikhata hai.

**Kyun karte hain:** Aankhon se dekho ki data sahi aaya ya nahi. Columns mein sahi values hain ya garbled data — yeh quickly check ho jaata hai. `LIMIT 10` isliye kyunki poori 40 rows dekhna zaroori nahi, sample kaafi hai.

---

## STEP 16 — CURATED Schema Switch karo aur Table Banao

```sql
USE SCHEMA CURATED;

CREATE OR REPLACE TABLE AURORA_DSQL_PRICING AS
SELECT *
FROM AWS_COST_DB.RAW.RAW_AURORA_DSQL_PRICING;
```

**Kya karta hai:** `CREATE TABLE AS SELECT (CTAS)` — ek hi command mein table bhi banata hai aur data bhi copy karta hai. RAW table ka saara data CURATED table mein aa jaata hai.

**Kyun karte hain:** Business users ko directly RAW data nahi dikhana chahiye. CURATED layer mein future mein type casting, data cleaning, aur transformations add kar sakte ho. RAW layer untouched rehti hai — agar kuch bhi galat ho to wahan se fresh start kar sakte ho.

---

## STEP 17 — CURATED Data Validate Karo

```sql
SELECT COUNT(*)
FROM AURORA_DSQL_PRICING;
```

**Kya karta hai:** CURATED table mein bhi 40 rows hain ya nahi confirm karta hai.

**Kyun karte hain:** RAW se CURATED mein sab data gaya ya kuch reh gaya — yeh verify karna.

---

## STEP 18 — REPORTING View Banao

```sql
USE SCHEMA REPORTING;

CREATE OR REPLACE VIEW COST_ANALYSIS_VIEW AS
SELECT
  Location,
  serviceName,
  PricePerUnit,
  Currency,
  Unit
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING;
```

**Kya karta hai:** `VIEW` ek virtual table hai — actual data store nahi hota, sirf ek saved query hai. Jab tum VIEW query karte ho tab underlying CURATED table se live data aata hai. Humne sirf 5 relevant columns rakhe hain, baaki 15 chupa diye.

**Kyun karte hain:** Analysts ko saare 20 columns nahi chahiye — confusing hoga. VIEW unhe sirf relevant columns dikhata hai. Agar CURATED data change ho to VIEW automatically updated rehta hai. Yeh data ka "simplified window" hai.

---

## STEP 19 — Reporting View Validate Karo

```sql
SELECT *
FROM COST_ANALYSIS_VIEW
LIMIT 10;
```

**Kya karta hai:** VIEW se pehle 10 rows preview karta hai.

**Kyun karte hain:** Confirm karna ki VIEW sahi columns de raha hai.

---

## DEMO QUERY 1 — Sabse Mahanga Region Kaun Sa Hai?

```sql
SELECT
  Location,
  MAX(PricePerUnit) AS COST
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
GROUP BY Location
ORDER BY COST DESC;
```

**Kya karta hai:** Har region ko group karta hai aur uska maximum price nikalata hai. `ORDER BY COST DESC` — highest price wale region pehle dikhenge.

**Business value:** Ek DevOps/Cloud engineer decide kar sakta hai ki client ka workload kahan deploy karna sasta padega. Expected result — Frankfurt, London, Paris, Hong Kong sabse mehnge hain.

---

## DEMO QUERY 2 — Har Region Mein Kitne Records Hain?

```sql
SELECT
  Region_Code,
  COUNT(*) AS RECORDS
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
GROUP BY Region_Code;
```

**Kya karta hai:** Har region code ke liye count karta hai ki pricing ke kitne entries hain.

**Business value:** Samajhna ki Snowflake mein kitne regions ka data load hua — coverage check.

---

## DEMO QUERY 3 — Saste Se Mahange Region Tak

```sql
SELECT
  Location,
  PricePerUnit,
  Currency
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
ORDER BY PricePerUnit DESC;
```

**Kya karta hai:** Sabhi locations ka price ek list mein dikhata hai, highest se lowest order mein.

**Business value:** Direct comparison — client ko batao ki kahan deploy karna cost-effective hai.

---

## STEP 20 — Summary Reporting View Banao

```sql
CREATE OR REPLACE VIEW REGION_COST_SUMMARY AS
SELECT
  LOCATION,
  MAX(PRICEPERUNIT) AS MAX_COST,
  MIN(PRICEPERUNIT) AS MIN_COST,
  COUNT(*) AS RECORD_COUNT
FROM AWS_COST_DB.CURATED.AURORA_DSQL_PRICING
GROUP BY LOCATION;
```

**Kya karta hai:** Har region ke liye ek row mein — maximum price, minimum price, aur total records. Aggregated summary view hai.

**Kyun karte hain:** Business leaders ko detail nahi chahiye. Unhe ek table chahiye jisme clearly dikhe "is region mein maximum kitna charge hoga." Yeh executive-level reporting view hai.

---

## STEP 21 — Summary Review Karo

```sql
SELECT *
FROM REGION_COST_SUMMARY
ORDER BY MAX_COST DESC;
```

**Kya karta hai:** Summary view ka data dikhata hai — sabse expensive region pehle.

---

## STEP 22 — Final Verification

```sql
SHOW VIEWS;
SHOW TABLES IN SCHEMA AWS_COST_DB.CURATED;
```

**Kya karta hai:** Project ke end mein confirm karo ki saari views aur tables ban gayi hain.

**Expected output:**
- Views: `COST_ANALYSIS_VIEW`, `REGION_COST_SUMMARY`
- Tables: `AURORA_DSQL_PRICING` (CURATED), `RAW_AURORA_DSQL_PRICING` (RAW)

---

# ROLE DIVISION — Kaun Karega Kya?

Ab sabse important part — is poore project mein kaam ka batwara kaise hoga:

---

## DevOps Engineer Ki Responsibility

- Snowflake account setup aur configuration karna
- Database (`AWS_COST_DB`) aur schemas (RAW, CURATED, REPORTING) banana
- Internal Stage create karna aur CSV files upload karna
- File Format define karna
- `COPY INTO` pipeline setup karna — yeh automation ka hissa hai
- Warehouse sizing aur cost control karna
- Access roles manage karna — kaun kya dekh sakta hai
- GitHub Actions ya Airflow se pipeline automate karna

Seedha bolo to — **infrastructure aur pipeline DevOps ka kaam hai.**

---

## Data Engineer Ki Responsibility

- RAW table ka schema design karna
- CURATED layer mein data cleaning aur transformation karna — jaise STRING columns ko proper FLOAT/INTEGER mein convert karna
- Data quality checks add karna — null values, duplicates handle karna
- Incremental load logic banana — sirf naya data load ho, poora dobara nahi
- dbt ya Snowflake Tasks se transformations automate karna
- Pipeline monitoring aur error handling

Seedha bolo to — **data ko raw se trusted banana Data Engineer ka kaam hai.**

---

## Data Analyst Ki Responsibility

- REPORTING views use karke analysis karna
- Business questions ke liye SQL queries likhna — jaise Demo Query 1, 2, 3
- `MAX`, `MIN`, `GROUP BY` se pricing insights nikalna
- Tableau, Power BI ya Snowsight se dashboards banana
- Client ko samjhana ki kaunsa region sasta hai
- `REGION_COST_SUMMARY` use karke executive reports banana

Seedha bolo to — **data se business decisions nikalna Data Analyst ka kaam hai.**

---

## Summary Table

| Kaam | DevOps | Data Engineer | Data Analyst |
|---|---|---|---|
| Database, Schema banana | ✅ | | |
| Stage setup + CSV upload | ✅ | | |
| COPY INTO pipeline | ✅ | | |
| RAW table design | | ✅ | |
| CURATED transformations | | ✅ | |
| REPORTING views banana | | ✅ | |
| Business queries likhna | | | ✅ |
| Dashboard banana | | | ✅ |
| Client insights dena | | | ✅ |

Real projects mein yeh teeno alag alag log hote hain. Lekin agar ek hi banda teeno kaam kar raha hai — to usse "Full Stack Data Engineer" ya "Data Platform Engineer" kehte hain.
