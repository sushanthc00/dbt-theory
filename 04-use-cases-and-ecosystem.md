# 4. Where dbt Shines — Use Cases & Ecosystem

---

## 4.1 Data Quality Engineering

dbt transforms data quality from an afterthought into a first-class engineering concern. Coming from PySpark/Flink where you'd write custom validation logic or use Great Expectations as a separate tool, dbt integrates quality checks directly into the transformation DAG.

### The Testing Pyramid for Data

```
                    ┌─────────────┐
                    │  Singular    │  Complex, cross-model business rules
                    │  Tests       │  (e.g., revenue reconciliation)
                    ├─────────────┤
                    │  Custom      │  Package-based tests
                    │  Generic     │  (dbt_expectations, dbt_utils)
                    │  Tests       │
                    ├─────────────┤
                    │  Built-in    │  unique, not_null, accepted_values,
                    │  Generic     │  relationships
                    │  Tests       │
                    └─────────────┘
                    ◄─── Coverage ──►
```

### Source Freshness Monitoring

```yaml
sources:
  - name: raw_payments
    freshness:
      warn_after: { count: 6, period: hour }
      error_after: { count: 12, period: hour }
    loaded_at_field: _loaded_at
    tables:
      - name: transactions
      - name: refunds
        freshness:
          error_after: { count: 24, period: hour }  # Override at table level
```

```bash
dbt source freshness
# Output: WARN if data is 6-12 hours stale, ERROR if >12 hours
```

This replaces custom staleness monitoring scripts. In an orchestrator (Airflow/Dagster), you'd run this as a pre-check before triggering the dbt build.

### Contract Enforcement (dbt 1.5+)

Model contracts enforce column names, types, and constraints at build time — similar to schema enforcement in Spark's StructType:

```yaml
models:
  - name: fct_orders
    config:
      contract:
        enforced: true
    columns:
      - name: order_id
        data_type: bigint
        constraints:
          - type: not_null
          - type: primary_key
      - name: customer_id
        data_type: bigint
        constraints:
          - type: not_null
      - name: order_total
        data_type: numeric(18,2)
        constraints:
          - type: not_null
          - type: check
            expression: "order_total >= 0"
```

If the model's SELECT produces columns that don't match the contract, the build fails before any data is written. This is your schema-on-write guarantee.

---

## 4.2 Data Lineage & Governance

### Automatic Lineage

Every `ref()` and `source()` call creates a lineage edge. dbt's `manifest.json` contains the complete graph, and `dbt docs serve` renders it as an interactive DAG.

```
raw.orders ──► stg_orders ──► int_order_enriched ──► fct_daily_revenue
raw.customers ──► stg_customers ──┘                        │
                                                           ▼
                                                    dim_customers
```

This lineage is:
- **Automatic** — derived from code, not manually maintained
- **Always current** — regenerated on every `dbt docs generate`
- **Column-level** (dbt 1.6+) — tracks which upstream columns feed into downstream columns

### Exposures — Connecting to Downstream Consumers

Exposures document what BI dashboards, ML models, or applications consume your dbt models:

```yaml
# models/exposures.yml
exposures:
  - name: weekly_revenue_dashboard
    type: dashboard
    maturity: high
    url: https://looker.company.com/dashboards/42
    description: "Executive weekly revenue dashboard"
    depends_on:
      - ref('fct_daily_revenue')
      - ref('dim_customers')
    owner:
      name: Analytics Team
      email: analytics@company.com
```

This creates a lineage edge from your dbt models to the dashboard, so when someone modifies `fct_daily_revenue`, they can see which dashboards will be affected.

### Meta & Tags for Governance

```yaml
models:
  - name: dim_customers
    meta:
      owner: "data-platform-team"
      contains_pii: true
      retention_days: 365
      classification: "confidential"
    tags: ['daily', 'pii', 'tier-1']
    columns:
      - name: email
        meta:
          pii: true
          masking_policy: "email_mask"
```

These metadata fields are available in `manifest.json` and can be consumed by governance tools, access control automation, or custom scripts.

---

## 4.3 Modular Data Modeling

### The Staging → Intermediate → Marts Pattern

This is the canonical dbt modeling pattern, and it maps cleanly to dimensional modeling concepts:

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   STAGING    │     │   INTERMEDIATE   │     │     MARTS       │
│              │     │                  │     │                 │
│ 1:1 source   │────►│ Business logic   │────►│ Consumption     │
│ Rename/cast  │     │ Joins/filters    │     │ Fact & Dim      │
│ Minimal logic│     │ Aggregations     │     │ Wide tables     │
│              │     │ Pivots           │     │ Star schemas    │
│ stg_*        │     │ int_*            │     │ fct_* / dim_*   │
└─────────────┘     └──────────────────┘     └─────────────────┘
  materialized:       materialized:            materialized:
  view                ephemeral/view           table/incremental
```

### Staging Layer — The Clean Interface

```sql
-- models/staging/ecommerce/stg_ecommerce__orders.sql
{{ config(materialized='view') }}

WITH source AS (
    SELECT * FROM {{ source('raw_ecommerce', 'orders') }}
),

renamed AS (
    SELECT
        -- Primary key
        id AS order_id,

        -- Foreign keys
        customer_id,
        product_id,

        -- Dimensions
        UPPER(TRIM(status)) AS order_status,
        shipping_method,

        -- Measures (type-safe)
        CAST(amount_cents AS DECIMAL(18,2)) / 100 AS order_amount,
        CAST(tax_cents AS DECIMAL(18,2)) / 100 AS tax_amount,

        -- Timestamps (standardized)
        CAST(created_at AS TIMESTAMP_NTZ) AS ordered_at,
        CAST(shipped_at AS TIMESTAMP_NTZ) AS shipped_at,
        CAST(delivered_at AS TIMESTAMP_NTZ) AS delivered_at,

        -- Metadata
        _loaded_at AS _etl_loaded_at
    FROM source
)

SELECT * FROM renamed
```

Rules for staging models:
- One staging model per source table (1:1 mapping)
- Only renaming, casting, and basic cleansing
- No joins, no aggregations, no business logic
- Naming convention: `stg_{source_name}__{table_name}`

### Intermediate Layer — Business Logic

```sql
-- models/intermediate/int_order_items_enriched.sql
{{ config(materialized='ephemeral') }}

SELECT
    o.order_id,
    o.customer_id,
    o.ordered_at,
    o.order_status,
    p.product_category,
    p.product_name,
    o.order_amount,
    o.tax_amount,
    o.order_amount + o.tax_amount AS total_amount,
    DATEDIFF('day', o.ordered_at, o.shipped_at) AS days_to_ship,
    CASE
        WHEN o.order_amount >= 500 THEN 'premium'
        WHEN o.order_amount >= 100 THEN 'standard'
        ELSE 'basic'
    END AS order_tier
FROM {{ ref('stg_ecommerce__orders') }} AS o
LEFT JOIN {{ ref('stg_ecommerce__products') }} AS p
    ON o.product_id = p.product_id
```

### Mart Layer — Consumption-Ready

**Fact table (event grain):**
```sql
-- models/marts/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
) }}

SELECT
    {{ dbt_utils.generate_surrogate_key(['o.order_id']) }} AS order_sk,
    o.order_id,
    o.customer_id,
    c.customer_sk,
    o.ordered_at,
    o.order_status,
    o.order_tier,
    o.product_category,
    o.order_amount,
    o.tax_amount,
    o.total_amount,
    o.days_to_ship
FROM {{ ref('int_order_items_enriched') }} AS o
LEFT JOIN {{ ref('dim_customers') }} AS c
    ON o.customer_id = c.customer_id

{% if is_incremental() %}
WHERE o.ordered_at > (SELECT MAX(ordered_at) FROM {{ this }})
{% endif %}
```

**Dimension table:**
```sql
-- models/marts/dim_customers.sql
{{ config(materialized='table') }}

SELECT
    {{ dbt_utils.generate_surrogate_key(['customer_id']) }} AS customer_sk,
    customer_id,
    customer_name,
    email,
    country,
    signup_date,
    first_order_date,
    last_order_date,
    total_orders,
    lifetime_value,
    CASE
        WHEN total_orders = 0 THEN 'prospect'
        WHEN last_order_date >= CURRENT_DATE - INTERVAL '90 days' THEN 'active'
        WHEN last_order_date >= CURRENT_DATE - INTERVAL '365 days' THEN 'lapsed'
        ELSE 'churned'
    END AS customer_segment
FROM {{ ref('int_customer_aggregated') }}
```

### One Big Table (OBT) Pattern

For simpler analytics or when star schemas add unnecessary complexity:

```sql
-- models/marts/obt_order_analytics.sql
{{ config(materialized='table') }}

SELECT
    -- Order facts
    o.order_id,
    o.ordered_at,
    o.order_amount,
    o.order_status,

    -- Customer dimensions (denormalized)
    c.customer_name,
    c.country AS customer_country,
    c.customer_segment,
    c.lifetime_value AS customer_ltv,

    -- Product dimensions (denormalized)
    p.product_name,
    p.product_category,
    p.unit_cost,

    -- Derived metrics
    o.order_amount - p.unit_cost AS gross_margin,
    DATE_TRUNC('month', o.ordered_at) AS order_month,
    ROW_NUMBER() OVER (
        PARTITION BY o.customer_id ORDER BY o.ordered_at
    ) AS customer_order_sequence
FROM {{ ref('stg_ecommerce__orders') }} AS o
LEFT JOIN {{ ref('dim_customers') }} AS c ON o.customer_id = c.customer_id
LEFT JOIN {{ ref('stg_ecommerce__products') }} AS p ON o.product_id = p.product_id
```

OBT trades storage efficiency for query simplicity. In columnar warehouses, the storage overhead is minimal, and BI tools perform better against wide, flat tables.

---

## 4.4 The Modern Data Stack — Where dbt Fits

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE MODERN DATA STACK                         │
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │  Sources  │   │    EL    │   │Warehouse │   │    BI     │    │
│  │          │   │          │   │          │   │          │    │
│  │ Postgres  │──►│ Fivetran │──►│Snowflake │──►│ Looker   │    │
│  │ MongoDB   │   │ Airbyte  │   │BigQuery  │   │ Tableau  │    │
│  │ APIs      │   │ Stitch   │   │Redshift  │   │ Metabase │    │
│  │ S3/GCS    │   │ Meltano  │   │Databricks│   │ Mode     │    │
│  └──────────┘   └──────────┘   └────┬─────┘   └──────────┘    │
│                                      │                          │
│                               ┌──────▼─────┐                   │
│                               │    dbt      │                   │
│                               │ (Transform) │                   │
│                               └──────┬─────┘                   │
│                                      │                          │
│                               ┌──────▼─────┐                   │
│                               │Orchestrator │                   │
│                               │             │                   │
│                               │ Airflow     │                   │
│                               │ Dagster     │                   │
│                               │ Prefect     │                   │
│                               │ dbt Cloud   │                   │
│                               └─────────────┘                   │
│                                                                 │
│  Cross-cutting: Git │ CI/CD │ Monitoring │ Data Catalog          │
└─────────────────────────────────────────────────────────────────┘
```

### Integration Points

| Layer | Tool | How dbt Integrates |
|-------|------|-------------------|
| Extraction/Loading | Fivetran, Airbyte, Meltano | dbt consumes their output via `source()` |
| Warehouse | Snowflake, BigQuery, Redshift, Databricks, Postgres | dbt connects via adapters in `profiles.yml` |
| Orchestration | Airflow | `BashOperator` or `dbt-airflow` provider; trigger `dbt build` |
| Orchestration | Dagster | `dagster-dbt` integration; each dbt model becomes a Dagster asset |
| Orchestration | Prefect | `prefect-dbt` integration; dbt commands as Prefect tasks |
| Orchestration | dbt Cloud | Built-in scheduler with job definitions |
| BI | Looker | LookML can reference dbt model descriptions via metadata sync |
| BI | Tableau, Metabase | Connect directly to dbt-materialized tables/views |
| Data Catalog | Atlan, DataHub, OpenMetadata | Ingest `manifest.json` for lineage and metadata |
| Quality | Monte Carlo, Elementary | Monitor dbt test results and model freshness |

### Orchestration Pattern: Airflow + dbt

```python
# Airflow DAG (simplified)
from airflow.operators.bash import BashOperator

with DAG('dbt_daily_build', schedule='0 6 * * *'):

    source_freshness = BashOperator(
        task_id='check_source_freshness',
        bash_command='cd /opt/dbt && dbt source freshness --target prod'
    )

    dbt_build = BashOperator(
        task_id='dbt_build',
        bash_command='cd /opt/dbt && dbt build --target prod --select tag:daily'
    )

    dbt_docs = BashOperator(
        task_id='generate_docs',
        bash_command='cd /opt/dbt && dbt docs generate --target prod'
    )

    source_freshness >> dbt_build >> dbt_docs
```

### Orchestration Pattern: Dagster + dbt (Asset-Based)

```python
# Dagster treats each dbt model as a software-defined asset
from dagster_dbt import DbtCliResource, dbt_assets

dbt_resource = DbtCliResource(project_dir="/opt/dbt")

@dbt_assets(manifest="/opt/dbt/target/manifest.json")
def my_dbt_assets(context, dbt: DbtCliResource):
    yield from dbt.cli(["build"], context=context).stream()
```

Dagster's approach is particularly powerful because it:
- Represents each dbt model as a first-class asset with its own materialization history
- Enables partial pipeline runs (only re-materialize stale assets)
- Provides a unified lineage view across dbt models and non-dbt assets (ML models, Python transforms)

---

## 4.5 dbt for Specific Use Cases

### Financial Reconciliation

```sql
-- tests/assert_revenue_reconciles.sql
-- Ensures the mart total matches the source total within tolerance
WITH mart_total AS (
    SELECT SUM(order_amount) AS total FROM {{ ref('fct_orders') }}
),
source_total AS (
    SELECT SUM(CAST(amount_cents AS DECIMAL(18,2)) / 100) AS total
    FROM {{ source('raw_ecommerce', 'orders') }}
    WHERE status != 'cancelled'
)
SELECT
    m.total AS mart_total,
    s.total AS source_total,
    ABS(m.total - s.total) AS difference
FROM mart_total m
CROSS JOIN source_total s
WHERE ABS(m.total - s.total) > 0.01  -- Tolerance for floating point
```

### Multi-Tenant Data Isolation

```sql
-- macros/filter_tenant.sql
{% macro filter_tenant() %}
    {% if target.name != 'prod' %}
        WHERE tenant_id = '{{ var("dev_tenant_id", "test_tenant") }}'
    {% endif %}
{% endmacro %}
```

```sql
-- models/staging/stg_orders.sql
SELECT * FROM {{ source('raw', 'orders') }}
{{ filter_tenant() }}
```

### Event Stream Processing (Bridging Flink → dbt)

If you're landing Flink/Kafka event streams into your warehouse (via Kafka Connect, Fivetran, etc.), dbt handles the downstream transformation:

```sql
-- models/marts/fct_user_sessions.sql
{{ config(
    materialized='incremental',
    unique_key='session_id',
    incremental_strategy='merge',
    partition_by={
        "field": "session_start",
        "data_type": "timestamp",
        "granularity": "day"
    }
) }}

WITH events AS (
    SELECT * FROM {{ ref('stg_clickstream_events') }}
    {% if is_incremental() %}
    WHERE event_timestamp > (SELECT MAX(session_start) - INTERVAL '2 hours' FROM {{ this }})
    {% endif %}
),

sessionized AS (
    SELECT
        {{ dbt_utils.generate_surrogate_key(['user_id', 'session_start']) }} AS session_id,
        user_id,
        MIN(event_timestamp) AS session_start,
        MAX(event_timestamp) AS session_end,
        COUNT(*) AS event_count,
        COUNT(DISTINCT page_url) AS unique_pages,
        DATEDIFF('second', MIN(event_timestamp), MAX(event_timestamp)) AS session_duration_seconds
    FROM (
        SELECT
            *,
            SUM(CASE WHEN time_since_last_event > 1800 THEN 1 ELSE 0 END)
                OVER (PARTITION BY user_id ORDER BY event_timestamp) AS session_index
        FROM (
            SELECT
                *,
                DATEDIFF('second',
                    LAG(event_timestamp) OVER (PARTITION BY user_id ORDER BY event_timestamp),
                    event_timestamp
                ) AS time_since_last_event
            FROM events
        )
    )
    GROUP BY user_id, session_index
)

SELECT * FROM sessionized
```

This is the dbt equivalent of a Flink session window — but operating in batch/micro-batch mode on warehouse data rather than in real-time.

---

## 4.6 Anti-Patterns & Pitfalls

| Anti-Pattern | Why It's Bad | What to Do Instead |
|-------------|-------------|-------------------|
| Business logic in staging models | Staging should be a clean interface; logic here is hidden | Move logic to intermediate or mart models |
| Hardcoded schema/table names | Breaks across environments | Always use `ref()` and `source()` |
| One massive model with 500+ lines | Untestable, unreviewable, slow to iterate | Break into intermediate models |
| Ephemeral models 4+ levels deep | Compiled SQL becomes enormous, hard to debug | Materialize intermediate steps as views |
| No tests on primary/foreign keys | Silent data quality issues | At minimum: `unique` + `not_null` on every key |
| Running `dbt run` then `dbt test` separately | A model can be built with bad data before tests catch it | Use `dbt build` (interleaves run + test) |
| Incremental models without `--full-refresh` plan | Data drift accumulates over time | Schedule periodic full refreshes |
| Ignoring the `target/compiled/` directory | Debugging Jinja in your head | Always check compiled SQL when troubleshooting |

---

## 4.7 Quick Reference: dbt Adapters

dbt supports many warehouses via adapters. Your `profiles.yml` `type` field determines which adapter is used.

| Warehouse | Adapter | `type` value | Notes |
|-----------|---------|-------------|-------|
| Snowflake | dbt-snowflake | `snowflake` | Most mature, full feature support |
| BigQuery | dbt-bigquery | `bigquery` | Partition/cluster support, `insert_overwrite` strategy |
| Redshift | dbt-redshift | `redshift` | `delete+insert` preferred over `merge` |
| Databricks | dbt-databricks | `databricks` | Unity Catalog support, Delta Lake |
| PostgreSQL | dbt-postgres | `postgres` | Great for development/testing |
| Spark | dbt-spark | `spark` | For Spark SQL / Hive Metastore |
| Trino/Starburst | dbt-trino | `trino` | Federated query engine |
| DuckDB | dbt-duckdb | `duckdb` | Local development, embedded analytics |

---

*Back to: [README.md](./README.md) — Guide index and quick mental model.*
