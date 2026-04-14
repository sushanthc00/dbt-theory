# 6. Additional Topics — Filling the Gaps

> Supplementary material covering areas not deeply addressed in guides 01–05: Python models, unit testing, dbt Mesh, advanced incremental patterns, model versioning, model access modifiers, groups, semantic layer deep dive, and operational best practices.

---

## 6.1 Python Models (dbt 1.3+)

While dbt is SQL-first, Python models let you run pandas, scikit-learn, or any Python logic inside the warehouse's Python runtime (Snowpark for Snowflake, PySpark for Databricks, DataFrames for BigQuery).

### When to Use Python Models

- ML feature engineering that's awkward in SQL (rolling statistics, text processing)
- Calling external APIs or libraries during transformation
- Complex procedural logic that doesn't translate cleanly to set-based SQL
- Statistical computations (scipy, statsmodels)

### Syntax

```python
# models/intermediate/int_customer_churn_features.py
def model(dbt, session):
    # Read upstream dbt models
    orders = dbt.ref("stg_orders").to_pandas()
    customers = dbt.ref("stg_customers").to_pandas()

    # Python logic
    customer_stats = (
        orders.groupby("customer_id")
        .agg(
            total_orders=("order_id", "count"),
            avg_order_value=("order_amount", "mean"),
            days_since_last_order=("ordered_at", lambda x: (pd.Timestamp.now() - x.max()).days),
        )
        .reset_index()
    )

    result = customers.merge(customer_stats, on="customer_id", how="left")
    return result
```

### Configuration

```python
def model(dbt, session):
    dbt.config(
        materialized="table",
        packages=["scikit-learn==1.3.0", "pandas"],  # Snowpark-specific
        schema="ml_features",
    )
    # ...
```

### Limitations

- Python models are always materialized as `table` (no view, incremental, or ephemeral)
- Execution is slower than SQL — the warehouse spins up a Python runtime
- Not all warehouses support Python models (Snowflake, Databricks, BigQuery do; Redshift, Postgres do not)
- Debugging is harder — no `target/compiled/` equivalent for Python
- Python models can `ref()` SQL models and vice versa, but mixing heavily creates a confusing DAG

### Best Practice

Use Python models sparingly. If you can express it in SQL (even complex SQL with window functions), prefer SQL. Reserve Python for genuinely procedural or library-dependent logic.

---

## 6.2 Unit Testing (dbt 1.8+)

Unit tests validate model logic with mocked inputs — no warehouse data needed. This is a game-changer for testing business logic in isolation.

### Why Unit Tests Matter

- Schema/generic tests validate data quality on materialized results
- Unit tests validate transformation logic before it touches real data
- Faster feedback loop — no need to run against the warehouse
- Test edge cases that may not exist in your current data

### Syntax

```yaml
# models/marts/_mart_unit_tests.yml
unit_tests:
  - name: test_order_tier_classification
    description: "Verify order tier logic assigns correct tiers"
    model: int_order_items_enriched
    given:
      - input: ref('stg_ecommerce__orders')
        rows:
          - { order_id: 1, customer_id: 100, order_amount: 50.00, order_status: "completed" }
          - { order_id: 2, customer_id: 101, order_amount: 250.00, order_status: "completed" }
          - { order_id: 3, customer_id: 102, order_amount: 750.00, order_status: "completed" }
      - input: ref('stg_ecommerce__products')
        rows:
          - { product_id: 1, product_category: "electronics", product_name: "Widget" }
    expect:
      rows:
        - { order_id: 1, order_tier: "basic" }
        - { order_id: 2, order_tier: "standard" }
        - { order_id: 3, order_tier: "premium" }
```

### Key Concepts

- `given`: Mock inputs for each `ref()` or `source()` the model uses
- `expect`: Expected output rows (only columns you care about)
- `overrides`: Override `var()` or `env_var()` values for the test
- Unit tests run with `dbt test` or `dbt build` alongside other tests

### Testing Incremental Logic

```yaml
unit_tests:
  - name: test_incremental_filter
    model: fct_events
    given:
      - input: ref('stg_events')
        rows:
          - { event_id: 1, occurred_at: "2024-01-15 10:00:00" }
          - { event_id: 2, occurred_at: "2024-01-16 10:00:00" }
      - input: this  # Mock the existing target table
        rows:
          - { event_id: 0, occurred_at: "2024-01-15 00:00:00" }
    expect:
      rows:
        - { event_id: 2 }  # Only the row after max(occurred_at)
```

The `this` input mocks the existing incremental table, letting you test the `is_incremental()` filter logic.

---

## 6.3 dbt Mesh & Multi-Project Architecture (dbt 1.6+)

dbt Mesh enables multiple dbt projects to reference each other's models — essential for large organizations with domain-oriented teams.

### The Problem Mesh Solves

In a monolithic dbt project:
- 500+ models owned by different teams
- Merge conflicts on shared files
- One team's broken model blocks everyone's CI
- No clear ownership boundaries

### Cross-Project References

```sql
-- In the "marketing" dbt project
SELECT *
FROM {{ ref('core_platform', 'dim_customers') }}
-- References dim_customers from the "core_platform" project
```

### Access Modifiers

Control which models other projects can reference:

```yaml
# In the core_platform project
models:
  - name: dim_customers
    access: public          # Other projects can ref() this
    config:
      contract:
        enforced: true      # Public models MUST have contracts

  - name: int_customer_aggregated
    access: protected       # Only models in the same project/group

  - name: stg_customers
    access: private         # Only models in the same directory/group
```

| Access Level | Who Can `ref()` It | Use For |
|-------------|-------------------|---------|
| `private` | Same group only | Staging, internal helpers |
| `protected` | Same project (default) | Intermediate models |
| `public` | Any project | Mart models, data products |

### Groups

Groups define ownership boundaries within a project:

```yaml
# dbt_project.yml
groups:
  - name: finance
    owner:
      name: Finance Data Team
      email: finance-data@company.com

  - name: marketing
    owner:
      name: Marketing Analytics
      email: marketing-analytics@company.com
```

```yaml
# models/finance/_finance_models.yml
models:
  - name: fct_revenue
    group: finance
    access: public
```

Groups enforce that `private` models are only referenced within their group, providing team-level encapsulation.

---

## 6.4 Model Versions (dbt 1.6+)

Model versioning allows non-breaking changes to public models — critical when other teams depend on your models.

### The Problem

You need to add a column to `dim_customers`, but renaming an existing column would break the marketing team's dashboard. Without versioning, you'd have to coordinate a synchronized migration.

### How It Works

```yaml
models:
  - name: dim_customers
    latest_version: 2
    access: public
    config:
      contract:
        enforced: true
    columns:
      - name: customer_id
        data_type: bigint
      - name: customer_name
        data_type: varchar
      - name: customer_segment
        data_type: varchar

    versions:
      - v: 1
        # Uses the default columns above
        # Deprecated — will be removed in Q3

      - v: 2
        columns:
          - include: all
          - name: loyalty_tier    # New column in v2
            data_type: varchar
```

```sql
-- Consumers can pin to a version
SELECT * FROM {{ ref('dim_customers', v=1) }}   -- Old version
SELECT * FROM {{ ref('dim_customers') }}         -- Latest (v2)
```

### Deprecation

```yaml
versions:
  - v: 1
    deprecation_date: 2024-09-01
    # dbt will warn consumers after this date
```

---

## 6.5 The Semantic Layer & MetricFlow (Deep Dive)

The dbt Semantic Layer centralizes metric definitions so every BI tool computes metrics the same way.

### Why It Matters

Without a semantic layer:
- Looker defines "revenue" as `SUM(amount) WHERE status = 'completed'`
- Tableau defines "revenue" as `SUM(amount) WHERE status IN ('completed', 'shipped')`
- An analyst's ad-hoc SQL uses yet another definition
- Nobody agrees on the numbers

### Defining Semantic Models

```yaml
# models/semantic/sem_orders.yml
semantic_models:
  - name: orders
    defaults:
      agg_time_dimension: ordered_at
    model: ref('fct_orders')

    entities:
      - name: order
        type: primary
        expr: order_id
      - name: customer
        type: foreign
        expr: customer_id

    dimensions:
      - name: ordered_at
        type: time
        type_params:
          time_granularity: day
      - name: order_status
        type: categorical
      - name: order_tier
        type: categorical

    measures:
      - name: order_total
        agg: sum
        expr: total_amount
      - name: order_count
        agg: count
        expr: order_id
      - name: avg_order_value
        agg: average
        expr: total_amount
      - name: unique_customers
        agg: count_distinct
        expr: customer_id
```

### Defining Metrics

```yaml
# models/semantic/metrics.yml
metrics:
  - name: revenue
    description: "Total revenue from completed orders"
    type: simple
    type_params:
      measure: order_total
    filter:
      - "{{ Dimension('order__order_status') }} = 'completed'"

  - name: revenue_per_customer
    description: "Average revenue per unique customer"
    type: derived
    type_params:
      expr: revenue / unique_customers
      metrics:
        - name: revenue
        - name: unique_customers

  - name: revenue_growth
    description: "Month-over-month revenue growth"
    type: derived
    type_params:
      expr: (current_revenue - previous_revenue) / previous_revenue
      metrics:
        - name: revenue
          offset_window: 1 month
          alias: previous_revenue
        - name: revenue
          alias: current_revenue
```

### Querying the Semantic Layer

BI tools that integrate with the dbt Semantic Layer (Tableau, Hex, Mode, Google Sheets) can query metrics directly. For ad-hoc use:

```bash
mf query --metrics revenue,order_count --group-by ordered_at__month --where "ordered_at >= '2024-01-01'"
```

MetricFlow generates the optimal SQL, handling joins, filters, and time grain conversions automatically.

---

## 6.6 Advanced Incremental Patterns

### Micro-Batch Strategy (dbt 1.9+)

Instead of processing all new rows in one query, micro-batch breaks the incremental load into time-based batches:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='occurred_at',
    begin='2023-01-01',
    batch_size='day',
    lookback=3
) }}

SELECT
    event_id,
    user_id,
    event_type,
    occurred_at
FROM {{ ref('stg_events') }}
```

Benefits:
- Each batch is an independent query — if one fails, others still succeed
- Automatic retry of failed batches
- Better warehouse resource utilization (smaller, more predictable queries)
- Built-in lookback for late-arriving data

### Late-Arriving Data Handling

```sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='merge'
) }}

SELECT * FROM {{ ref('stg_events') }}

{% if is_incremental() %}
    -- Lookback window: re-process last 6 hours to catch late arrivals
    WHERE occurred_at >= (
        SELECT DATEADD('hour', -6, MAX(occurred_at))
        FROM {{ this }}
    )
{% endif %}
```

The trade-off: wider lookback = more reprocessing cost but fewer missed rows. Tune based on your source system's latency characteristics.

### Handling Deletes in Incremental Models

Standard incremental models don't handle source deletes. Options:

```sql
-- Option 1: Soft delete via full outer join
{{ config(
    materialized='incremental',
    unique_key='customer_id',
    incremental_strategy='merge'
) }}

SELECT
    s.customer_id,
    s.customer_name,
    s.email,
    CASE
        WHEN s.customer_id IS NULL THEN TRUE
        ELSE FALSE
    END AS is_deleted,
    CURRENT_TIMESTAMP() AS _dbt_updated_at
FROM {{ ref('stg_customers') }} AS s

{% if is_incremental() %}
    -- Also include existing rows not in source (deleted)
    FULL OUTER JOIN {{ this }} AS t
        ON s.customer_id = t.customer_id
    WHERE s._etl_loaded_at > (SELECT MAX(_dbt_updated_at) FROM {{ this }})
        OR (s.customer_id IS NULL AND t.is_deleted = FALSE)
{% endif %}
```

```sql
-- Option 2: Use snapshots for delete tracking, incremental for current state
-- The snapshot captures the delete event; the incremental model filters to active records
```

---

## 6.7 Operational Best Practices

### Project Structure for Large Teams

```
dbt_project/
├── models/
│   ├── staging/
│   │   ├── ecommerce/          # One folder per source system
│   │   │   ├── _ecommerce__sources.yml
│   │   │   ├── _ecommerce__models.yml
│   │   │   ├── stg_ecommerce__orders.sql
│   │   │   └── stg_ecommerce__customers.sql
│   │   └── payments/
│   │       ├── _payments__sources.yml
│   │       └── stg_payments__transactions.sql
│   ├── intermediate/
│   │   ├── finance/
│   │   └── marketing/
│   └── marts/
│       ├── finance/
│       │   ├── _finance__models.yml
│       │   └── fct_revenue.sql
│       └── marketing/
│           ├── _marketing__models.yml
│           └── fct_campaigns.sql
├── tests/
│   ├── generic/                # Custom generic tests
│   │   └── test_positive_value.sql
│   └── singular/               # Model-specific tests
│       └── assert_revenue_reconciles.sql
├── macros/
│   ├── _macros.yml             # Macro documentation
│   ├── generate_schema_name.sql
│   └── utils/
│       ├── cents_to_dollars.sql
│       └── safe_divide.sql
├── seeds/
├── snapshots/
├── analyses/                   # Ad-hoc SQL (not materialized)
├── dbt_project.yml
├── packages.yml
└── profiles.yml                # NOT committed (in .gitignore)
```

### YAML File Naming Convention

Use underscore-prefixed YAML files colocated with their models:
- `_sources.yml` or `_{source_name}__sources.yml` for source definitions
- `_models.yml` or `_{domain}__models.yml` for model properties
- This keeps documentation close to the code it describes

### CI/CD Pipeline Structure

```yaml
# .github/workflows/dbt-ci.yml (simplified)
on:
  pull_request:
    branches: [main]

jobs:
  dbt-ci:
    steps:
      - uses: actions/checkout@v4

      - name: Install dbt
        run: pip install dbt-snowflake

      - name: Install packages
        run: dbt deps

      - name: Lint SQL
        run: sqlfluff lint models/ --dialect snowflake

      - name: Run changed models
        run: |
          dbt build \
            --select state:modified+ \
            --defer \
            --state ./prod-artifacts/ \
            --target ci
        env:
          DBT_USER: ${{ secrets.DBT_CI_USER }}
          DBT_PASSWORD: ${{ secrets.DBT_CI_PASSWORD }}

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: dbt-artifacts
          path: target/
```

### Monitoring & Alerting

| What to Monitor | How | Alert Threshold |
|----------------|-----|----------------|
| Source freshness | `dbt source freshness` | Per-source SLA |
| Test failures | `dbt build` exit code + `run_results.json` | Any error-severity failure |
| Model run time | `run_results.json` timing | >2x historical average |
| Row count changes | `dbt_expectations.expect_table_row_count_to_be_between` | >20% deviation |
| Warehouse cost | Warehouse query history | Per-run cost threshold |

### Elementary (Open-Source dbt Monitoring)

```yaml
# packages.yml
packages:
  - package: elementary-data/elementary
    version: [">=0.13.0", "<0.14.0"]
```

Elementary provides:
- Automated anomaly detection on row counts, freshness, and schema changes
- A self-hosted dashboard for dbt observability
- Slack/email alerts on test failures and anomalies
- Historical test result tracking

---

## 6.8 Custom Generic Tests

Beyond built-in tests, you can create reusable test macros:

```sql
-- tests/generic/test_positive_value.sql
{% test positive_value(model, column_name) %}

SELECT {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < 0

{% endtest %}
```

Usage in YAML:
```yaml
models:
  - name: fct_orders
    columns:
      - name: order_amount
        tests:
          - positive_value
```

### Parameterized Custom Test

```sql
-- tests/generic/test_value_in_range.sql
{% test value_in_range(model, column_name, min_val, max_val) %}

SELECT {{ column_name }}
FROM {{ model }}
WHERE {{ column_name }} < {{ min_val }}
   OR {{ column_name }} > {{ max_val }}

{% endtest %}
```

```yaml
columns:
  - name: discount_pct
    tests:
      - value_in_range:
          min_val: 0
          max_val: 100
```

---

## 6.9 Analyses & Exposures — Often Overlooked Features

### Analyses

SQL files in the `analyses/` directory are compiled by dbt but NOT materialized. Use them for:
- Ad-hoc investigative queries that benefit from `ref()` and Jinja
- Audit queries you run periodically but don't want as permanent models
- SQL templates for data consumers

```sql
-- analyses/monthly_revenue_by_segment.sql
SELECT
    DATE_TRUNC('month', ordered_at) AS revenue_month,
    customer_segment,
    SUM(total_amount) AS monthly_revenue
FROM {{ ref('fct_orders') }} f
JOIN {{ ref('dim_customers') }} c ON f.customer_id = c.customer_id
WHERE ordered_at >= '{{ var("analysis_start_date", "2024-01-01") }}'
GROUP BY 1, 2
ORDER BY 1, 2
```

Compile with `dbt compile`, then find the rendered SQL in `target/compiled/analyses/`.

### Exposures (Expanded)

Beyond basic exposure definitions (covered in 04), exposures support maturity levels and tags:

```yaml
exposures:
  - name: executive_kpi_dashboard
    type: dashboard
    maturity: high           # high | medium | low
    url: https://looker.company.com/dashboards/42
    description: "C-suite weekly KPI review dashboard"
    depends_on:
      - ref('fct_daily_revenue')
      - ref('dim_customers')
      - metric('revenue')     # Can depend on semantic layer metrics
    owner:
      name: Analytics Team
      email: analytics@company.com
    tags: ['tier-1', 'executive']
    meta:
      sla: "99.9%"
      refresh_frequency: "hourly"
```

Exposures appear in the dbt docs DAG, making it visible which downstream consumers are affected by model changes.

---

## 6.10 dbt Artifacts & Programmatic Access

dbt generates JSON artifacts that power metadata-driven workflows:

| Artifact | Generated By | Contains |
|----------|-------------|----------|
| `manifest.json` | `dbt parse`, `dbt compile`, `dbt run`, `dbt build` | Full project graph: models, tests, sources, macros, exposures, metrics |
| `run_results.json` | `dbt run`, `dbt test`, `dbt build` | Execution results: timing, status, rows affected, failures |
| `catalog.json` | `dbt docs generate` | Column-level metadata from the warehouse (types, stats) |
| `sources.json` | `dbt source freshness` | Source freshness check results |

### Programmatic Use Cases

- **Custom lineage tools**: Parse `manifest.json` to build lineage graphs in DataHub, Atlan, or custom UIs
- **Cost monitoring**: Parse `run_results.json` to track per-model execution time and correlate with warehouse costs
- **Automated documentation**: Extract model/column descriptions from `manifest.json` for data catalogs
- **Slim CI**: Compare current `manifest.json` against production to identify changed models
- **Alerting**: Parse `run_results.json` for failures and send to Slack/PagerDuty

```python
# Example: Parse run_results.json for slow models
import json

with open("target/run_results.json") as f:
    results = json.load(f)

for result in results["results"]:
    if result["execution_time"] > 300:  # > 5 minutes
        print(f"SLOW: {result['unique_id']} took {result['execution_time']:.0f}s")
```

---

*Next: [07-snowflake-end-to-end.md](./07-snowflake-end-to-end.md) — Complete end-to-end guide for dbt + Snowflake.*
