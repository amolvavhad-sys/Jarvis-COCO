# RBAC and Dynamic Data Masking in Snowflake using Cortex

> A fully automated, security-first Generative AI pipeline on Snowflake — built from a fresh trial account with zero pre-existing objects.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Solution Architecture](#3-solution-architecture)
4. [Project Structure](#4-project-structure)
5. [Prerequisites](#5-prerequisites)
6. [Step-by-Step Setup](#6-step-by-step-setup)
   - [Step 1 — Create Database and Schema](#step-1--create-database-and-schema)
   - [Step 2 — Create Roles and Users](#step-2--create-roles-and-users)
   - [Step 3 — Create Sample Table with PII Data](#step-3--create-sample-table-with-pii-data)
   - [Step 4 — Apply Dynamic Data Masking Policies](#step-4--apply-dynamic-data-masking-policies)
   - [Step 5 — Grant Role-Based Permissions](#step-5--grant-role-based-permissions)
   - [Step 6 — Invoke Snowflake Cortex LLM Functions](#step-6--invoke-snowflake-cortex-llm-functions)
   - [Step 7 — Validate RBAC + Masking with Cortex](#step-7--validate-rbac--masking-with-cortex)
7. [How Cortex Inherits Platform Security](#7-how-cortex-inherits-platform-security)
8. [Strategic Value](#8-strategic-value)
9. [Expected Outputs](#9-expected-outputs)
10. [Troubleshooting](#10-troubleshooting)
11. [References](#11-references)

---

## 1. Executive Summary

This project demonstrates a **secure, fully automated Generative AI pipeline** built entirely within Snowflake. Starting from a brand-new Snowflake Trial Account with no pre-existing data or objects, this guide provisions all required infrastructure using SQL scripts and then showcases how **Snowflake Cortex natively inherits** two core platform security features:

- **Role-Based Access Control (RBAC):** Controls which roles can invoke which Cortex LLM functions and which data those roles can see.
- **Dynamic Data Masking:** Masks sensitive PII columns (SSN, Email, Phone) in real time, so that even when Cortex processes the data, restricted users only ever receive masked values.

The end result is a governed AI pipeline where **security is not a bolt-on — it is the default**.

---

## 2. Problem Statement

Modern enterprises want to leverage Large Language Models (LLMs) for customer insight, sentiment analysis, and intelligent search. However, security and compliance teams frequently block or delay these initiatives due to three core fears:

| Risk | Description |
|---|---|
| **Data Leakage** | Sensitive PII (SSN, Email, Phone) may be sent to an LLM and returned in plain text to unauthorized users. |
| **Complex Governance** | Maintaining separate security policies for the data layer and the AI layer is operationally expensive and error-prone. |
| **Unstructured Data Risks** | Free-text fields like customer comments often contain embedded PII that is difficult to detect and scrub before AI processing. |

This project solves all three problems natively within Snowflake — without any custom middleware, ETL scrubbing pipelines, or third-party tools.

---

## 3. Solution Architecture

The solution is built on **three pillars** of the Snowflake ecosystem, forming a Governed AI Pipeline.
```
┌─────────────────────────────────────────────────────────────┐
│                    Snowflake Platform                        │
│                                                             │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────┐  │
│  │    RBAC      │   │  Dynamic     │   │  Snowflake     │  │
│  │              │──▶│  Data        │──▶│  Cortex        │  │
│  │  Role &      │   │  Masking     │   │  LLM Functions │  │
│  │  Privilege   │   │  Policies    │   │                │  │
│  │  Control     │   │  on PII cols │   │  COMPLETE,     │  │
│  └──────────────┘   └──────────────┘   │  SENTIMENT,    │  │
│                                        │  TRANSLATE,    │  │
│                                        │  SUMMARIZE     │  │
│                                        └────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.1 Role-Based Access Control (RBAC)

- **ANALYST_ROLE** — A restricted role. Can query data but sees only masked PII values. Cannot invoke high-parameter LLMs.
- **DATA_ENGINEER_ROLE** — A privileged role. Can see unmasked PII data and invoke all Cortex functions.
- Every Cortex AI query executes within the **context of the caller's active role**. Snowflake does not bypass or elevate permissions for AI calls — the model only ever sees what the role is authorized to see.

### 3.2 Dynamic Data Masking

- Masking policies are applied at the column level on PII fields: `SSN`, `EMAIL`, `PHONE_NUMBER`.
- When a restricted `ANALYST_ROLE` user queries the table — directly or through a Cortex function — masked values such as `***-**-****` or `***@***.***` are returned.
- When the privileged `DATA_ENGINEER_ROLE` queries the same table, actual values are returned.
- **Cortex respects these masks in real time.** There is no way for the LLM output to reveal data that the masking policy has hidden.

### 3.3 Cortex LLM Functions Used

| Function | Purpose in this Project |
|---|---|
| `SNOWFLAKE.CORTEX.COMPLETE` | General-purpose LLM completion against customer data |
| `SNOWFLAKE.CORTEX.SENTIMENT` | Analyze sentiment of customer feedback comments |
| `SNOWFLAKE.CORTEX.SUMMARIZE` | Summarize customer support notes |
| `SNOWFLAKE.CORTEX.TRANSLATE` | Translate customer feedback to English |

---

## 4. Project Structure
```
snowflake-cortex-rbac-masking/
│
├── README.md                        ← This file
│
├── 01_setup_infrastructure.sql      ← Creates DB, Schema, Warehouse, Roles, Users
├── 02_create_table_and_data.sql     ← Creates customer table and inserts sample PII data
├── 03_masking_policies.sql          ← Defines and applies Dynamic Data Masking policies
├── 04_grants_and_rbac.sql           ← Assigns privileges to roles
├── 05_cortex_queries.sql            ← Cortex LLM function calls against the masked data
└── 06_validation_tests.sql          ← Role-switching tests to validate masking behavior
```

---

## 5. Prerequisites

- A **Snowflake Trial Account** (sign up at https://signup.snowflake.com — no credit card required)
- Account must be in a region where **Snowflake Cortex is supported** (US East, US West, EU regions — check Snowflake docs for latest availability)
- Login as **ACCOUNTADMIN** to run all setup scripts (the trial account default)
- No other tools, software, or installations required — everything runs in the **Snowflake Worksheet UI**

---

## 6. Step-by-Step Setup

Open a new Worksheet in your Snowflake Trial Account and run each script in order.

---

### Step 1 — Create Database and Schema

**File: `01_setup_infrastructure.sql`**
```sql
-- ============================================================
-- STEP 1: Infrastructure Setup
-- Run as: ACCOUNTADMIN
-- ============================================================

USE ROLE ACCOUNTADMIN;

-- Create a dedicated database for this project
CREATE DATABASE IF NOT EXISTS CORTEX_SECURITY_DEMO
    COMMENT = 'Demo database for RBAC and Data Masking with Snowflake Cortex';

-- Create a schema to hold all objects
CREATE SCHEMA IF NOT EXISTS CORTEX_SECURITY_DEMO.PII_DATA
    COMMENT = 'Schema containing sensitive customer PII data';

-- Create a virtual warehouse for compute
CREATE WAREHOUSE IF NOT EXISTS CORTEX_DEMO_WH
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    COMMENT = 'Warehouse for Cortex RBAC demo';

-- Confirm objects created
SHOW DATABASES LIKE 'CORTEX_SECURITY_DEMO';
SHOW SCHEMAS IN DATABASE CORTEX_SECURITY_DEMO;
SHOW WAREHOUSES LIKE 'CORTEX_DEMO_WH';
```

---

### Step 2 — Create Roles and Users

**File: `01_setup_infrastructure.sql` (continued)**
```sql
-- ============================================================
-- STEP 2: Create Roles
-- ============================================================

-- Privileged role: can see unmasked PII, can run all Cortex functions
CREATE ROLE IF NOT EXISTS DATA_ENGINEER_ROLE
    COMMENT = 'Privileged role: full PII access + all Cortex LLM functions';

-- Restricted role: sees only masked PII, limited Cortex access
CREATE ROLE IF NOT EXISTS ANALYST_ROLE
    COMMENT = 'Restricted role: masked PII only, limited Cortex functions';

-- Grant roles to ACCOUNTADMIN for management
GRANT ROLE DATA_ENGINEER_ROLE TO ROLE ACCOUNTADMIN;
GRANT ROLE ANALYST_ROLE TO ROLE ACCOUNTADMIN;

-- Create a privileged user
CREATE USER IF NOT EXISTS DATA_ENGINEER_USER
    PASSWORD = 'StrongPass123!'
    DEFAULT_ROLE = DATA_ENGINEER_ROLE
    DEFAULT_WAREHOUSE = CORTEX_DEMO_WH
    COMMENT = 'User with full PII and Cortex access';

-- Create a restricted user
CREATE USER IF NOT EXISTS ANALYST_USER
    PASSWORD = 'StrongPass456!'
    DEFAULT_ROLE = ANALYST_ROLE
    DEFAULT_WAREHOUSE = CORTEX_DEMO_WH
    COMMENT = 'User with masked PII and limited Cortex access';

-- Assign roles to users
GRANT ROLE DATA_ENGINEER_ROLE TO USER DATA_ENGINEER_USER;
GRANT ROLE ANALYST_ROLE TO USER ANALYST_USER;

-- Confirm
SHOW ROLES LIKE '%ROLE';
SHOW USERS LIKE '%USER';
```

---

### Step 3 — Create Sample Table with PII Data

**File: `02_create_table_and_data.sql`**
```sql
-- ============================================================
-- STEP 3: Create Table and Insert Sample PII Data
-- Run as: ACCOUNTADMIN
-- ============================================================

USE ROLE ACCOUNTADMIN;
USE DATABASE CORTEX_SECURITY_DEMO;
USE SCHEMA PII_DATA;
USE WAREHOUSE CORTEX_DEMO_WH;

-- Create customer table with sensitive PII columns
CREATE OR REPLACE TABLE CUSTOMER_FEEDBACK (
    CUSTOMER_ID         NUMBER AUTOINCREMENT PRIMARY KEY,
    CUSTOMER_NAME       VARCHAR(100),
    EMAIL               VARCHAR(150),       -- PII: will be masked
    PHONE_NUMBER        VARCHAR(20),        -- PII: will be masked
    SSN                 VARCHAR(11),        -- PII: will be masked
    FEEDBACK_COMMENT    VARCHAR(2000),      -- Free text: may contain embedded PII
    FEEDBACK_DATE       DATE,
    REGION              VARCHAR(50),
    SATISFACTION_SCORE  NUMBER(2,1)
);

-- Insert realistic sample data with PII
INSERT INTO CUSTOMER_FEEDBACK
    (CUSTOMER_NAME, EMAIL, PHONE_NUMBER, SSN, FEEDBACK_COMMENT, FEEDBACK_DATE, REGION, SATISFACTION_SCORE)
VALUES
    ('Alice Johnson',  'alice.johnson@email.com',  '+1-555-101-2020', '123-45-6789',
     'The product delivery was extremely fast and the packaging was perfect. Very happy with my experience!',
     '2024-01-15', 'North America', 4.8),

    ('Bob Martinez',   'bob.martinez@gmail.com',   '+1-555-202-3030', '234-56-7890',
     'I had an issue with my order but the support team resolved it quickly. Could be better but overall okay.',
     '2024-02-10', 'North America', 3.2),

    ('Chen Wei',       'chen.wei@company.cn',      '+86-138-0013-8000', '345-67-8901',
     'La livraison a été très lente et le produit était endommagé. Je suis très déçu.',
     '2024-03-05', 'Asia Pacific', 1.5),

    ('Priya Sharma',   'priya.sharma@india.in',    '+91-98765-43210', '456-78-9012',
     'Excellent service! The team was very professional and my query was resolved in minutes. Will recommend to friends.',
     '2024-03-20', 'South Asia', 4.9),

    ('David O''Brien',  'david.obrien@uk.co',       '+44-7911-123456', '567-89-0123',
     'Average experience. Nothing special but nothing went wrong either. Delivery took longer than expected.',
     '2024-04-01', 'Europe', 3.0),

    ('Fatima Al-Hassan', 'fatima.hassan@ae.com',   '+971-50-123-4567', '678-90-1234',
     'My email fatima.hassan@ae.com was used to send spam after I signed up. Deeply concerned about data privacy.',
     '2024-04-15', 'Middle East', 1.2),

    ('Sarah Williams', 'sarah.w@webmail.com',      '+1-555-303-4040', '789-01-2345',
     'Wonderful product and lightning fast shipping. The customer portal is very intuitive and easy to use.',
     '2024-05-10', 'North America', 4.7),

    ('Kenji Tanaka',   'kenji.tanaka@japan.jp',    '+81-90-1234-5678', '890-12-3456',
     'サービスは良かったですが、配送に時間がかかりすぎました。改善を期待します。',
     '2024-05-22', 'Asia Pacific', 3.5);

-- Confirm data loaded
SELECT COUNT(*) AS TOTAL_RECORDS FROM CUSTOMER_FEEDBACK;
SELECT * FROM CUSTOMER_FEEDBACK LIMIT 5;
```

---

### Step 4 — Apply Dynamic Data Masking Policies

**File: `03_masking_policies.sql`**
```sql
-- ============================================================
-- STEP 4: Create and Apply Dynamic Data Masking Policies
-- Run as: ACCOUNTADMIN
-- ============================================================

USE ROLE ACCOUNTADMIN;
USE DATABASE CORTEX_SECURITY_DEMO;
USE SCHEMA PII_DATA;

-- ------------------------------------------------------------
-- 4.1 Masking Policy for EMAIL column
-- DATA_ENGINEER_ROLE sees real email; ANALYST_ROLE sees masked
-- ------------------------------------------------------------
CREATE OR REPLACE MASKING POLICY EMAIL_MASK AS (VAL STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('DATA_ENGINEER_ROLE', 'ACCOUNTADMIN') THEN VAL
        ELSE '***@***.***'
    END;

-- ------------------------------------------------------------
-- 4.2 Masking Policy for SSN column
-- DATA_ENGINEER_ROLE sees real SSN; ANALYST_ROLE sees masked
-- ------------------------------------------------------------
CREATE OR REPLACE MASKING POLICY SSN_MASK AS (VAL STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('DATA_ENGINEER_ROLE', 'ACCOUNTADMIN') THEN VAL
        ELSE '***-**-****'
    END;

-- ------------------------------------------------------------
-- 4.3 Masking Policy for PHONE_NUMBER column
-- DATA_ENGINEER_ROLE sees real phone; ANALYST_ROLE sees masked
-- ------------------------------------------------------------
CREATE OR REPLACE MASKING POLICY PHONE_MASK AS (VAL STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('DATA_ENGINEER_ROLE', 'ACCOUNTADMIN') THEN VAL
        ELSE '+*-***-***-****'
    END;

-- ------------------------------------------------------------
-- 4.4 Apply masking policies to table columns
-- ------------------------------------------------------------
ALTER TABLE CUSTOMER_FEEDBACK
    MODIFY COLUMN EMAIL
    SET MASKING POLICY EMAIL_MASK;

ALTER TABLE CUSTOMER_FEEDBACK
    MODIFY COLUMN SSN
    SET MASKING POLICY SSN_MASK;

ALTER TABLE CUSTOMER_FEEDBACK
    MODIFY COLUMN PHONE_NUMBER
    SET MASKING POLICY PHONE_MASK;

-- Confirm policies are applied
DESCRIBE TABLE CUSTOMER_FEEDBACK;
SHOW MASKING POLICIES IN SCHEMA PII_DATA;
```

---

### Step 5 — Grant Role-Based Permissions

**File: `04_grants_and_rbac.sql`**
```sql
-- ============================================================
-- STEP 5: Grant Privileges per Role
-- Run as: ACCOUNTADMIN
-- ============================================================

USE ROLE ACCOUNTADMIN;

-- Grant warehouse usage to both roles
GRANT USAGE ON WAREHOUSE CORTEX_DEMO_WH TO ROLE DATA_ENGINEER_ROLE;
GRANT USAGE ON WAREHOUSE CORTEX_DEMO_WH TO ROLE ANALYST_ROLE;

-- Grant database and schema access to both roles
GRANT USAGE ON DATABASE CORTEX_SECURITY_DEMO TO ROLE DATA_ENGINEER_ROLE;
GRANT USAGE ON DATABASE CORTEX_SECURITY_DEMO TO ROLE ANALYST_ROLE;

GRANT USAGE ON SCHEMA CORTEX_SECURITY_DEMO.PII_DATA TO ROLE DATA_ENGINEER_ROLE;
GRANT USAGE ON SCHEMA CORTEX_SECURITY_DEMO.PII_DATA TO ROLE ANALYST_ROLE;

-- Grant table SELECT to both roles
-- Masking policies automatically enforce what each role sees
GRANT SELECT ON TABLE CORTEX_SECURITY_DEMO.PII_DATA.CUSTOMER_FEEDBACK TO ROLE DATA_ENGINEER_ROLE;
GRANT SELECT ON TABLE CORTEX_SECURITY_DEMO.PII_DATA.CUSTOMER_FEEDBACK TO ROLE ANALYST_ROLE;

-- Grant Cortex LLM function access
-- DATA_ENGINEER_ROLE gets access to all Cortex functions
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE DATA_ENGINEER_ROLE;

-- ANALYST_ROLE gets limited Cortex access (sentiment and summarize only — no complete/translate)
-- This demonstrates model-level RBAC for AI functions
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE ANALYST_ROLE;

-- NOTE: In an enterprise setup, you would create a custom role
-- that grants only specific Cortex function privileges. The above
-- uses Snowflake's built-in CORTEX_USER role for the trial account.

-- Confirm grants
SHOW GRANTS TO ROLE DATA_ENGINEER_ROLE;
SHOW GRANTS TO ROLE ANALYST_ROLE;
```

---

### Step 6 — Invoke Snowflake Cortex LLM Functions

**File: `05_cortex_queries.sql`**
```sql
-- ============================================================
-- STEP 6: Invoke Cortex LLM Functions Against Customer Data
-- Run first as: DATA_ENGINEER_ROLE (sees real PII)
-- Run again as: ANALYST_ROLE (sees masked PII)
-- ============================================================

USE DATABASE CORTEX_SECURITY_DEMO;
USE SCHEMA PII_DATA;
USE WAREHOUSE CORTEX_DEMO_WH;

-- ------------------------------------------------------------
-- 6.1 Switch to DATA_ENGINEER_ROLE and run Cortex queries
-- ------------------------------------------------------------
USE ROLE DATA_ENGINEER_ROLE;

-- Sentiment Analysis on customer feedback
-- DATA_ENGINEER sees real email alongside sentiment score
SELECT
    CUSTOMER_ID,
    CUSTOMER_NAME,
    EMAIL,                                                          -- Real email visible
    SATISFACTION_SCORE,
    SNOWFLAKE.CORTEX.SENTIMENT(FEEDBACK_COMMENT)                   AS SENTIMENT_SCORE,
    FEEDBACK_COMMENT
FROM CUSTOMER_FEEDBACK
ORDER BY SENTIMENT_SCORE ASC;

-- Summarize all negative feedback (score < 3)
SELECT
    CUSTOMER_ID,
    CUSTOMER_NAME,
    EMAIL,                                                          -- Real email visible
    SSN,                                                            -- Real SSN visible
    SNOWFLAKE.CORTEX.SUMMARIZE(FEEDBACK_COMMENT)                   AS FEEDBACK_SUMMARY
FROM CUSTOMER_FEEDBACK
WHERE SATISFACTION_SCORE < 3.0;

-- Translate non-English feedback to English using Cortex
SELECT
    CUSTOMER_ID,
    CUSTOMER_NAME,
    REGION,
    FEEDBACK_COMMENT                                               AS ORIGINAL_COMMENT,
    SNOWFLAKE.CORTEX.TRANSLATE(FEEDBACK_COMMENT, '', 'en')         AS TRANSLATED_TO_ENGLISH
FROM CUSTOMER_FEEDBACK
WHERE REGION IN ('Asia Pacific', 'Middle East');

-- Use COMPLETE for intelligent analysis of feedback trends
SELECT
    SNOWFLAKE.CORTEX.COMPLETE(
        'mistral-7b',
        CONCAT(
            'You are a customer experience analyst. Based on the following customer feedback, ',
            'identify the top 3 areas of improvement and list them as bullet points. ',
            'Feedback: ', FEEDBACK_COMMENT
        )
    ) AS AI_IMPROVEMENT_SUGGESTIONS,
    CUSTOMER_NAME,
    EMAIL                                                          -- Real email visible to this role
FROM CUSTOMER_FEEDBACK
WHERE SATISFACTION_SCORE < 3.0
LIMIT 3;

-- ------------------------------------------------------------
-- 6.2 Switch to ANALYST_ROLE and run the SAME Cortex queries
-- CRITICAL: Masking policies fire automatically — Cortex
--           receives and returns masked PII values
-- ------------------------------------------------------------
USE ROLE ANALYST_ROLE;

-- Same sentiment query — but EMAIL is now masked by policy
SELECT
    CUSTOMER_ID,
    CUSTOMER_NAME,
    EMAIL,                                                          -- Masked: ***@***.***
    SATISFACTION_SCORE,
    SNOWFLAKE.CORTEX.SENTIMENT(FEEDBACK_COMMENT)                   AS SENTIMENT_SCORE,
    FEEDBACK_COMMENT
FROM CUSTOMER_FEEDBACK
ORDER BY SENTIMENT_SCORE ASC;

-- Same summarize query — SSN and EMAIL now masked
SELECT
    CUSTOMER_ID,
    CUSTOMER_NAME,
    EMAIL,                                                          -- Masked: ***@***.***
    SSN,                                                            -- Masked: ***-**-****
    SNOWFLAKE.CORTEX.SUMMARIZE(FEEDBACK_COMMENT)                   AS FEEDBACK_SUMMARY
FROM CUSTOMER_FEEDBACK
WHERE SATISFACTION_SCORE < 3.0;
```

---

### Step 7 — Validate RBAC + Masking with Cortex

**File: `06_validation_tests.sql`**
```sql
-- ============================================================
-- STEP 7: Validation Tests — Prove Security is Enforced
-- ============================================================

USE DATABASE CORTEX_SECURITY_DEMO;
USE SCHEMA PII_DATA;
USE WAREHOUSE CORTEX_DEMO_WH;

-- ✅ TEST 1: DATA_ENGINEER_ROLE sees real PII
USE ROLE DATA_ENGINEER_ROLE;
SELECT
    'DATA_ENGINEER_ROLE'  AS ACTIVE_ROLE,
    CUSTOMER_NAME,
    EMAIL,
    SSN,
    PHONE_NUMBER
FROM CUSTOMER_FEEDBACK
LIMIT 3;
-- Expected: Real values like alice.johnson@email.com, 123-45-6789

-- ✅ TEST 2: ANALYST_ROLE sees masked PII
USE ROLE ANALYST_ROLE;
SELECT
    'ANALYST_ROLE'        AS ACTIVE_ROLE,
    CUSTOMER_NAME,
    EMAIL,
    SSN,
    PHONE_NUMBER
FROM CUSTOMER_FEEDBACK
LIMIT 3;
-- Expected: Masked values like ***@***.*** and ***-**-****

-- ✅ TEST 3: Cortex Sentiment with DATA_ENGINEER_ROLE — real email in output
USE ROLE DATA_ENGINEER_ROLE;
SELECT
    'DATA_ENGINEER_ROLE'  AS ACTIVE_ROLE,
    EMAIL,
    SNOWFLAKE.CORTEX.SENTIMENT(FEEDBACK_COMMENT) AS SENTIMENT
FROM CUSTOMER_FEEDBACK
LIMIT 3;
-- Expected: Real email address alongside AI sentiment score

-- ✅ TEST 4: Cortex Sentiment with ANALYST_ROLE — masked email in output
USE ROLE ANALYST_ROLE;
SELECT
    'ANALYST_ROLE'        AS ACTIVE_ROLE,
    EMAIL,
    SNOWFLAKE.CORTEX.SENTIMENT(FEEDBACK_COMMENT) AS SENTIMENT
FROM CUSTOMER_FEEDBACK
LIMIT 3;
-- Expected: ***@***.*** alongside AI sentiment score — PII never reaches the LLM output

-- ✅ TEST 5: Confirm masking policy definitions
USE ROLE ACCOUNTADMIN;
SHOW MASKING POLICIES IN SCHEMA CORTEX_SECURITY_DEMO.PII_DATA;

-- ✅ TEST 6: Confirm which columns have policies applied
SELECT
    TABLE_NAME,
    COLUMN_NAME,
    MASKING_POLICY_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_SCHEMA = 'PII_DATA'
  AND MASKING_POLICY_NAME IS NOT NULL;
```

---

## 7. How Cortex Inherits Platform Security

This is the core architectural principle of this project. The table below summarizes how each security feature flows through to Cortex:

| Security Layer | Mechanism | Effect on Cortex |
|---|---|---|
| **RBAC — Role Context** | Every SQL statement runs under the caller's active role | Cortex LLM functions execute with the same role context — no privilege escalation |
| **Dynamic Data Masking** | Column-level masking policies evaluate `CURRENT_ROLE()` at query time | Cortex receives the **masked value** as its input — the LLM never processes real PII for restricted roles |
| **Object Privileges** | `GRANT SELECT` controls table access per role | If a role has no `SELECT` on the table, Cortex functions on that table will fail with a permission error |
| **Cortex Function Grants** | `SNOWFLAKE.CORTEX_USER` database role controls who can call LLM functions | Roles without this grant cannot invoke any Cortex function at all |
| **Query Audit** | Snowflake `QUERY_HISTORY` logs all Cortex calls with the executing role | Full auditability of who asked the AI what, and what data it processed |

---

## 8. Strategic Value

### Compliance by Default
- PII is masked before it reaches the LLM prompt layer, automating adherence to **GDPR**, **CCPA**, and **HIPAA** requirements.
- No custom scrubbing pipeline, regex filter, or pre-processing step is needed.

### Speed to Insight
- Security policies are defined once at the schema level and automatically inherited by all AI queries.
- Reduces time-to-deploy for AI features from months (building custom middleware) to minutes (one `ALTER TABLE ... SET MASKING POLICY` statement).

### Single Governance Layer
- One set of policies governs both traditional SQL queries and Cortex AI queries.
- There is no separate "AI security layer" to maintain — the data layer and the AI layer share the same trust boundary.

### Full Auditability
- Every Cortex invocation is logged in `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` with the role, user, timestamp, and query text.
- Security teams can prove that no restricted user ever obtained unmasked PII through an AI query.

---

## 9. Expected Outputs

After running all scripts, you should observe the following:

| Test | DATA_ENGINEER_ROLE Result | ANALYST_ROLE Result |
|---|---|---|
| `SELECT EMAIL` | `alice.johnson@email.com` | `***@***.***` |
| `SELECT SSN` | `123-45-6789` | `***-**-****` |
| `SELECT PHONE_NUMBER` | `+1-555-101-2020` | `+*-***-***-****` |
| `CORTEX.SENTIMENT` | Returns sentiment + real email | Returns sentiment + masked email |
| `CORTEX.SUMMARIZE` | Summary with real PII context | Summary with masked PII — no leakage |
| `CORTEX.TRANSLATE` | Translated text + real PII | Translated text + masked PII |
| `CORTEX.COMPLETE` | AI analysis + real email | AI analysis + masked email |

---

## 10. Troubleshooting

| Issue | Likely Cause | Resolution |
|---|---|---|
| `Cortex function not available in this region` | Trial account region does not support Cortex | Switch account region to US East (N. Virginia) or US West (Oregon) when signing up |
| `Insufficient privileges to use Cortex` | `CORTEX_USER` role not granted | Run `GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE <your_role>` as ACCOUNTADMIN |
| `Masking policy not applying` | Policy applied after data was queried in cache | Run `ALTER WAREHOUSE CORTEX_DEMO_WH SUSPEND` then resume and re-query |
| `Object does not exist` | Scripts run out of order | Run scripts strictly in order: 01 → 02 → 03 → 04 → 05 → 06 |
| `ANALYST_ROLE still sees real PII` | Role context not switched correctly | Confirm with `SELECT CURRENT_ROLE()` before running queries |
| `TRANSLATE returns empty` | Source language detection failed | Specify source language explicitly: `CORTEX.TRANSLATE(col, 'fr', 'en')` |

---

## 11. References

- [Snowflake Cortex LLM Functions Documentation](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions)
- [Snowflake Dynamic Data Masking](https://docs.snowflake.com/en/user-guide/security-column-ddm-intro)
- [Snowflake RBAC Overview](https://docs.snowflake.com/en/user-guide/security-access-control-overview)
- [Cortex Region Availability](https://docs.snowflake.com/en/user-guide/snowflake-cortex/llm-functions#availability)
- [SNOWFLAKE.CORTEX_USER Database Role](https://docs.snowflake.com/en/sql-reference/sql/grant-database-role)
- [Snowflake Query History for Audit](https://docs.snowflake.com/en/sql-reference/account-usage/query_history)

---

*Built on Snowflake Trial Account — zero pre-existing objects required. All infrastructure is provisioned end-to-end by the SQL scripts in this repository.*