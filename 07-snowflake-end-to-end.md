# 7. End-to-End Guide: dbt + Snowflake Data Warehouse

> A comprehensive, hands-on guide covering every step from Snowflake account setup through production deployment. Designed as a complete reference for building a real dbt project on Snowflake.

---

## Table of Contents

1. [Snowflake Fundamentals for dbt Engineers](#71-snowflake-fundamentals-for-dbt-engineers)
2. [Environment Setup & Installation](#72-environment-setup--installation)
3. [Snowflake Account Configuration](#73-snowflake-account-configuration)
4. [dbt Project Initialization & Connection](#74-dbt-project-initialization--connection)
5. [Source Data Setup](#75-source-data-setup)
6. [Building the Staging Layer](#76-building-the-staging-layer)
7. [Building the Intermediate Layer](#77-building-the-intermediate-layer)
8. [Building the Mart Layer](#78-building-the-mart-layer)
9. [Testing Strategy](#79-testing-strategy)
10. [Snapshots for SCD Type 2](#710-snapshots-for-scd-type-2)
11. [Seeds & Reference Data](#711-seeds--reference-data)
12. [Macros & Jinja Patterns for Snowflake](#712-macros--jinja-patterns-for-snowflake)
13. [Documentation & Lineage](#713-documentation--lineage)
14. [Snowflake-Specific Optimizations](#714-snowflake-specific-optimizations)
15. [Environment Management (Dev/Staging/Prod)](#715-environment-management-devstagingprod)
16. [CI/CD Pipeline](#716-cicd-pipeline)
17. [Orchestration](#717-orchestration)
18. [Monitoring & Observability](#718-monitoring--observability)
19. [Security & Governance](#719-security--governance)
20. [Cost Management](#720-cost-management)
21. [Troubleshooting Common Issues](#721-troubleshooting-common-issues)
22. [Production Checklist](#722-production-checklist)

---

## 7.1 Snowflake Fundamentals for dbt Engineers

Before wiring up dbt, you need to understand Snowflake's architecture — it directly impacts how you configure dbt and optimize your models.

### Snowflake's Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   CLOUD SERVICES LAYER                   │
│  Authentication, metadata, query parsing & optimization  │
│  Access control, infrastructure management               │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                    COMPUTE LAYER                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ Warehouse │  │ Warehouse │  │ Warehouse │  ...        │
│  │   (XS)    │  │   (M)     │  │   (L)     │             │
│  │ dbt_dev   │  │ dbt_ci    │  │ dbt_prod  │             │
│  └──────────┘  └──────────┘  └──────────┘              │
│  Independent, auto-scaling, auto-suspend                 │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│                    STORAGE LAYER                         │
│  Centralized, compressed columnar storage (micro-parts)  │
│  Automatic clustering, zero-copy cloning                 │
│  Time Travel (1-90 days), Fail-safe (7 days)             │
└─────────────────────────────────────────────────────────┘
```

### Key Concepts That Affect dbt

| Snowflake Concept | Impact on dbt | What You Configure |
|-------------------|---------------|-------------------|
| **Virtual Warehouse** | Compute engine that runs your dbt queries | `warehouse` in `profiles.yml` — size it per environment |
| **Database** | Top-level container for schemas | `database` in `profiles.yml` — separate per environment |
| **Schema** | Container for tables/views within a database | `schema` in `profiles.yml` + `generate_schema_name` macro |
| **Role** | RBAC access control | `role` in `profiles.yml` — different roles per environment |
| **Auto-suspend** | Warehouse pauses after idle period | Set to 60s for dev, 300s for prod (saves credits) |
| **Auto-resume** | Warehouse starts on query arrival | Always enabled — dbt queries trigger resume |
| **Clustering** | Physical data organization for query performance | `cluster_by` in model config |
| **Zero-copy clone** | Instant, free copy of a database/schema | Used for CI environments |
| **Time Travel** | Query historical data states | Safety net for dbt mistakes |

### Snowflake Object Hierarchy

```
Account
├── Database: RAW              (landing zone — EL tools write here)
│   ├── Schema: ECOMMERCE
│   │   ├── Table: ORDERS
│   │   ├── Table: CUSTOMERS
│   │   └── Table: PRODUCTS
│   └── Schema: PAYMENTS
│       └── Table: TRANSACTIONS
├── Database: ANALYTICS        (dbt writes here — prod)
│   ├── Schema: STAGING
│   ├── Schema: INTERMEDIATE
│   ├── Schema: MARTS
│   └── Schema: SNAPSHOTS
├── Database: ANALYTICS_DEV    (dbt writes here — dev)
│   └── Schema: DBT_<USERNAME>
└── Database: ANALYTICS_CI     (dbt writes here — CI)
    └── Schema: PR_<NUMBER>
```

---

## 7.2 Environment Setup & Installation

### Prerequisites

- Python 3.9+ installed
- A Snowflake account (trial works fine for learning)
- Git installed
- A code editor (VS Code recommended with the dbt Power User extension)

### Install dbt-snowflake

```bash
# Create a virtual environment (recommended)
python -m venv dbt-env
source dbt-env/bin/activate        # Linux/Mac
# dbt-env\Scripts\activate         # Windows

# Install dbt with the Snowflake adapter
pip install dbt-snowflake

# Verify installation
dbt --version
```

Expected output:
```
Core:
  - installed: 1.9.x
  - latest:    1.9.x

Plugins:
  - snowflake: 1.9.x
```

### Install Supporting Tools

```bash
# SQLFluff for SQL linting (optional but recommended)
pip install sqlfluff sqlfluff-templater-dbt

# pre-commit for Git hooks (optional)
pip install pre-commit
```

---

## 7.3 Snowflake Account Configuration

Before dbt can connect, you need to set up the Snowflake objects. Run these SQL statements in the Snowflake UI (Snowsight) or via SnowSQL.

### Create Databases

```sql
-- Landing zone for raw data (EL tools write here)
CREATE DATABASE IF NOT EXISTS RAW;

-- Production analytics (dbt writes here)
CREATE DATABASE IF NOT EXISTS ANALYTICS;

-- Development analytics
CREATE DATABASE IF NOT EXISTS ANALYTICS_DEV;

-- CI/CD analytics
CREATE DATABASE IF NOT EXISTS ANALYTICS_CI;
```

### Create Warehouses

```sql
-- Development: extra-small, auto-suspend after 60 seconds
CREATE WAREHOUSE IF NOT EXISTS DBT_DEV
    WAREHOUSE_SIZE = 'X-SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'dbt development warehouse';

-- CI/CD: small, auto-suspend after 120 seconds
CREATE WAREHOUSE IF NOT EXISTS DBT_CI
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'dbt CI/CD warehouse';

-- Production: medium, auto-suspend after 300 seconds
CREATE WAREHOUSE IF NOT EXISTS DBT_PROD
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 300
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE
    COMMENT = 'dbt production warehouse';
```

### Create Roles & Users

```sql
-- Create roles
CREATE ROLE IF NOT EXISTS DBT_DEV_ROLE;
CREATE ROLE IF NOT EXISTS DBT_CI_ROLE;
CREATE ROLE IF NOT EXISTS DBT_PROD_ROLE;
CREATE ROLE IF NOT EXISTS ANALYST_ROLE;

-- Grant role hierarchy
GRANT ROLE DBT_DEV_ROLE TO ROLE SYSADMIN;
GRANT ROLE DBT_CI_ROLE TO ROLE SYSADMIN;
GRANT ROLE DBT_PROD_ROLE TO ROLE SYSADMIN;
GRANT ROLE ANALYST_ROLE TO ROLE SYSADMIN;

-- Create service account for dbt (production)
CREATE USER IF NOT EXISTS DBT_PROD_USER
    PASSWORD = '<strong_password>'
    DEFAULT_ROLE = DBT_PROD_ROLE
    DEFAULT_WAREHOUSE = DBT_PROD
    COMMENT = 'dbt production service account';

GRANT ROLE DBT_PROD_ROLE TO USER DBT_PROD_USER;

-- Create service account for CI
CREATE USER IF NOT EXISTS DBT_CI_USER
    PASSWORD = '<strong_password>'
    DEFAULT_ROLE = DBT_CI_ROLE
    DEFAULT_WAREHOUSE = DBT_CI
    COMMENT = 'dbt CI service account';

GRANT ROLE DBT_CI_ROLE TO USER DBT_CI_USER;
```

### Grant Permissions

```sql
-- Dev role: read RAW, write ANALYTICS_DEV
GRANT USAGE ON DATABASE RAW TO ROLE DBT_DEV_ROLE;
GRANT USAGE ON ALL SCHEMAS IN DATABASE RAW TO ROLE DBT_DEV_ROLE;
GRANT SELECT ON ALL TABLES IN DATABASE RAW TO ROLE DBT_DEV_ROLE;
GRANT SELECT ON FUTURE TABLES IN DATABASE RAW TO ROLE DBT_DEV_ROLE;

GRANT ALL ON DATABASE ANALYTICS_DEV TO ROLE DBT_DEV_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ANALYTICS_DEV TO ROLE DBT_DEV_ROLE;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE ANALYTICS_DEV TO ROLE DBT_DEV_ROLE;

GRANT USAGE ON WAREHOUSE DBT_DEV TO ROLE DBT_DEV_ROLE;

-- CI role: read RAW, write ANALYTICS_CI
GRANT USAGE ON DATABASE RAW TO ROLE DBT_CI_ROLE;
GRANT USAGE ON ALL SCHEMAS IN DATABASE RAW TO ROLE DBT_CI_ROLE;
GRANT SELECT ON ALL TABLES IN DATABASE RAW TO ROLE DBT_CI_ROLE;
GRANT SELECT ON FUTURE TABLES IN DATABASE RAW TO ROLE DBT_CI_ROLE;

GRANT ALL ON DATABASE ANALYTICS_CI TO ROLE DBT_CI_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ANALYTICS_CI TO ROLE DBT_CI_ROLE;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE ANALYTICS_CI TO ROLE DBT_CI_ROLE;

GRANT USAGE ON WAREHOUSE DBT_CI TO ROLE DBT_CI_ROLE;

-- Prod role: read RAW, write ANALYTICS
GRANT USAGE ON DATABASE RAW TO ROLE DBT_PROD_ROLE;
GRANT USAGE ON ALL SCHEMAS IN DATABASE RAW TO ROLE DBT_PROD_ROLE;
GRANT SELECT ON ALL TABLES IN DATABASE RAW TO ROLE DBT_PROD_ROLE;
GRANT SELECT ON FUTURE TABLES IN DATABASE RAW TO ROLE DBT_PROD_ROLE;

GRANT ALL ON DATABASE ANALYTICS TO ROLE DBT_PROD_ROLE;
GRANT ALL ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE DBT_PROD_ROLE;
GRANT ALL ON FUTURE SCHEMAS IN DATABASE ANALYTICS TO ROLE DBT_PROD_ROLE;

GRANT USAGE ON WAREHOUSE DBT_PROD TO ROLE DBT_PROD_ROLE;

-- Analyst role: read-only on ANALYTICS
GRANT USAGE ON DATABASE ANALYTICS TO ROLE ANALYST_ROLE;
GRANT USAGE ON ALL SCHEMAS IN DATABASE ANALYTICS TO ROLE ANALYST_ROLE;
GRANT SELECT ON ALL TABLES IN DATABASE ANALYTICS TO ROLE ANALYST_ROLE;
GRANT SELECT ON ALL VIEWS IN DATABASE ANALYTICS TO ROLE ANALYST_ROLE;
GRANT SELECT ON FUTURE TABLES IN DATABASE ANALYTICS TO ROLE ANALYST_ROLE;
GRANT SELECT ON FUTURE VIEWS IN DATABASE ANALYTICS TO ROLE ANALYST_ROLE;

GRANT USAGE ON WAREHOUSE DBT_DEV TO ROLE ANALYST_ROLE;

-- Grant your personal user the dev role
GRANT ROLE DBT_DEV_ROLE TO USER <YOUR_USERNAME>;
```

### Key-Pair Authentication (Recommended for Production)

Password auth works for dev, but production should use key-pair authentication:

```bash
# Generate RSA key pair
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt
openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub

# Extract the public key (strip headers)
grep -v "BEGIN\|END" rsa_key.pub | tr -d '\n'
```

```sql
-- Assign public key to user in Snowflake
ALTER USER DBT_PROD_USER SET RSA_PUBLIC_KEY='MIIBIjANBg...';
```

---

## 7.4 dbt Project Initialization & Connection

### Initialize the Project

```bash
# Create a new dbt project
dbt init snowflake_analytics

# Follow the prompts:
# - Which database would you like to use? snowflake
# - account: xy12345.us-east-1
# - user: your_username
# - authentication method: password (for dev)
# - password: your_password
# - role: DBT_DEV_ROLE
# - warehouse: DBT_DEV
# - database: ANALYTICS_DEV
# - schema: dbt_<your_name>
# - threads: 4
```

This creates two things:
1. A project directory `snowflake_analytics/` with the standard dbt structure
2. A `~/.dbt/profiles.yml` entry for the connection

### Configure `profiles.yml`

Edit `~/.dbt/profiles.yml` for a complete multi-environment setup:

```yaml
snowflake_analytics:
  target: dev

  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('SNOWFLAKE_USER') }}"
      password: "{{ env_var('SNOWFLAKE_PASSWORD') }}"
      role: DBT_DEV_ROLE
      database: ANALYTICS_DEV
      warehouse: DBT_DEV
      schema: "dbt_{{ env_var('SNOWFLAKE_USER') | lower }}"
      threads: 4
      query_tag: "dbt_dev"
      # Snowflake-specific connection options
      client_session_keep_alive: true
      connect_retries: 3
      connect_timeout: 30
      retry_on_database_errors: true
      retry_all: false

    ci:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: "{{ env_var('DBT_CI_USER') }}"
      password: "{{ env_var('DBT_CI_PASSWORD') }}"
      role: DBT_CI_ROLE
      database: ANALYTICS_CI
      warehouse: DBT_CI
      schema: "pr_{{ env_var('CI_MERGE_REQUEST_IID', 'local') }}"
      threads: 8
      query_tag: "dbt_ci"

    prod:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      user: DBT_PROD_USER
      # Key-pair auth for production
      private_key_path: "{{ env_var('DBT_PROD_KEY_PATH') }}"
      private_key_passphrase: "{{ env_var('DBT_PROD_KEY_PASSPHRASE', '') }}"
      role: DBT_PROD_ROLE
      database: ANALYTICS
      warehouse: DBT_PROD
      schema: ANALYTICS
      threads: 16
      query_tag: "dbt_prod"
```

### Set Environment Variables

```bash
# Add to your shell profile (~/.bashrc, ~/.zshrc, etc.)
export SNOWFLAKE_ACCOUNT="xy12345.us-east-1"
export SNOWFLAKE_USER="your_username"
export SNOWFLAKE_PASSWORD="your_password"
```

### Test the Connection

```bash
cd snowflake_analytics
dbt debug
```

Expected output:
```
  Connection test: [OK connection ok]
  All checks passed!
```

### Configure `dbt_project.yml`

```yaml
name: 'snowflake_analytics'
version: '1.0.0'
config-version: 2

profile: 'snowflake_analytics'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets: ["target", "dbt_packages"]

vars:
  start_date: '2023-01-01'

models:
  snowflake_analytics:
    +materialized: view
    staging:
      +materialized: view
      +schema: staging
      +tags: ['staging']
    intermediate:
      +materialized: ephemeral
      +tags: ['intermediate']
    marts:
      +materialized: table
      +schema: marts
      +tags: ['marts']

seeds:
  snowflake_analytics:
    +schema: seeds

snapshots:
  snowflake_analytics:
    +target_schema: snapshots

tests:
  snowflake_analytics:
    +severity: error
    +store_failures: true
    +schema: test_results
```

### Install Packages

```yaml
# packages.yml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]

  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<0.11.0"]

  - package: dbt-labs/codegen
    version: [">=0.12.0", "<0.13.0"]

  - package: elementary-data/elementary
    version: [">=0.13.0", "<0.14.0"]
```

```bash
dbt deps
```

---

## 7.5 Source Data Setup

For this guide, we'll create sample raw data in Snowflake to work with.

### Create Raw Schema & Tables

```sql
USE ROLE SYSADMIN;
USE DATABASE RAW;

CREATE SCHEMA IF NOT EXISTS ECOMMERCE;
USE SCHEMA ECOMMERCE;

-- Orders table
CREATE OR REPLACE TABLE ORDERS (
    ID NUMBER AUTOINCREMENT,
    CUSTOMER_ID NUMBER,
    PRODUCT_ID NUMBER,
    AMOUNT_CENTS NUMBER,
    TAX_CENTS NUMBER,
    STATUS VARCHAR(20),
    SHIPPING_METHOD VARCHAR(30),
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    UPDATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Customers table
CREATE OR REPLACE TABLE CUSTOMERS (
    ID NUMBER AUTOINCREMENT,
    NAME VARCHAR(200),
    EMAIL VARCHAR(200),
    COUNTRY_CODE VARCHAR(3),
    PLAN_TYPE VARCHAR(20) DEFAULT 'free',
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    UPDATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Products table
CREATE OR REPLACE TABLE PRODUCTS (
    ID NUMBER AUTOINCREMENT,
    NAME VARCHAR(200),
    CATEGORY VARCHAR(100),
    UNIT_COST_CENTS NUMBER,
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Payments table
CREATE OR REPLACE TABLE PAYMENTS (
    ID NUMBER AUTOINCREMENT,
    ORDER_ID NUMBER,
    PAYMENT_METHOD VARCHAR(30),
    AMOUNT_CENTS NUMBER,
    STATUS VARCHAR(20),
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Events table (for incremental model demo)
CREATE OR REPLACE TABLE EVENTS (
    EVENT_ID VARCHAR(36) DEFAULT UUID_STRING(),
    USER_ID NUMBER,
    EVENT_TYPE VARCHAR(50),
    PAGE_URL VARCHAR(500),
    EVENT_PAYLOAD VARIANT,
    OCCURRED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _LOADED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Insert Sample Data

```sql
-- Customers
INSERT INTO CUSTOMERS (NAME, EMAIL, COUNTRY_CODE, PLAN_TYPE, CREATED_AT, UPDATED_AT)
VALUES
    ('Alice Johnson', 'alice@example.com', 'US', 'premium', '2023-01-15', '2024-06-01'),
    ('Bob Smith', 'bob@example.com', 'GB', 'free', '2023-03-22', '2023-03-22'),
    ('Carlos Ruiz', 'carlos@example.com', 'ES', 'premium', '2023-05-10', '2024-01-15'),
    ('Diana Chen', 'diana@example.com', 'SG', 'enterprise', '2023-07-01', '2024-03-20'),
    ('Eva Mueller', 'eva@example.com', 'DE', 'free', '2023-09-14', '2023-09-14'),
    ('Frank Tanaka', 'frank@example.com', 'JP', 'premium', '2023-11-30', '2024-05-10'),
    ('Grace Kim', 'grace@example.com', 'KR', 'enterprise', '2024-01-05', '2024-07-01'),
    ('Hassan Ali', 'hassan@example.com', 'AE', 'free', '2024-02-18', '2024-02-18'),
    ('Iris Patel', 'iris@example.com', 'IN', 'premium', '2024-04-01', '2024-08-15'),
    ('Jack Wilson', 'jack@example.com', 'AU', 'free', '2024-05-20', '2024-05-20');

-- Products
INSERT INTO PRODUCTS (NAME, CATEGORY, UNIT_COST_CENTS, CREATED_AT)
VALUES
    ('Wireless Mouse', 'Electronics', 1500, '2023-01-01'),
    ('Mechanical Keyboard', 'Electronics', 4500, '2023-01-01'),
    ('USB-C Hub', 'Electronics', 2500, '2023-02-15'),
    ('Desk Lamp', 'Office', 3000, '2023-03-01'),
    ('Notebook Set', 'Office', 800, '2023-03-01'),
    ('Coffee Mug', 'Lifestyle', 500, '2023-04-01'),
    ('Standing Desk Mat', 'Office', 3500, '2023-05-01'),
    ('Webcam HD', 'Electronics', 5000, '2023-06-01'),
    ('Headphone Stand', 'Accessories', 1200, '2023-07-01'),
    ('Cable Organizer', 'Accessories', 600, '2023-08-01');

-- Orders (spread across dates)
INSERT INTO ORDERS (CUSTOMER_ID, PRODUCT_ID, AMOUNT_CENTS, TAX_CENTS, STATUS, SHIPPING_METHOD, CREATED_AT, UPDATED_AT)
SELECT
    UNIFORM(1, 10, RANDOM()) AS CUSTOMER_ID,
    UNIFORM(1, 10, RANDOM()) AS PRODUCT_ID,
    UNIFORM(500, 100000, RANDOM()) AS AMOUNT_CENTS,
    UNIFORM(50, 10000, RANDOM()) AS TAX_CENTS,
    CASE UNIFORM(1, 5, RANDOM())
        WHEN 1 THEN 'pending'
        WHEN 2 THEN 'processing'
        WHEN 3 THEN 'shipped'
        WHEN 4 THEN 'delivered'
        WHEN 5 THEN 'cancelled'
    END AS STATUS,
    CASE UNIFORM(1, 3, RANDOM())
        WHEN 1 THEN 'standard'
        WHEN 2 THEN 'express'
        WHEN 3 THEN 'overnight'
    END AS SHIPPING_METHOD,
    DATEADD('day', -UNIFORM(0, 730, RANDOM()), CURRENT_TIMESTAMP()) AS CREATED_AT,
    DATEADD('day', -UNIFORM(0, 730, RANDOM()), CURRENT_TIMESTAMP()) AS UPDATED_AT
FROM TABLE(GENERATOR(ROWCOUNT => 500));

-- Payments
INSERT INTO PAYMENTS (ORDER_ID, PAYMENT_METHOD, AMOUNT_CENTS, STATUS, CREATED_AT)
SELECT
    o.ID,
    CASE UNIFORM(1, 4, RANDOM())
        WHEN 1 THEN 'credit_card'
        WHEN 2 THEN 'bank_transfer'
        WHEN 3 THEN 'paypal'
        WHEN 4 THEN 'crypto'
    END,
    o.AMOUNT_CENTS + o.TAX_CENTS,
    CASE WHEN o.STATUS = 'cancelled' THEN 'refunded' ELSE 'completed' END,
    o.CREATED_AT
FROM ORDERS o;

-- Events
INSERT INTO EVENTS (USER_ID, EVENT_TYPE, PAGE_URL, EVENT_PAYLOAD, OCCURRED_AT)
SELECT
    UNIFORM(1, 10, RANDOM()),
    CASE UNIFORM(1, 5, RANDOM())
        WHEN 1 THEN 'page_view'
        WHEN 2 THEN 'add_to_cart'
        WHEN 3 THEN 'checkout_start'
        WHEN 4 THEN 'purchase'
        WHEN 5 THEN 'search'
    END,
    CASE UNIFORM(1, 4, RANDOM())
        WHEN 1 THEN '/products'
        WHEN 2 THEN '/cart'
        WHEN 3 THEN '/checkout'
        WHEN 4 THEN '/home'
    END,
    OBJECT_CONSTRUCT('session_id', UUID_STRING(), 'device', 
        CASE UNIFORM(1,3,RANDOM()) WHEN 1 THEN 'mobile' WHEN 2 THEN 'desktop' ELSE 'tablet' END),
    DATEADD('minute', -UNIFORM(0, 525600, RANDOM()), CURRENT_TIMESTAMP())
FROM TABLE(GENERATOR(ROWCOUNT => 5000));
```

### Define Sources in dbt

```yaml
# models/staging/ecommerce/_ecommerce__sources.yml
version: 2

sources:
  - name: ecommerce
    description: "Raw e-commerce data loaded by Fivetran/Airbyte"
    database: RAW
    schema: ECOMMERCE
    loader: fivetran
    loaded_at_field: _LOADED_AT

    freshness:
      warn_after: { count: 12, period: hour }
      error_after: { count: 24, period: hour }

    tables:
      - name: orders
        description: "Raw orders from the e-commerce platform"
        columns:
          - name: ID
            description: "Primary key"
            tests:
              - unique
              - not_null

      - name: customers
        description: "Raw customer records"
        columns:
          - name: ID
            description: "Primary key"
            tests:
              - unique
              - not_null

      - name: products
        description: "Product catalog"
        columns:
          - name: ID
            description: "Primary key"
            tests:
              - unique
              - not_null

      - name: payments
        description: "Payment transactions"
        columns:
          - name: ID
            description: "Primary key"
            tests:
              - unique
              - not_null

      - name: events
        description: "Clickstream events"
        freshness:
          warn_after: { count: 1, period: hour }
          error_after: { count: 6, period: hour }
        columns:
          - name: EVENT_ID
            description: "Primary key (UUID)"
            tests:
              - unique
              - not_null
```

Verify sources:
```bash
dbt source freshness
```

---

## 7.6 Building the Staging Layer

Staging models are the clean interface between raw data and your business logic. One model per source table, minimal transformation.

### Project Structure

```
models/
└── staging/
    └── ecommerce/
        ├── _ecommerce__sources.yml      (source definitions — created above)
        ├── _ecommerce__models.yml       (model properties & tests)
        ├── stg_ecommerce__orders.sql
        ├── stg_ecommerce__customers.sql
        ├── stg_ecommerce__products.sql
        ├── stg_ecommerce__payments.sql
        └── stg_ecommerce__events.sql
```

### Staging Models

```sql
-- models/staging/ecommerce/stg_ecommerce__orders.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'orders') }}
),

renamed AS (
    SELECT
        -- Primary key
        id AS order_id,

        -- Foreign keys
        customer_id,
        product_id,

        -- Measures (convert cents to dollars)
        ROUND(amount_cents / 100.0, 2) AS order_amount,
        ROUND(tax_cents / 100.0, 2) AS tax_amount,
        ROUND((amount_cents + tax_cents) / 100.0, 2) AS total_amount,

        -- Dimensions
        LOWER(TRIM(status)) AS order_status,
        LOWER(TRIM(shipping_method)) AS shipping_method,

        -- Timestamps
        created_at AS ordered_at,
        updated_at,

        -- Metadata
        _loaded_at AS _etl_loaded_at
    FROM source
)

SELECT * FROM renamed
```

```sql
-- models/staging/ecommerce/stg_ecommerce__customers.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'customers') }}
),

renamed AS (
    SELECT
        id AS customer_id,
        TRIM(name) AS customer_name,
        LOWER(TRIM(email)) AS email,
        UPPER(TRIM(country_code)) AS country_code,
        LOWER(TRIM(plan_type)) AS plan_type,
        created_at AS signed_up_at,
        updated_at,
        _loaded_at AS _etl_loaded_at
    FROM source
)

SELECT * FROM renamed
```

```sql
-- models/staging/ecommerce/stg_ecommerce__products.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'products') }}
),

renamed AS (
    SELECT
        id AS product_id,
        TRIM(name) AS product_name,
        LOWER(TRIM(category)) AS product_category,
        ROUND(unit_cost_cents / 100.0, 2) AS unit_cost,
        created_at AS product_created_at,
        _loaded_at AS _etl_loaded_at
    FROM source
)

SELECT * FROM renamed
```

```sql
-- models/staging/ecommerce/stg_ecommerce__payments.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'payments') }}
),

renamed AS (
    SELECT
        id AS payment_id,
        order_id,
        LOWER(TRIM(payment_method)) AS payment_method,
        ROUND(amount_cents / 100.0, 2) AS payment_amount,
        LOWER(TRIM(status)) AS payment_status,
        created_at AS paid_at,
        _loaded_at AS _etl_loaded_at
    FROM source
)

SELECT * FROM renamed
```

```sql
-- models/staging/ecommerce/stg_ecommerce__events.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('ecommerce', 'events') }}
),

renamed AS (
    SELECT
        event_id,
        user_id,
        LOWER(TRIM(event_type)) AS event_type,
        page_url,
        event_payload,
        -- Extract fields from VARIANT (Snowflake semi-structured)
        event_payload:session_id::VARCHAR AS session_id,
        event_payload:device::VARCHAR AS device_type,
        occurred_at,
        _loaded_at AS _etl_loaded_at
    FROM source
)

SELECT * FROM renamed
```

### Staging Model Properties & Tests

```yaml
# models/staging/ecommerce/_ecommerce__models.yml
version: 2

models:
  - name: stg_ecommerce__orders
    description: "Cleaned orders from the e-commerce platform. One row per order."
    columns:
      - name: order_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: customer_id
        description: "FK to customers"
        tests:
          - not_null
          - relationships:
              to: ref('stg_ecommerce__customers')
              field: customer_id
      - name: order_status
        description: "Current order status"
        tests:
          - accepted_values:
              values: ['pending', 'processing', 'shipped', 'delivered', 'cancelled']
      - name: order_amount
        tests:
          - not_null

  - name: stg_ecommerce__customers
    description: "Cleaned customer records. One row per customer."
    columns:
      - name: customer_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: email
        tests:
          - unique
          - not_null
      - name: plan_type
        tests:
          - accepted_values:
              values: ['free', 'premium', 'enterprise']

  - name: stg_ecommerce__products
    description: "Product catalog. One row per product."
    columns:
      - name: product_id
        description: "Primary key"
        tests:
          - unique
          - not_null

  - name: stg_ecommerce__payments
    description: "Payment transactions. One row per payment."
    columns:
      - name: payment_id
        description: "Primary key"
        tests:
          - unique
          - not_null
      - name: order_id
        tests:
          - not_null
          - relationships:
              to: ref('stg_ecommerce__orders')
              field: order_id
      - name: payment_method
        tests:
          - accepted_values:
              values: ['credit_card', 'bank_transfer', 'paypal', 'crypto']

  - name: stg_ecommerce__events
    description: "Clickstream events. One row per event."
    columns:
      - name: event_id
        description: "Primary key (UUID)"
        tests:
          - unique
          - not_null
      - name: event_type
        tests:
          - accepted_values:
              values: ['page_view', 'add_to_cart', 'checkout_start', 'purchase', 'search']
```

### Build & Test Staging

```bash
# Build all staging models
dbt build --select staging

# Check compiled SQL
cat target/compiled/snowflake_analytics/models/staging/ecommerce/stg_ecommerce__orders.sql
```

---

## 7.7 Building the Intermediate Layer

Intermediate models contain business logic — joins, aggregations, and derived calculations.

```sql
-- models/intermediate/int_orders_enriched.sql
{{ config(materialized='ephemeral') }}

SELECT
    o.order_id,
    o.customer_id,
    o.product_id,
    o.ordered_at,
    o.order_status,
    o.shipping_method,
    o.order_amount,
    o.tax_amount,
    o.total_amount,

    -- Product enrichment
    p.product_name,
    p.product_category,
    p.unit_cost,

    -- Derived: gross margin
    o.order_amount - p.unit_cost AS gross_margin,

    -- Derived: order tier
    CASE
        WHEN o.order_amount >= 500 THEN 'premium'
        WHEN o.order_amount >= 100 THEN 'standard'
        ELSE 'basic'
    END AS order_tier,

    -- Derived: fulfillment status
    CASE
        WHEN o.order_status IN ('delivered') THEN 'fulfilled'
        WHEN o.order_status IN ('shipped', 'processing') THEN 'in_progress'
        WHEN o.order_status = 'pending' THEN 'awaiting'
        WHEN o.order_status = 'cancelled' THEN 'cancelled'
        ELSE 'unknown'
    END AS fulfillment_status,

    -- Time dimensions
    DATE_TRUNC('day', o.ordered_at) AS order_date,
    DATE_TRUNC('week', o.ordered_at) AS order_week,
    DATE_TRUNC('month', o.ordered_at) AS order_month

FROM {{ ref('stg_ecommerce__orders') }} AS o
LEFT JOIN {{ ref('stg_ecommerce__products') }} AS p
    ON o.product_id = p.product_id
```

```sql
-- models/intermediate/int_customer_order_summary.sql
{{ config(materialized='ephemeral') }}

SELECT
    c.customer_id,
    c.customer_name,
    c.email,
    c.country_code,
    c.plan_type,
    c.signed_up_at,

    -- Order aggregations
    COUNT(o.order_id) AS total_orders,
    COUNT(CASE WHEN o.order_status = 'delivered' THEN 1 END) AS delivered_orders,
    COUNT(CASE WHEN o.order_status = 'cancelled' THEN 1 END) AS cancelled_orders,

    COALESCE(SUM(o.total_amount), 0) AS lifetime_value,
    COALESCE(AVG(o.total_amount), 0) AS avg_order_value,

    MIN(o.ordered_at) AS first_order_at,
    MAX(o.ordered_at) AS last_order_at,

    DATEDIFF('day', MAX(o.ordered_at), CURRENT_TIMESTAMP()) AS days_since_last_order,
    DATEDIFF('day', c.signed_up_at, MIN(o.ordered_at)) AS days_to_first_order

FROM {{ ref('stg_ecommerce__customers') }} AS c
LEFT JOIN {{ ref('stg_ecommerce__orders') }} AS o
    ON c.customer_id = o.customer_id
GROUP BY
    c.customer_id,
    c.customer_name,
    c.email,
    c.country_code,
    c.plan_type,
    c.signed_up_at
```

```sql
-- models/intermediate/int_payment_methods_pivoted.sql
{{ config(materialized='ephemeral') }}

SELECT
    order_id,
    SUM(CASE WHEN payment_method = 'credit_card' THEN payment_amount ELSE 0 END) AS credit_card_amount,
    SUM(CASE WHEN payment_method = 'bank_transfer' THEN payment_amount ELSE 0 END) AS bank_transfer_amount,
    SUM(CASE WHEN payment_method = 'paypal' THEN payment_amount ELSE 0 END) AS paypal_amount,
    SUM(CASE WHEN payment_method = 'crypto' THEN payment_amount ELSE 0 END) AS crypto_amount,
    SUM(payment_amount) AS total_paid,
    COUNT(*) AS payment_count,
    MAX(paid_at) AS last_payment_at
FROM {{ ref('stg_ecommerce__payments') }}
WHERE payment_status = 'completed'
GROUP BY order_id
```

---

## 7.8 Building the Mart Layer

Mart models are consumption-ready — these are what BI tools and analysts query.

### Dimension: Customers

```sql
-- models/marts/dim_customers.sql
{{ config(
    materialized='table',
    tags=['daily', 'core']
) }}

WITH customer_summary AS (
    SELECT * FROM {{ ref('int_customer_order_summary') }}
),

country_names AS (
    SELECT * FROM {{ ref('seed_country_codes') }}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['cs.customer_id']) }} AS customer_sk,
    cs.customer_id,
    cs.customer_name,
    cs.email,
    cs.country_code,
    cn.country_name,
    cn.region,
    cs.plan_type,
    cs.signed_up_at,

    -- Order metrics
    cs.total_orders,
    cs.delivered_orders,
    cs.cancelled_orders,
    cs.lifetime_value,
    cs.avg_order_value,
    cs.first_order_at,
    cs.last_order_at,
    cs.days_since_last_order,
    cs.days_to_first_order,

    -- Customer segmentation
    CASE
        WHEN cs.total_orders = 0 THEN 'prospect'
        WHEN cs.days_since_last_order <= 30 THEN 'active'
        WHEN cs.days_since_last_order <= 90 THEN 'at_risk'
        WHEN cs.days_since_last_order <= 365 THEN 'lapsed'
        ELSE 'churned'
    END AS customer_segment,

    -- Value tier
    CASE
        WHEN cs.lifetime_value >= 5000 THEN 'platinum'
        WHEN cs.lifetime_value >= 1000 THEN 'gold'
        WHEN cs.lifetime_value >= 200 THEN 'silver'
        ELSE 'bronze'
    END AS value_tier,

    -- Metadata
    CURRENT_TIMESTAMP() AS _dbt_updated_at

FROM customer_summary AS cs
LEFT JOIN country_names AS cn
    ON cs.country_code = cn.country_code
```

### Fact: Orders

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_sk',
    incremental_strategy='merge',
    cluster_by=['order_date'],
    tags=['daily', 'core']
) }}

WITH orders AS (
    SELECT * FROM {{ ref('int_orders_enriched') }}
),

payments AS (
    SELECT * FROM {{ ref('int_payment_methods_pivoted') }}
)

SELECT
    {{ dbt_utils.generate_surrogate_key(['o.order_id']) }} AS order_sk,
    o.order_id,
    o.customer_id,
    o.product_id,

    -- Timestamps
    o.ordered_at,
    o.order_date,
    o.order_week,
    o.order_month,

    -- Order attributes
    o.order_status,
    o.fulfillment_status,
    o.shipping_method,
    o.order_tier,
    o.product_name,
    o.product_category,

    -- Financial measures
    o.order_amount,
    o.tax_amount,
    o.total_amount,
    o.unit_cost,
    o.gross_margin,

    -- Payment breakdown
    COALESCE(p.credit_card_amount, 0) AS credit_card_amount,
    COALESCE(p.bank_transfer_amount, 0) AS bank_transfer_amount,
    COALESCE(p.paypal_amount, 0) AS paypal_amount,
    COALESCE(p.crypto_amount, 0) AS crypto_amount,
    COALESCE(p.total_paid, 0) AS total_paid,
    COALESCE(p.payment_count, 0) AS payment_count,

    -- Derived
    CASE
        WHEN COALESCE(p.total_paid, 0) >= o.total_amount THEN TRUE
        ELSE FALSE
    END AS is_fully_paid,

    -- Metadata
    CURRENT_TIMESTAMP() AS _dbt_updated_at

FROM orders AS o
LEFT JOIN payments AS p
    ON o.order_id = p.order_id

{% if is_incremental() %}
WHERE o.ordered_at > (
    SELECT DATEADD('hour', -6, MAX(ordered_at))
    FROM {{ this }}
)
{% endif %}
```

### Fact: Daily Revenue

```sql
-- models/marts/fct_daily_revenue.sql
{{ config(
    materialized='table',
    tags=['daily', 'finance']
) }}

SELECT
    order_date,
    order_month,

    -- Volume metrics
    COUNT(DISTINCT order_id) AS total_orders,
    COUNT(DISTINCT customer_id) AS unique_customers,

    -- Revenue metrics
    SUM(order_amount) AS gross_revenue,
    SUM(CASE WHEN order_status != 'cancelled' THEN order_amount ELSE 0 END) AS net_revenue,
    SUM(tax_amount) AS total_tax,
    SUM(total_amount) AS total_revenue_with_tax,
    SUM(gross_margin) AS total_gross_margin,

    -- Averages
    AVG(order_amount) AS avg_order_value,

    -- Status breakdown
    COUNT(CASE WHEN order_status = 'delivered' THEN 1 END) AS delivered_orders,
    COUNT(CASE WHEN order_status = 'cancelled' THEN 1 END) AS cancelled_orders,
    COUNT(CASE WHEN order_status = 'pending' THEN 1 END) AS pending_orders,

    -- Payment method breakdown
    SUM(credit_card_amount) AS credit_card_revenue,
    SUM(bank_transfer_amount) AS bank_transfer_revenue,
    SUM(paypal_amount) AS paypal_revenue,
    SUM(crypto_amount) AS crypto_revenue,

    -- Category breakdown
    COUNT(DISTINCT product_category) AS unique_categories,

    -- Metadata
    CURRENT_TIMESTAMP() AS _dbt_updated_at

FROM {{ ref('fct_orders') }}
GROUP BY order_date, order_month
ORDER BY order_date
```

### Fact: Events (Incremental with Micro-Batch Pattern)

```sql
-- models/marts/fct_events.sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    cluster_by=['occurred_date'],
    tags=['hourly', 'product']
) }}

SELECT
    event_id,
    user_id,
    event_type,
    page_url,
    session_id,
    device_type,
    occurred_at,
    DATE_TRUNC('day', occurred_at) AS occurred_date,
    DATE_TRUNC('hour', occurred_at) AS occurred_hour,

    -- Session sequencing
    ROW_NUMBER() OVER (
        PARTITION BY session_id
        ORDER BY occurred_at
    ) AS event_sequence_in_session,

    -- Time since previous event (same user)
    DATEDIFF('second',
        LAG(occurred_at) OVER (PARTITION BY user_id ORDER BY occurred_at),
        occurred_at
    ) AS seconds_since_last_event,

    CURRENT_TIMESTAMP() AS _dbt_updated_at

FROM {{ ref('stg_ecommerce__events') }}

{% if is_incremental() %}
WHERE occurred_at > (
    SELECT DATEADD('hour', -3, MAX(occurred_at))
    FROM {{ this }}
)
{% endif %}
```

### Build the Marts

```bash
# Build everything in DAG order
dbt build

# Or build just the marts
dbt build --select marts
```

---

## 7.9 Testing Strategy

### Mart-Level Tests

```yaml
# models/marts/_marts__models.yml
version: 2

models:
  - name: dim_customers
    description: "Customer dimension with segmentation and lifetime metrics"
    config:
      contract:
        enforced: true
    columns:
      - name: customer_sk
        data_type: varchar
        description: "Surrogate key"
        tests:
          - unique
          - not_null
      - name: customer_id
        data_type: number
        tests:
          - unique
          - not_null
      - name: customer_segment
        data_type: varchar
        tests:
          - accepted_values:
              values: ['prospect', 'active', 'at_risk', 'lapsed', 'churned']
      - name: value_tier
        data_type: varchar
        tests:
          - accepted_values:
              values: ['platinum', 'gold', 'silver', 'bronze']
      - name: lifetime_value
        data_type: number
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              row_condition: "total_orders > 0"

  - name: fct_orders
    description: "Order fact table with payment details"
    columns:
      - name: order_sk
        tests:
          - unique
          - not_null
      - name: order_id
        tests:
          - unique
          - not_null
      - name: customer_id
        tests:
          - not_null
          - relationships:
              to: ref('dim_customers')
              field: customer_id
      - name: order_amount
        tests:
          - not_null
      - name: total_amount
        tests:
          - not_null

  - name: fct_daily_revenue
    description: "Daily revenue aggregation"
    columns:
      - name: order_date
        tests:
          - unique
          - not_null
      - name: total_orders
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
      - name: net_revenue
        tests:
          - not_null

  - name: fct_events
    description: "Clickstream events fact table"
    columns:
      - name: event_id
        tests:
          - unique
          - not_null
```

### Singular Tests (Business Rule Validation)

```sql
-- tests/singular/assert_revenue_reconciles_with_source.sql
-- Ensures mart revenue matches source data within tolerance

WITH mart_total AS (
    SELECT SUM(order_amount) AS total
    FROM {{ ref('fct_orders') }}
    WHERE order_status != 'cancelled'
),

source_total AS (
    SELECT SUM(AMOUNT_CENTS / 100.0) AS total
    FROM {{ source('ecommerce', 'orders') }}
    WHERE LOWER(STATUS) != 'cancelled'
)

SELECT
    m.total AS mart_total,
    s.total AS source_total,
    ABS(m.total - s.total) AS difference
FROM mart_total m
CROSS JOIN source_total s
WHERE ABS(m.total - s.total) > 1.00  -- $1 tolerance for rounding
```

```sql
-- tests/singular/assert_no_orphaned_orders.sql
-- Every order should have a corresponding customer

SELECT o.order_id, o.customer_id
FROM {{ ref('fct_orders') }} o
LEFT JOIN {{ ref('dim_customers') }} c
    ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL
```

```sql
-- tests/singular/assert_daily_revenue_not_negative.sql

SELECT order_date, net_revenue
FROM {{ ref('fct_daily_revenue') }}
WHERE net_revenue < 0
```

### Run Tests

```bash
# Run all tests
dbt test

# Run tests for a specific model
dbt test --select fct_orders

# Run only singular tests
dbt test --select test_type:singular

# Store test failures in the warehouse for investigation
dbt test --store-failures
```

---

## 7.10 Snapshots for SCD Type 2

Track how customer records change over time:

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}

{{ config(
    target_database='ANALYTICS',
    target_schema='SNAPSHOTS',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=True
) }}

SELECT
    id AS customer_id,
    name,
    email,
    country_code,
    plan_type,
    updated_at
FROM {{ source('ecommerce', 'customers') }}

{% endsnapshot %}
```

```bash
# Run snapshots
dbt snapshot

# Verify: query the snapshot table
# SELECT * FROM ANALYTICS.SNAPSHOTS.SNAP_CUSTOMERS WHERE customer_id = 1 ORDER BY dbt_valid_from;
```

---

## 7.11 Seeds & Reference Data

```csv
country_code,country_name,region
US,United States,North America
CA,Canada,North America
GB,United Kingdom,Europe
DE,Germany,Europe
FR,France,Europe
ES,Spain,Europe
JP,Japan,Asia Pacific
SG,Singapore,Asia Pacific
AU,Australia,Asia Pacific
IN,India,Asia Pacific
KR,South Korea,Asia Pacific
AE,United Arab Emirates,Middle East
BR,Brazil,South America
```

Save as `seeds/seed_country_codes.csv`.

```yaml
# dbt_project.yml (add under seeds section)
seeds:
  snowflake_analytics:
    +schema: seeds
    seed_country_codes:
      +column_types:
        country_code: varchar(3)
        country_name: varchar(100)
        region: varchar(50)
```

```bash
dbt seed
```

---

## 7.12 Macros & Jinja Patterns for Snowflake

### Snowflake-Specific Schema Name Override

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- elif target.name == 'prod' -%}
        {{ custom_schema_name | trim | upper }}
    {%- else -%}
        {{ default_schema | upper }}_{{ custom_schema_name | trim | upper }}
    {%- endif -%}
{%- endmacro %}
```

This ensures:
- Prod: `ANALYTICS.STAGING`, `ANALYTICS.MARTS`
- Dev: `ANALYTICS_DEV.DBT_ALICE_STAGING`, `ANALYTICS_DEV.DBT_ALICE_MARTS`

### Cents to Dollars Macro

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name, precision=2) %}
    ROUND(CAST({{ column_name }} AS NUMBER(18, {{ precision }})) / 100.0, {{ precision }})
{% endmacro %}
```

### Safe Divide

```sql
-- macros/safe_divide.sql
{% macro safe_divide(numerator, denominator, default=0) %}
    CASE
        WHEN {{ denominator }} IS NULL OR {{ denominator }} = 0
        THEN {{ default }}
        ELSE CAST({{ numerator }} AS NUMBER(38, 6)) / NULLIF({{ denominator }}, 0)
    END
{% endmacro %}
```

### Grant Permissions Post-Hook

```sql
-- macros/grant_select.sql
{% macro grant_select_to_role(role='ANALYST_ROLE') %}
    GRANT SELECT ON {{ this }} TO ROLE {{ role }};
{% endmacro %}
```

Usage in `dbt_project.yml`:
```yaml
models:
  snowflake_analytics:
    marts:
      +post-hook:
        - "{{ grant_select_to_role('ANALYST_ROLE') }}"
```

### Snowflake VARIANT Extraction Macro

```sql
-- macros/extract_json.sql
{% macro extract_json(column, path, as_type='VARCHAR') %}
    {{ column }}:{{ path }}::{{ as_type }}
{% endmacro %}
```

Usage:
```sql
SELECT
    {{ extract_json('event_payload', 'session_id') }} AS session_id,
    {{ extract_json('event_payload', 'device', 'VARCHAR') }} AS device_type,
    {{ extract_json('event_payload', 'duration', 'NUMBER') }} AS duration_ms
FROM {{ ref('stg_ecommerce__events') }}
```

### Dynamic Warehouse Sizing

```sql
-- macros/use_warehouse.sql
{% macro use_warehouse(warehouse_name) %}
    {% set sql %}
        USE WAREHOUSE {{ warehouse_name }};
    {% endset %}
    {% do run_query(sql) %}
{% endmacro %}
```

Usage as a pre-hook for heavy models:
```sql
{{ config(
    materialized='table',
    pre_hook="{{ use_warehouse('DBT_PROD_LARGE') }}",
    post_hook="{{ use_warehouse('DBT_PROD') }}"
) }}
```

---

## 7.13 Documentation & Lineage

### Generate and Serve Docs

```bash
# Generate documentation artifacts
dbt docs generate

# Serve locally (opens browser)
dbt docs serve --port 8080
```

### Add Exposures

```yaml
# models/exposures.yml
version: 2

exposures:
  - name: executive_revenue_dashboard
    type: dashboard
    maturity: high
    url: https://looker.company.com/dashboards/revenue
    description: "Weekly executive revenue review dashboard"
    depends_on:
      - ref('fct_daily_revenue')
      - ref('dim_customers')
    owner:
      name: Analytics Team
      email: analytics@company.com
    tags: ['tier-1', 'executive']

  - name: customer_segmentation_report
    type: analysis
    maturity: medium
    url: https://hex.company.com/notebooks/customer-segments
    description: "Monthly customer segmentation analysis"
    depends_on:
      - ref('dim_customers')
      - ref('fct_orders')
    owner:
      name: Marketing Analytics
      email: marketing-analytics@company.com

  - name: product_recommendation_model
    type: ml
    maturity: low
    description: "ML model for product recommendations"
    depends_on:
      - ref('fct_orders')
      - ref('fct_events')
    owner:
      name: ML Engineering
      email: ml-eng@company.com
```

---

## 7.14 Snowflake-Specific Optimizations

### Clustering Keys

Snowflake automatically clusters data in micro-partitions, but for large tables (>1TB), explicit clustering improves query performance:

```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_sk',
    incremental_strategy='merge',
    cluster_by=['order_date', 'customer_id']
) }}
```

When to cluster:
- Tables > 1TB
- Queries consistently filter on specific columns
- High cardinality columns used in WHERE/JOIN clauses

When NOT to cluster:
- Small tables (Snowflake's automatic clustering is sufficient)
- Tables with unpredictable query patterns
- Clustering costs credits for maintenance — don't over-cluster

### Transient Tables

Transient tables skip Fail-safe (7-day recovery), reducing storage costs. Good for intermediate/staging tables that can be rebuilt:

```sql
{{ config(
    materialized='table',
    transient=true
) }}
```

### Query Tags

Tag all dbt queries for cost attribution and monitoring:

```yaml
# profiles.yml
prod:
  type: snowflake
  query_tag: "dbt_prod"
  # ...
```

Or per-model:
```sql
{{ config(
    query_tag='dbt_mart_finance'
) }}
```

Query tags appear in `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY`, enabling cost-per-model analysis.

### Copy Grants

Preserve existing grants when dbt rebuilds a table:

```sql
{{ config(
    materialized='table',
    copy_grants=true
) }}
```

Without this, every `dbt run` would drop and recreate the table, losing any manually applied grants.

### Snowflake-Specific Incremental Strategies

Snowflake supports `merge` (default), `append`, and `delete+insert`:

```sql
-- Merge (default, best for most cases)
{{ config(
    materialized='incremental',
    incremental_strategy='merge',
    unique_key='order_id'
) }}

-- Delete+Insert (can be faster for large batches)
{{ config(
    materialized='incremental',
    incremental_strategy='delete+insert',
    unique_key='order_id'
) }}

-- Append (no dedup, fastest)
{{ config(
    materialized='incremental',
    incremental_strategy='append'
) }}
```

### Warehouse Size Recommendations

| Use Case | Warehouse Size | Auto-Suspend | Threads |
|----------|---------------|-------------|---------|
| Dev (single developer) | X-Small | 60s | 4 |
| CI/CD | Small | 120s | 8 |
| Prod (< 100 models) | Medium | 300s | 16 |
| Prod (100-500 models) | Large | 300s | 24 |
| Prod (heavy incrementals) | X-Large | 300s | 32 |

### Multi-Cluster Warehouses for Production

For production workloads with concurrent queries:

```sql
ALTER WAREHOUSE DBT_PROD SET
    MIN_CLUSTER_COUNT = 1
    MAX_CLUSTER_COUNT = 3
    SCALING_POLICY = 'STANDARD';
```

This auto-scales compute when dbt runs overlap with analyst queries.

---

## 7.15 Environment Management (Dev/Staging/Prod)

### Environment Isolation Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    ENVIRONMENT LAYOUT                         │
│                                                              │
│  DEV (per-developer)                                         │
│  ├── Database: ANALYTICS_DEV                                 │
│  ├── Schema:   DBT_ALICE_STAGING, DBT_ALICE_MARTS            │
│  ├── Warehouse: DBT_DEV (X-Small)                            │
│  └── Role:     DBT_DEV_ROLE                                  │
│                                                              │
│  CI (per-PR, ephemeral)                                      │
│  ├── Database: ANALYTICS_CI                                  │
│  ├── Schema:   PR_123_STAGING, PR_123_MARTS                  │
│  ├── Warehouse: DBT_CI (Small)                               │
│  └── Role:     DBT_CI_ROLE                                   │
│                                                              │
│  PROD                                                        │
│  ├── Database: ANALYTICS                                     │
│  ├── Schema:   STAGING, MARTS, SNAPSHOTS                     │
│  ├── Warehouse: DBT_PROD (Medium)                            │
│  └── Role:     DBT_PROD_ROLE                                 │
└─────────────────────────────────────────────────────────────┘
```

### Zero-Copy Cloning for CI

Instead of building from scratch, clone the production database for CI:

```sql
-- In CI pipeline (before dbt build)
CREATE OR REPLACE DATABASE ANALYTICS_CI CLONE ANALYTICS;
```

This is instant and free (no additional storage until data diverges). dbt then runs only modified models against the clone.

### Switching Environments

```bash
# Development (default)
dbt build

# CI
dbt build --target ci

# Production
dbt build --target prod
```

---

## 7.16 CI/CD Pipeline

### GitHub Actions Example

```yaml
# .github/workflows/dbt-ci.yml
name: dbt CI

on:
  pull_request:
    branches: [main]
    paths:
      - 'models/**'
      - 'macros/**'
      - 'tests/**'
      - 'snapshots/**'
      - 'seeds/**'
      - 'dbt_project.yml'
      - 'packages.yml'

env:
  SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
  DBT_CI_USER: ${{ secrets.DBT_CI_USER }}
  DBT_CI_PASSWORD: ${{ secrets.DBT_CI_PASSWORD }}
  CI_MERGE_REQUEST_IID: ${{ github.event.pull_request.number }}

jobs:
  dbt-ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install dbt-snowflake sqlfluff sqlfluff-templater-dbt

      - name: Install dbt packages
        run: dbt deps

      - name: Lint SQL
        run: sqlfluff lint models/ --dialect snowflake
        continue-on-error: true  # Don't block on lint warnings

      - name: Download production manifest
        uses: actions/download-artifact@v4
        with:
          name: prod-manifest
          path: ./prod-artifacts/
        continue-on-error: true  # First run won't have artifacts

      - name: dbt build (Slim CI)
        run: |
          dbt build \
            --target ci \
            --select state:modified+ \
            --defer \
            --state ./prod-artifacts/ \
            --fail-fast

      - name: Upload CI artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: ci-artifacts
          path: target/

      - name: Cleanup CI schema
        if: always()
        run: |
          dbt run-operation drop_ci_schema --target ci
```

### Production Deployment

```yaml
# .github/workflows/dbt-prod.yml
name: dbt Production Deploy

on:
  push:
    branches: [main]
    paths:
      - 'models/**'
      - 'macros/**'
      - 'tests/**'
      - 'snapshots/**'
      - 'seeds/**'
      - 'dbt_project.yml'
      - 'packages.yml'

env:
  SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
  DBT_PROD_KEY_PATH: /tmp/rsa_key.p8

jobs:
  dbt-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install dbt-snowflake

      - name: Write private key
        run: echo "${{ secrets.DBT_PROD_PRIVATE_KEY }}" > /tmp/rsa_key.p8

      - name: Install dbt packages
        run: dbt deps

      - name: Check source freshness
        run: dbt source freshness --target prod

      - name: dbt build (production)
        run: |
          dbt build --target prod --fail-fast

      - name: Generate docs
        run: dbt docs generate --target prod

      - name: Upload production manifest
        uses: actions/upload-artifact@v4
        with:
          name: prod-manifest
          path: |
            target/manifest.json
            target/run_results.json

      - name: Cleanup private key
        if: always()
        run: rm -f /tmp/rsa_key.p8
```

### CI Cleanup Macro

```sql
-- macros/drop_ci_schema.sql
{% macro drop_ci_schema() %}
    {% set schemas = ['PR_' ~ env_var('CI_MERGE_REQUEST_IID', 'unknown') ~ '_STAGING',
                      'PR_' ~ env_var('CI_MERGE_REQUEST_IID', 'unknown') ~ '_MARTS'] %}
    {% for schema in schemas %}
        {% set sql %}
            DROP SCHEMA IF EXISTS {{ target.database }}.{{ schema }} CASCADE;
        {% endset %}
        {% do run_query(sql) %}
        {{ log("Dropped schema: " ~ schema, info=True) }}
    {% endfor %}
{% endmacro %}
```

---

## 7.17 Orchestration

### Airflow + dbt (Cosmos)

The `astronomer-cosmos` package renders each dbt model as an Airflow task, giving you per-model visibility:

```python
# dags/dbt_daily.py
from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import SnowflakeUserPasswordProfileMapping

profile_config = ProfileConfig(
    profile_name="snowflake_analytics",
    target_name="prod",
    profile_mapping=SnowflakeUserPasswordProfileMapping(
        conn_id="snowflake_default",
        profile_args={
            "database": "ANALYTICS",
            "schema": "ANALYTICS",
            "warehouse": "DBT_PROD",
            "role": "DBT_PROD_ROLE",
        },
    ),
)

dbt_dag = DbtDag(
    project_config=ProjectConfig("/opt/dbt/snowflake_analytics"),
    profile_config=profile_config,
    execution_config=ExecutionConfig(dbt_executable_path="/opt/dbt-env/bin/dbt"),
    schedule_interval="0 6 * * *",  # Daily at 6 AM UTC
    start_date=datetime(2024, 1, 1),
    catchup=False,
    dag_id="dbt_daily_build",
    default_args={"retries": 1, "retry_delay": timedelta(minutes=5)},
)
```

### Dagster + dbt

```python
# dagster_project/assets.py
from dagster import Definitions
from dagster_dbt import DbtCliResource, dbt_assets

dbt = DbtCliResource(
    project_dir="/opt/dbt/snowflake_analytics",
    target="prod",
)

@dbt_assets(manifest="/opt/dbt/snowflake_analytics/target/manifest.json")
def snowflake_dbt_assets(context, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()

defs = Definitions(
    assets=[snowflake_dbt_assets],
    resources={"dbt": dbt},
)
```

### Simple Cron-Based (for smaller teams)

```bash
#!/bin/bash
# scripts/dbt_daily.sh

set -euo pipefail

cd /opt/dbt/snowflake_analytics

# Activate virtual environment
source /opt/dbt-env/bin/activate

# Install/update packages
dbt deps

# Check source freshness
dbt source freshness --target prod || echo "WARNING: Source freshness check failed"

# Run the build
dbt build --target prod --fail-fast 2>&1 | tee /var/log/dbt/daily_$(date +%Y%m%d).log

# Generate docs
dbt docs generate --target prod

# Alert on failure
if [ $? -ne 0 ]; then
    curl -X POST "$SLACK_WEBHOOK_URL" \
        -H 'Content-type: application/json' \
        -d '{"text":"dbt daily build FAILED. Check logs."}'
fi
```

```cron
# crontab -e
0 6 * * * /opt/dbt/scripts/dbt_daily.sh
```

---

## 7.18 Monitoring & Observability

### Snowflake Query History for dbt

```sql
-- Find all dbt queries and their cost
SELECT
    query_tag,
    query_type,
    database_name,
    schema_name,
    execution_status,
    total_elapsed_time / 1000 AS elapsed_seconds,
    bytes_scanned / (1024*1024*1024) AS gb_scanned,
    credits_used_cloud_services,
    start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_tag LIKE 'dbt_%'
    AND start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 50;
```

### dbt Run Results Analysis

```sql
-- Create a table to store historical run results
CREATE TABLE IF NOT EXISTS ANALYTICS.MONITORING.DBT_RUN_RESULTS (
    invocation_id VARCHAR,
    unique_id VARCHAR,
    status VARCHAR,
    execution_time FLOAT,
    rows_affected NUMBER,
    run_started_at TIMESTAMP_NTZ,
    loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Load from run_results.json (via a Python script or Snowpipe)
```

### Elementary dbt Package

After installing Elementary (see packages.yml above):

```bash
# Run Elementary models to populate monitoring tables
dbt run --select elementary

# Generate the Elementary report
edr report --profile-target prod
```

Elementary provides:
- Test result history and trends
- Model execution time tracking
- Anomaly detection on row counts and schema changes
- A self-hosted HTML dashboard

### Slack Alerting on Failures

```sql
-- macros/alert_on_failure.sql
{% macro alert_on_failure(results) %}
    {% if execute %}
        {% set failures = results | selectattr("status", "equalto", "error") | list %}
        {% if failures | length > 0 %}
            {% set message = "dbt build had " ~ failures | length ~ " failures: " %}
            {% for f in failures %}
                {% set message = message ~ f.node.unique_id ~ " " %}
            {% endfor %}
            {{ log(message, info=True) }}
            {# In practice, call a webhook or use dbt Cloud's built-in alerting #}
        {% endif %}
    {% endif %}
{% endmacro %}
```

---

## 7.19 Security & Governance

### Row-Level Security with Snowflake

```sql
-- macros/apply_row_access_policy.sql
{% macro apply_row_access_policy(policy_name, column_name) %}
    {% if target.name == 'prod' %}
        ALTER TABLE {{ this }} ADD ROW ACCESS POLICY {{ policy_name }} ON ({{ column_name }});
    {% endif %}
{% endmacro %}
```

### Column-Level Masking

```sql
-- In Snowflake (one-time setup)
CREATE OR REPLACE MASKING POLICY email_mask AS (val STRING)
RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('DBT_PROD_ROLE', 'SYSADMIN') THEN val
        ELSE REGEXP_REPLACE(val, '.+@', '***@')
    END;
```

```sql
-- Apply via dbt post-hook
{{ config(
    post_hook="ALTER TABLE {{ this }} MODIFY COLUMN email SET MASKING POLICY email_mask"
) }}
```

### Tagging for Governance

```sql
-- macros/apply_tags.sql
{% macro apply_governance_tags(pii_columns=[]) %}
    {% if target.name == 'prod' %}
        {% for col in pii_columns %}
            ALTER TABLE {{ this }} MODIFY COLUMN {{ col }}
                SET TAG governance.pii = 'true';
        {% endfor %}
    {% endif %}
{% endmacro %}
```

Usage:
```sql
{{ config(
    post_hook="{{ apply_governance_tags(pii_columns=['email', 'customer_name']) }}"
) }}
```

### Network Policies

Restrict dbt connections to specific IPs:

```sql
CREATE NETWORK POLICY dbt_access_policy
    ALLOWED_IP_LIST = ('10.0.0.0/8', '172.16.0.0/12')  -- Your CI/CD runners
    BLOCKED_IP_LIST = ();

ALTER USER DBT_PROD_USER SET NETWORK_POLICY = dbt_access_policy;
ALTER USER DBT_CI_USER SET NETWORK_POLICY = dbt_access_policy;
```

---

## 7.20 Cost Management

### Credit Consumption Monitoring

```sql
-- Daily credit usage by dbt warehouse
SELECT
    warehouse_name,
    DATE_TRUNC('day', start_time) AS usage_date,
    SUM(credits_used) AS total_credits,
    SUM(credits_used) * 3.00 AS estimated_cost_usd  -- Adjust rate per your contract
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE warehouse_name LIKE 'DBT_%'
    AND start_time >= DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 2 DESC, 3 DESC;
```

### Cost Optimization Strategies

| Strategy | Impact | Implementation |
|----------|--------|---------------|
| Right-size warehouses | High | Start small, scale up based on query times |
| Auto-suspend aggressively | High | 60s for dev, 120s for CI, 300s for prod |
| Use transient tables | Medium | `transient=true` for staging/intermediate tables |
| Incremental models | High | Avoid full table rebuilds for large fact tables |
| Slim CI | High | Only build changed models in PRs |
| Cluster only large tables | Medium | Don't cluster tables < 1TB |
| Schedule off-peak | Low | Run prod builds during off-peak hours for lower queuing |
| Resource monitors | Safety | Set credit limits to prevent runaway costs |

### Resource Monitors

```sql
-- Create a resource monitor for dbt warehouses
CREATE RESOURCE MONITOR dbt_monitor
    WITH CREDIT_QUOTA = 1000  -- Monthly limit
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 75 PERCENT DO NOTIFY
        ON 90 PERCENT DO NOTIFY
        ON 100 PERCENT DO SUSPEND;

-- Apply to warehouses
ALTER WAREHOUSE DBT_PROD SET RESOURCE_MONITOR = dbt_monitor;
ALTER WAREHOUSE DBT_CI SET RESOURCE_MONITOR = dbt_monitor;
ALTER WAREHOUSE DBT_DEV SET RESOURCE_MONITOR = dbt_monitor;
```

---

## 7.21 Troubleshooting Common Issues

### Connection Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `250001: Could not connect to Snowflake` | Wrong account identifier | Use format `xy12345.us-east-1` (include region) |
| `Authentication failed` | Wrong credentials or role | Verify user/password, check role exists and is granted |
| `Warehouse 'X' does not exist` | Warehouse not created or no access | Create warehouse, grant USAGE to role |
| `Database 'X' does not exist` | Database not created or no access | Create database, grant access to role |
| `SSL: CERTIFICATE_VERIFY_FAILED` | Corporate proxy/firewall | Set `insecure_mode: true` in profiles.yml (dev only!) |

### Build Issues

| Error | Cause | Fix |
|-------|-------|-----|
| `Compilation Error: 'ref' not found` | Model name typo or missing model | Check model file exists and name matches |
| `Compilation Error: circular dependency` | Model A refs B, B refs A | Restructure to break the cycle |
| `Runtime Error: Object does not exist` | Source table missing or wrong schema | Verify source config matches actual Snowflake objects |
| `Incremental model full-refreshing unexpectedly` | Table was dropped or schema changed | Check `on_schema_change` config |
| `Permission denied` | Role lacks required grants | Run GRANT statements from Section 7.3 |

### Performance Issues

| Symptom | Diagnosis | Fix |
|---------|-----------|-----|
| Slow incremental model | Large lookback window or no clustering | Reduce lookback, add `cluster_by` |
| Warehouse queuing | Too many concurrent queries | Increase warehouse size or use multi-cluster |
| Slow view queries | View chains (view → view → view) | Materialize intermediate steps as tables |
| High credit usage | Oversized warehouse or too-frequent runs | Right-size warehouse, optimize schedule |

### Debugging Workflow

```bash
# 1. Check compiled SQL
dbt compile --select my_model
cat target/compiled/snowflake_analytics/models/marts/my_model.sql

# 2. Run the compiled SQL manually in Snowflake to see the error
# Copy from target/compiled/ and paste into Snowsight

# 3. Check run results for timing and errors
cat target/run_results.json | python -m json.tool

# 4. Test connection
dbt debug

# 5. Check for parsing errors
dbt parse

# 6. Retry failed models from last run
dbt retry
```

---

## 7.22 Production Checklist

### Before Going Live

- [ ] **Snowflake Setup**
  - [ ] Separate databases for dev/ci/prod
  - [ ] Dedicated warehouses per environment (right-sized)
  - [ ] RBAC roles with least-privilege access
  - [ ] Key-pair authentication for service accounts
  - [ ] Resource monitors with credit limits
  - [ ] Network policies restricting access

- [ ] **dbt Project**
  - [ ] `profiles.yml` uses `env_var()` for all secrets
  - [ ] `generate_schema_name` macro customized for environment isolation
  - [ ] All models follow naming conventions (`stg_`, `int_`, `fct_`, `dim_`)
  - [ ] Staging layer: one model per source, views only
  - [ ] Mart layer: tables or incrementals with `cluster_by` where needed
  - [ ] `copy_grants=true` on all mart models
  - [ ] `transient=true` on staging/intermediate tables

- [ ] **Testing**
  - [ ] `unique` + `not_null` on every primary key
  - [ ] `relationships` tests on every foreign key
  - [ ] `accepted_values` on every enum/status column
  - [ ] Source freshness configured with appropriate thresholds
  - [ ] At least one reconciliation singular test per mart
  - [ ] Model contracts enforced on all public mart models
  - [ ] `store_failures=true` for debugging test failures

- [ ] **CI/CD**
  - [ ] Slim CI with `state:modified+` and `--defer`
  - [ ] SQL linting (SQLFluff) in CI pipeline
  - [ ] Production manifest artifact stored for state comparison
  - [ ] CI schema cleanup after PR merge/close
  - [ ] Production deployment on merge to main

- [ ] **Orchestration**
  - [ ] Scheduled builds (daily minimum for marts)
  - [ ] Source freshness check before builds
  - [ ] Failure alerting (Slack/email/PagerDuty)
  - [ ] Retry logic for transient failures

- [ ] **Monitoring**
  - [ ] Query tag set for all dbt queries
  - [ ] Credit usage monitoring and alerting
  - [ ] Model execution time tracking
  - [ ] Elementary or equivalent for test result history

- [ ] **Documentation**
  - [ ] All models have descriptions in YAML
  - [ ] All columns on mart models have descriptions
  - [ ] Exposures defined for downstream consumers
  - [ ] `dbt docs generate` runs in production pipeline
  - [ ] Documentation site accessible to stakeholders

- [ ] **Security**
  - [ ] No passwords in code or config files
  - [ ] PII columns have masking policies
  - [ ] Row access policies where needed
  - [ ] Governance tags applied to sensitive tables
  - [ ] Audit logging enabled

---

*Back to: [README.md](./README.md) — Guide index and quick mental model.*
