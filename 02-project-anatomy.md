# 2. The Anatomy of a dbt Project

---

## 2.1 Models — The Core Unit of Work

A dbt model is a `.sql` file containing a single `SELECT` statement. That's it. No `CREATE TABLE`, no `INSERT INTO` — just a SELECT. dbt wraps it in the appropriate DDL at execution time based on the materialization strategy.

### File Location & Naming

```
models/
├── staging/           # 1:1 mappings from source tables
│   ├── _stg_models.yml
│   ├── stg_orders.sql
│   └── stg_customers.sql
├── intermediate/      # Business logic, joins, aggregations
│   ├── int_order_items_pivoted.sql
│   └── int_customer_lifetime_value.sql
└── marts/             # Consumption-ready, wide tables
    ├── _mart_models.yml
    ├── fct_daily_revenue.sql
    └── dim_customers.sql
```

Convention: prefix with `stg_`, `int_`, `fct_`, `dim_` to signal the model's role in the DAG.

### The `ref()` Function — DAG Construction

`ref()` is the single most important function in dbt. It does two things:
1. **Resolves** the model name to its fully qualified table/view name in the warehouse (handling schema, database, and alias)
2. **Declares a dependency** that dbt uses to build the execution DAG

```sql
-- models/marts/fct_daily_revenue.sql
SELECT
    o.order_date,
    SUM(o.amount) AS total_revenue,
    COUNT(DISTINCT o.customer_id) AS unique_customers
FROM {{ ref('stg_orders') }} AS o
WHERE o.status = 'completed'
GROUP BY o.order_date
```

At compile time, `{{ ref('stg_orders') }}` becomes something like `"analytics"."staging"."stg_orders"` — the actual schema depends on your target environment.

**Cross-project refs** (dbt Mesh, available in dbt 1.6+):
```sql
-- Reference a model from another dbt project
SELECT * FROM {{ ref('shared_project', 'dim_customers') }}
```

### The `source()` Function — Raw Data Entry Points

`source()` references raw tables that exist in your warehouse but are NOT managed by dbt:

```yaml
# models/staging/_sources.yml
sources:
  - name: raw_ecommerce
    database: raw_db
    schema: public
    tables:
      - name: orders
        loaded_at_field: _etl_loaded_at
        freshness:
          warn_after: { count: 12, period: hour }
          error_after: { count: 24, period: hour }
      - name: customers
```

```sql
-- models/staging/stg_orders.sql
SELECT
    id AS order_id,
    customer_id,
    amount,
    status,
    created_at AS order_date
FROM {{ source('raw_ecommerce', 'orders') }}
```

`source()` enables:
- Source freshness checks (`dbt source freshness`)
- Lineage tracking from raw tables through to marts
- Centralized source configuration (rename once, propagate everywhere)

---

## 2.2 Materializations — How Models Become Database Objects

Materialization controls *how* dbt persists the result of your SELECT statement. This is where performance engineering happens.

### Comparison Matrix

| Materialization | Warehouse Object | Storage Cost | Query Performance | Rebuild Cost | Use When |
|----------------|-----------------|-------------|-------------------|-------------|----------|
| `view` | VIEW | None | Query-time compute | Instant | Staging models, light transforms, data < 100M rows |
| `table` | TABLE | Full | Pre-computed, fast | Full rebuild every run | Mart models, heavy aggregations, frequently queried |
| `incremental` | TABLE | Full | Pre-computed, fast | Only new/changed rows | Large fact tables, append-heavy, event streams |
| `ephemeral` | CTE (no object) | None | Inlined into parent | N/A | Helper logic, avoid warehouse object clutter |

### View Materialization

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}

SELECT
    id AS order_id,
    customer_id,
    CAST(amount AS DECIMAL(18,2)) AS amount,
    status,
    created_at AS order_date
FROM {{ source('raw_ecommerce', 'orders') }}
```

Under the hood, dbt executes:
```sql
CREATE OR REPLACE VIEW "analytics"."staging"."stg_orders" AS (
    SELECT
        id AS order_id,
        customer_id,
        CAST(amount AS DECIMAL(18,2)) AS amount,
        status,
        created_at AS order_date
    FROM "raw_db"."public"."orders"
);
```

### Table Materialization

```sql
{{ config(materialized='table') }}

SELECT
    customer_id,
    MIN(order_date) AS first_order_date,
    MAX(order_date) AS last_order_date,
    COUNT(*) AS total_orders,
    SUM(amount) AS lifetime_value
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

Under the hood (simplified):
```sql
CREATE OR REPLACE TABLE "analytics"."marts"."dim_customers" AS (
    SELECT ...
);
```

dbt actually uses a swap pattern for atomicity: creates a temp table, then renames it to replace the existing one. This prevents downstream queries from hitting a half-built table.

### Incremental Materialization (Deep Dive)

This is the most complex and most powerful materialization. It's analogous to how you'd handle micro-batch processing in Spark Structured Streaming or Flink checkpointed state.

```sql
-- models/marts/fct_events.sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge',
    on_schema_change='append_new_columns'
) }}

SELECT
    event_id,
    user_id,
    event_type,
    event_payload,
    occurred_at
FROM {{ ref('stg_events') }}

{% if is_incremental() %}
    -- This filter only applies on incremental runs (not full refreshes)
    WHERE occurred_at > (SELECT MAX(occurred_at) FROM {{ this }})
{% endif %}
```

How it works:
1. **First run**: dbt creates the table with ALL rows (the `is_incremental()` block is skipped)
2. **Subsequent runs**: dbt only processes rows matching the `WHERE` clause and merges them into the existing table
3. `{{ this }}` refers to the current model's existing table in the warehouse

**Incremental Strategies** (warehouse-dependent):

| Strategy | Behavior | Best For | Supported On |
|----------|----------|----------|-------------|
| `append` | INSERT only, no dedup | Immutable event logs | All warehouses |
| `merge` | MERGE/UPSERT on `unique_key` | Mutable dimensions, late-arriving facts | Snowflake, BigQuery, Databricks |
| `delete+insert` | DELETE matching keys, then INSERT | When MERGE is expensive | Redshift, Postgres |
| `insert_overwrite` | Replace entire partitions | Partition-aligned fact tables | BigQuery, Spark, Databricks |

**Critical gotcha**: If your incremental logic has a bug and misses rows, those rows are permanently lost from the target table. Always have a `--full-refresh` strategy as a safety net:
```bash
dbt run --select fct_events --full-refresh
```

### Ephemeral Materialization

```sql
{{ config(materialized='ephemeral') }}

SELECT
    order_id,
    CASE
        WHEN amount > 1000 THEN 'high_value'
        WHEN amount > 100 THEN 'medium_value'
        ELSE 'low_value'
    END AS order_tier
FROM {{ ref('stg_orders') }}
```

This model creates NO warehouse object. Instead, its SQL is injected as a CTE into any model that references it via `ref()`. Use sparingly — deeply nested ephemeral models create massive compiled SQL that's hard to debug.

---

## 2.3 Tests — Data Quality as Code

### Generic (Schema) Tests

Declared in YAML, these are parameterized assertions:

```yaml
# models/marts/_mart_models.yml
models:
  - name: fct_daily_revenue
    description: "Daily revenue aggregation"
    columns:
      - name: order_date
        description: "Calendar date of the revenue"
        tests:
          - unique
          - not_null
      - name: total_revenue
        tests:
          - not_null
      - name: unique_customers
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
              inclusive: true
```

Built-in generic tests:
- `unique` — No duplicate values
- `not_null` — No NULL values
- `accepted_values` — Column only contains specified values
- `relationships` — Foreign key integrity (referential integrity check)

Each test compiles to a SELECT that returns failing rows:
```sql
-- Compiled test for 'unique' on order_date
SELECT order_date
FROM "analytics"."marts"."fct_daily_revenue"
GROUP BY order_date
HAVING COUNT(*) > 1;
-- If this returns 0 rows → test passes
```

### Singular (Custom) Tests

For complex, model-specific assertions, write a SQL file in the `tests/` directory:

```sql
-- tests/assert_revenue_not_negative.sql
-- This test FAILS if any rows are returned
SELECT
    order_date,
    total_revenue
FROM {{ ref('fct_daily_revenue') }}
WHERE total_revenue < 0
```

### Test Severity & Thresholds

```yaml
columns:
  - name: email
    tests:
      - not_null:
          severity: warn          # warn instead of error
      - unique:
          severity: error
          error_if: ">100"        # fail only if >100 duplicates
          warn_if: ">10"          # warn if >10 duplicates
```

---

## 2.4 Snapshots — Type 2 Slowly Changing Dimensions

Snapshots capture how a row changes over time. If you've implemented SCD Type 2 in a warehouse before, you know the pain. dbt automates it.

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}

{{ config(
    target_schema='snapshots',
    unique_key='customer_id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=True
) }}

SELECT
    customer_id,
    name,
    email,
    plan_type,
    updated_at
FROM {{ source('raw_ecommerce', 'customers') }}

{% endsnapshot %}
```

**What dbt does under the hood:**

On first run, creates the snapshot table with added columns:
- `dbt_scd_id` — Surrogate key for the snapshot row
- `dbt_updated_at` — When this version was captured
- `dbt_valid_from` — When this version became active
- `dbt_valid_to` — When this version was superseded (NULL = current)

On subsequent runs:
1. Compares current source data against the latest snapshot
2. For changed rows: sets `dbt_valid_to` on the old version, inserts a new row
3. For deleted rows (if `invalidate_hard_deletes=True`): closes the record

**Snapshot Strategies:**

| Strategy | Detects Changes Via | Use When |
|----------|-------------------|----------|
| `timestamp` | Compares `updated_at` column | Source has a reliable timestamp |
| `check` | Compares specified columns | No reliable timestamp; compares actual values |

```sql
-- check strategy example
{{ config(
    strategy='check',
    check_cols=['name', 'email', 'plan_type']
) }}
```

**Result table example:**

| customer_id | name | plan_type | dbt_valid_from | dbt_valid_to |
|-------------|------|-----------|----------------|-------------|
| 42 | Alice | free | 2024-01-01 | 2024-06-15 |
| 42 | Alice | premium | 2024-06-15 | NULL |

---

## 2.5 Seeds — Static Reference Data

Seeds are CSV files that dbt loads into your warehouse as tables. Use them for small, static reference data that changes infrequently.

```
seeds/
├── country_codes.csv
├── currency_exchange_rates.csv
└── product_categories.csv
```

```csv
-- seeds/country_codes.csv
country_code,country_name,region
US,United States,North America
GB,United Kingdom,Europe
DE,Germany,Europe
JP,Japan,Asia Pacific
```

```bash
dbt seed    # Loads all CSVs into the warehouse
```

Configure column types explicitly (dbt infers types, but inference can be wrong):

```yaml
# dbt_project.yml
seeds:
  my_project:
    country_codes:
      +column_types:
        country_code: varchar(2)
        country_name: varchar(100)
        region: varchar(50)
```

**When to use seeds vs. sources:**
- Seeds: < 1000 rows, changes via PR (version controlled), reference/lookup data
- Sources: Any size, loaded by external EL tools, transactional/operational data

---

## 2.6 Macros & Jinja — DRY, Dynamic SQL

Jinja is dbt's templating engine. If you've used Jinja2 in Python (Flask, Ansible), the syntax is identical. Macros are reusable Jinja functions.

### Jinja Basics in dbt

```
{{ ... }}  — Expression (outputs a value)
{% ... %}  — Statement (control flow, no output)
{# ... #}  — Comment (stripped from compiled SQL)
```

### Common Patterns

**Conditional logic:**
```sql
SELECT
    order_id,
    amount,
    {% if target.name == 'prod' %}
        customer_email    -- Only include PII in prod
    {% else %}
        MD5(customer_email) AS customer_email_hash
    {% endif %}
FROM {{ ref('stg_orders') }}
```

**Looping:**
```sql
{% set payment_methods = ['credit_card', 'bank_transfer', 'paypal', 'crypto'] %}

SELECT
    order_id,
    {% for method in payment_methods %}
        SUM(CASE WHEN payment_method = '{{ method }}' THEN amount ELSE 0 END)
            AS {{ method }}_amount
        {% if not loop.last %},{% endif %}
    {% endfor %}
FROM {{ ref('stg_payments') }}
GROUP BY order_id
```

### Writing Custom Macros

```sql
-- macros/cents_to_dollars.sql
{% macro cents_to_dollars(column_name, precision=2) %}
    ROUND(CAST({{ column_name }} AS DECIMAL(18, {{ precision }})) / 100, {{ precision }})
{% endmacro %}
```

Usage in a model:
```sql
SELECT
    order_id,
    {{ cents_to_dollars('amount_cents') }} AS amount_dollars,
    {{ cents_to_dollars('tax_cents', 4) }} AS tax_dollars
FROM {{ ref('stg_orders') }}
```

### Advanced Macro: Generate Schema Name Override

One of the most commonly customized macros — controls which schema models land in:

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- elif target.name == 'prod' -%}
        {{ custom_schema_name | trim }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

This ensures:
- In prod: models go to their declared schema (e.g., `staging`, `marts`)
- In dev: models go to `dbt_yourname_staging`, `dbt_yourname_marts` (isolated)

### Advanced Macro: Dynamic Pivot

A realistic macro you'd use in production — pivoting rows to columns dynamically:

```sql
-- macros/pivot.sql
{% macro pivot(column, values, alias=True, agg='SUM', then_value=1, else_value=0, prefix='', suffix='', quote_identifiers=True) %}
    {% for value in values %}
        {{ agg }}(
            CASE
                WHEN {{ column }} = '{{ value }}'
                THEN {{ then_value }}
                ELSE {{ else_value }}
            END
        ) AS {{ prefix }}{{ value | replace(' ', '_') | lower }}{{ suffix }}
        {% if not loop.last %},{% endif %}
    {% endfor %}
{% endmacro %}
```

Usage:
```sql
SELECT
    customer_id,
    {{ pivot(
        column='order_status',
        values=['pending', 'shipped', 'delivered', 'cancelled'],
        agg='COUNT',
        prefix='orders_'
    ) }}
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

### The `run_query()` Macro — Introspection

Execute SQL at compile time and use the results in your Jinja logic:

```sql
{% macro get_column_values(table, column) %}
    {% set query %}
        SELECT DISTINCT {{ column }} FROM {{ table }} ORDER BY 1
    {% endset %}
    {% set results = run_query(query) %}
    {% if execute %}
        {% set values = results.columns[0].values() %}
        {{ return(values) }}
    {% else %}
        {{ return([]) }}
    {% endif %}
{% endmacro %}
```

---

## 2.7 The `dbt_project.yml` File

This is the root configuration file. Every dbt project has exactly one.

```yaml
# dbt_project.yml
name: 'ecommerce_analytics'
version: '1.0.0'
config-version: 2

profile: 'ecommerce_analytics'   # Maps to profiles.yml

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"             # Compiled SQL output
clean-targets: ["target", "dbt_packages"]

# Global variable definitions
vars:
  start_date: '2023-01-01'
  enable_pii: false

# Model-level configuration (cascading)
models:
  ecommerce_analytics:            # Must match 'name' above
    +materialized: view           # Default for all models
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: ephemeral
    marts:
      +materialized: table
      +schema: marts
      +tags: ['daily']

# Seed configuration
seeds:
  ecommerce_analytics:
    +schema: seeds

# Snapshot configuration
snapshots:
  ecommerce_analytics:
    +target_schema: snapshots
```

**Configuration precedence** (highest to lowest):
1. `{{ config() }}` block in the model file
2. Resource-specific YAML properties file
3. `dbt_project.yml` directory-level config
4. `dbt_project.yml` project-level config

### Using Variables

Define in `dbt_project.yml`:
```yaml
vars:
  lookback_days: 30
```

Access in models:
```sql
WHERE order_date >= CURRENT_DATE - INTERVAL '{{ var("lookback_days") }}' DAY
```

Override at runtime:
```bash
dbt run --vars '{"lookback_days": 90}'
```

---

## 2.8 Packages — Standing on the Shoulders of Giants

Packages are reusable dbt projects published to the [dbt Hub](https://hub.getdbt.com/) or Git repositories.

```yaml
# packages.yml (project root)
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.1.0", "<2.0.0"]

  - package: calogica/dbt_expectations
    version: [">=0.10.0", "<0.11.0"]

  - package: dbt-labs/codegen
    version: [">=0.12.0", "<0.13.0"]

  # Git-based package (private or custom)
  - git: "https://github.com/your-org/shared-dbt-macros.git"
    revision: v1.2.0
```

Install with:
```bash
dbt deps
```

### Essential Packages

| Package | Purpose | Key Macros/Tests |
|---------|---------|-----------------|
| `dbt_utils` | Swiss army knife | `surrogate_key`, `pivot`, `union_relations`, `date_spine`, `star` |
| `dbt_expectations` | Great Expectations-style tests | `expect_column_values_to_be_between`, `expect_table_row_count_to_be_between` |
| `codegen` | Code generation helpers | `generate_source`, `generate_model_yaml`, `generate_base_model` |
| `dbt_date` | Date/time utilities | `get_date_dimension`, `day_of_week`, fiscal calendar helpers |
| `audit_helper` | Data auditing | `compare_relations`, `compare_column_values` |

### Using `dbt_utils.surrogate_key`

```sql
SELECT
    {{ dbt_utils.generate_surrogate_key(['order_id', 'line_item_id']) }} AS order_line_sk,
    order_id,
    line_item_id,
    product_id,
    quantity,
    unit_price
FROM {{ ref('stg_order_lines') }}
```

### Using `dbt_utils.union_relations`

When you have identically structured tables across schemas (e.g., multi-tenant):

```sql
{{ dbt_utils.union_relations(
    relations=[
        ref('stg_orders_us'),
        ref('stg_orders_eu'),
        ref('stg_orders_apac')
    ],
    include=["order_id", "customer_id", "amount", "order_date"]
) }}
```

---

*Next: [03-workflow-and-cli.md](./03-workflow-and-cli.md) — The development loop and CLI mastery.*
