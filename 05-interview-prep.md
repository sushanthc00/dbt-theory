# 5. dbt Interview Preparation — Senior/Staff Data Engineer Level

> Calibrated for ~4 years experience, transitioning from distributed systems (PySpark/Flink) and relational databases (Oracle SQL) into analytics engineering. Covers conceptual depth, scenario-based problem solving, and hands-on SQL/Jinja challenges.

---

## Section A: Core Concepts & Philosophy

### A1. Foundational Questions

**Q: Explain the difference between ETL and ELT. Why has the industry shifted toward ELT?**

Expected depth:
- ETL transforms data outside the warehouse (Spark, Flink, custom code); ELT transforms inside the warehouse using SQL
- The shift happened because cloud warehouses (Snowflake, BigQuery, Redshift) decouple compute from storage and offer elastic, on-demand processing
- ELT reduces data movement, simplifies architecture, and leverages the warehouse's MPP engine for transformations
- Mention that ETL still wins for streaming, ML feature engineering, and heavy pre-processing of unstructured data
- A mature org typically uses both: ETL for ingestion/complex processing, ELT for analytical transformations

**Q: What is dbt and what problem does it solve? What does it NOT do?**

Expected depth:
- dbt is the transformation layer in ELT — it compiles Jinja-templated SQL, resolves a DAG of dependencies, and executes SELECT-based transformations inside the warehouse
- It does NOT extract or load data. It assumes raw data already exists in the warehouse
- It brings software engineering practices to SQL: version control, CI/CD, testing, documentation, DRY principles
- Every model is a SELECT statement; dbt wraps it in appropriate DDL (CREATE VIEW/TABLE AS, MERGE INTO)

**Q: How does dbt build a DAG? What happens if you have a circular dependency?**

Expected depth:
- dbt parses all `.sql` files, extracts `ref()` and `source()` calls, and builds a directed acyclic graph
- Execution order is determined by topological sort — models are run in dependency order
- Circular dependencies are detected at parse time and dbt raises an error before any execution
- Independent models (no dependency between them) run in parallel up to the `threads` limit

**Q: What is the `ref()` function and why is it critical?**

Expected depth:
- `ref()` does two things: resolves a model name to its fully qualified database object (handling schema, database, alias) AND declares a dependency edge in the DAG
- It enables environment isolation — `ref('stg_orders')` resolves to `dev.dbt_alice.stg_orders` in dev and `prod.analytics.stg_orders` in prod
- Cross-project refs (dbt Mesh, 1.6+) allow referencing models across dbt projects
- Never hardcode table names; always use `ref()` or `source()`

**Q: Compare `ref()` and `source()`. When do you use each?**

- `source()` references raw tables NOT managed by dbt (loaded by Fivetran, Airbyte, etc.)
- `ref()` references models managed by dbt
- `source()` enables freshness monitoring and creates lineage from raw → staging
- Both contribute to the DAG, but `source()` nodes are leaf nodes (no upstream dbt dependencies)

---

### A2. Architecture & Design Questions

**Q: Describe the staging → intermediate → marts modeling pattern. Why this layering?**

Expected depth:
- Staging: 1:1 mapping from source tables. Only renaming, casting, basic cleansing. Materialized as views. Convention: `stg_{source}__{table}`
- Intermediate: Business logic, joins, aggregations, pivots. Materialized as ephemeral or views. Convention: `int_{description}`
- Marts: Consumption-ready tables for BI/analytics. Fact and dimension tables or OBTs. Materialized as tables or incrementals. Convention: `fct_*`, `dim_*`
- This layering provides separation of concerns, testability at each layer, and reusability (multiple marts can reference the same intermediate models)

**Q: When would you choose a Star Schema vs. One Big Table (OBT) in dbt?**

- Star Schema: When you have multiple fact tables sharing dimensions, when storage efficiency matters, when BI tools are optimized for star schemas (Looker's symmetric aggregates), when you need to update dimensions independently
- OBT: When query simplicity is paramount, when you have a single primary fact, when BI users write ad-hoc SQL, when columnar storage makes denormalization cheap
- In practice, many teams use a hybrid: star schema for the core model, OBTs for specific high-traffic dashboards

**Q: How would you model a multi-tenant SaaS application in dbt?**

- Use a `tenant_id` column throughout the DAG
- Create a macro that filters by tenant in dev (for speed) but includes all tenants in prod
- Consider separate schemas per tenant if isolation is required
- Use dbt variables or environment variables to control tenant filtering
- Test referential integrity within tenant boundaries

**Q: You have a 500-line SQL model. How do you refactor it?**

- Break it into intermediate models, each handling one logical step (join, filter, aggregate)
- Each intermediate model should be independently testable
- Use ephemeral materialization for intermediates that don't need to be queried directly
- The mart model becomes a simple join/union of intermediate outputs
- Rule of thumb: if a CTE is reused or exceeds ~50 lines, extract it into its own model

---

## Section B: Materializations — Deep Dive

### B1. Materialization Selection

**Q: Compare all four materializations. When do you use each?**

| Materialization | Creates | Rebuild Cost | Best For | Avoid When |
|----------------|---------|-------------|----------|------------|
| `view` | VIEW | Instant (no data stored) | Staging, light transforms, < 100M rows | Heavy aggregations queried frequently |
| `table` | TABLE | Full rebuild every run | Marts, heavy aggregations, frequently queried | Very large tables that change incrementally |
| `incremental` | TABLE | Only new/changed rows | Large fact tables, event streams, append-heavy | Small tables (overhead not worth it) |
| `ephemeral` | CTE (no object) | N/A | Helper logic, avoid object clutter | Deep nesting (>2 levels), debugging needed |

**Q: Explain how incremental models work under the hood. What SQL does dbt generate?**

Expected depth:
- First run: `CREATE TABLE AS SELECT ...` (full dataset, `is_incremental()` returns false)
- Subsequent runs: dbt generates a MERGE/INSERT/DELETE+INSERT depending on the strategy
- `{{ this }}` refers to the existing target table
- `is_incremental()` returns true only when: the model is materialized as incremental AND the target table already exists AND `--full-refresh` is NOT passed
- The `unique_key` determines how dbt identifies existing rows for merge/upsert

**Q: What are the different incremental strategies? When would you pick each?**

- `append`: INSERT only, no dedup. Use for immutable event logs where duplicates are impossible
- `merge`: UPSERT on `unique_key`. Use for mutable data or late-arriving facts. Most common
- `delete+insert`: DELETE matching keys then INSERT. Use on Redshift/Postgres where MERGE is expensive or unavailable
- `insert_overwrite`: Replace entire partitions. Use on BigQuery/Databricks for partition-aligned fact tables

**Q: What happens if your incremental model misses rows? How do you recover?**

- Missed rows are permanently absent from the target table (they won't be picked up on future runs if they fall outside the incremental filter)
- Recovery: `dbt run --select model_name --full-refresh` rebuilds from scratch
- Prevention: use overlapping windows (e.g., `WHERE event_time > MAX(event_time) - INTERVAL '2 hours'`) to catch late-arriving data
- Schedule periodic full refreshes (weekly/monthly) as a safety net

**Q: Can you have a composite `unique_key` in an incremental model?**

- Yes. Pass a list: `unique_key=['order_id', 'line_item_id']`
- Or use `dbt_utils.generate_surrogate_key()` to create a single composite key
- The generated MERGE statement will match on all columns in the composite key

---

### B2. Scenario-Based Materialization Questions

**Q: You have a 2TB fact table that grows by ~50M rows/day. How do you model this in dbt?**

- Incremental materialization with `merge` or `insert_overwrite` strategy
- Partition by date (if warehouse supports it) for efficient pruning
- Use a lookback window in the incremental filter to handle late-arriving data
- Set `on_schema_change: 'append_new_columns'` for forward compatibility
- Schedule weekly full refreshes to correct any drift
- Monitor row counts and data freshness with tests

```sql
{{ config(
    materialized='incremental',
    unique_key='event_id',
    incremental_strategy='insert_overwrite',
    partition_by={
        "field": "event_date",
        "data_type": "date",
        "granularity": "day"
    },
    on_schema_change='append_new_columns'
) }}
```

**Q: A downstream dashboard is slow because it queries a view that joins 8 tables. What do you do?**

- Change the materialization from `view` to `table` (pre-compute the join)
- If the table is large and append-heavy, use `incremental`
- Consider creating an OBT specifically for that dashboard
- Add clustering/sort keys on the columns the dashboard filters on
- Profile the compiled SQL to identify the expensive join

---

## Section C: Testing & Data Quality

**Q: What types of tests does dbt support? Give examples of each.**

- Generic (schema) tests: Declared in YAML. Built-in: `unique`, `not_null`, `accepted_values`, `relationships`. Package-based: `dbt_expectations.expect_column_values_to_be_between`
- Singular (custom) tests: SQL files in `tests/` that return failing rows. Any row returned = test failure
- Unit tests (dbt 1.8+): Test model logic with mocked inputs, without hitting the warehouse

**Q: How would you test referential integrity between a fact and dimension table?**

```yaml
models:
  - name: fct_orders
    columns:
      - name: customer_id
        tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
```

This compiles to a LEFT JOIN that returns any `customer_id` in `fct_orders` that doesn't exist in `dim_customers`.

**Q: What are model contracts? When would you enforce them?**

- Contracts (dbt 1.5+) enforce column names, data types, and constraints at build time
- If the model's SELECT doesn't match the contract, the build fails before writing data
- Use on mart models that downstream teams depend on — prevents breaking changes
- Analogous to schema enforcement in Spark's StructType or a database DDL constraint

**Q: How do you handle test severity? When would you use `warn` vs `error`?**

- `error` (default): Fails the build. Use for critical assertions (primary key uniqueness, not-null on required fields)
- `warn`: Logs a warning but doesn't fail. Use for known data quality issues being addressed, or soft constraints
- `error_if` / `warn_if` thresholds: `error_if: ">100"` — only fail if more than 100 rows violate. Useful for large tables with known edge cases

**Q: Describe a data quality testing strategy for a production dbt project.**

- Every primary key: `unique` + `not_null`
- Every foreign key: `relationships` test
- Every enum/status column: `accepted_values`
- Critical business metrics: singular tests for reconciliation (mart total vs source total)
- Source freshness checks before every build
- Model contracts on all mart models
- `dbt_expectations` for statistical tests (row count ranges, value distributions)
- Run `dbt build` (not `dbt run` then `dbt test`) to interleave tests with model execution

---

## Section D: Snapshots & SCD

**Q: How does dbt implement SCD Type 2? Explain the mechanics.**

- Snapshots compare current source data against the latest snapshot state
- Changed rows: the old version gets `dbt_valid_to` set to current timestamp; a new row is inserted with `dbt_valid_to = NULL`
- Added columns: `dbt_scd_id`, `dbt_updated_at`, `dbt_valid_from`, `dbt_valid_to`
- Two strategies: `timestamp` (compares an `updated_at` column) and `check` (compares specified column values)
- `invalidate_hard_deletes=True` closes records for rows deleted from the source

**Q: When would you use `check` strategy vs `timestamp` strategy?**

- `timestamp`: When the source has a reliable, always-updated `updated_at` column. More efficient (only compares one column)
- `check`: When there's no reliable timestamp, or when the timestamp doesn't update for all change types. Compares actual column values but is more expensive (compares N columns)

**Q: A snapshot table has grown to 500M rows and queries are slow. How do you optimize?**

- Add clustering/sort keys on `dbt_valid_from` and the business key
- Create a view that filters to current records only (`WHERE dbt_valid_to IS NULL`)
- Partition by `dbt_valid_from` date if the warehouse supports it
- Consider archiving very old historical records to a separate table
- If only current state is needed for most queries, maintain a separate `dim_` table alongside the snapshot

---

## Section E: Macros, Jinja & Advanced Patterns

**Q: Write a macro that generates a CASE statement for mapping status codes to labels.**

```sql
{% macro map_status(column_name, mapping, default='Unknown') %}
    CASE {{ column_name }}
        {% for code, label in mapping.items() %}
        WHEN '{{ code }}' THEN '{{ label }}'
        {% endfor %}
        ELSE '{{ default }}'
    END
{% endmacro %}
```

Usage:
```sql
SELECT
    order_id,
    {{ map_status('status_code', {'P': 'Pending', 'S': 'Shipped', 'D': 'Delivered', 'C': 'Cancelled'}) }} AS order_status
FROM {{ ref('stg_orders') }}
```

**Q: Explain the `generate_schema_name` macro. Why would you override it?**

- Controls which schema a model lands in
- Default behavior: if a custom schema is set, it's appended to the target schema (e.g., `dbt_alice_staging`)
- Override to make prod use clean schema names (`staging`, `marts`) while dev uses prefixed names (`dbt_alice_staging`)
- This is one of the most commonly customized macros in production dbt projects

**Q: What is `run_query()` and when would you use it?**

- Executes SQL at compile time (not at model execution time) and returns results as an Agate table
- Use for dynamic code generation: query the warehouse for column names, distinct values, or metadata, then use the results in Jinja loops
- Must be wrapped in `{% if execute %}` guard because `run_query()` doesn't execute during parsing phase
- Example: dynamically pivot columns based on actual data values

**Q: What's the difference between `{{ }}`, `{% %}`, and `{# #}` in Jinja?**

- `{{ }}`: Expression — evaluates and outputs a value (like Python's f-string `{}`)
- `{% %}`: Statement — control flow (if/for/set/macro), produces no output
- `{# #}`: Comment — stripped from compiled SQL entirely

**Q: How do you handle environment-specific logic in dbt?**

- `{{ target.name }}` — returns the current target (dev/staging/prod)
- `{{ target.schema }}` — returns the target schema
- `{{ var('my_var') }}` — project variables, overridable at runtime
- `{{ env_var('MY_SECRET') }}` — environment variables (for secrets)
- Combine these in Jinja conditionals for environment-specific behavior

---

## Section F: Workflow, CLI & Operations

**Q: What is the difference between `dbt run` and `dbt build`?**

- `dbt run`: Only materializes models (views/tables). Does NOT run tests, seeds, or snapshots
- `dbt build`: Runs seeds → models → tests → snapshots in DAG order, interleaving tests after each model. If a model's tests fail, downstream models are skipped
- `dbt build` is the recommended command for production deployments

**Q: Explain node selection syntax. How would you run only changed models and their downstream dependents?**

```bash
dbt build --select state:modified+
```

- `state:modified` selects models whose SQL or config has changed compared to a previous manifest
- The `+` suffix includes all downstream dependents
- Requires `--state` flag pointing to the previous manifest, or `--defer` to fall back to production for unselected models
- This is the foundation of "Slim CI" — only build what changed in a PR

**Q: How do you manage dev/staging/prod environments in dbt?**

- `profiles.yml` defines connection targets (dev, staging, prod) with different databases, schemas, warehouses, and credentials
- Dev: per-developer schema (`dbt_alice`), small warehouse, subset of data
- Staging: shared schema, medium warehouse, full data, mirrors prod config
- Prod: production schema, large warehouse, full data, deployed via CI/CD
- `generate_schema_name` macro controls schema naming per environment
- `--target` flag switches between environments

**Q: What is `--defer` and how does Slim CI work?**

- `--defer` tells dbt: "for any model I'm NOT building in this run, resolve `ref()` calls to the production version"
- Combined with `state:modified+`, this means CI only builds changed models but can still reference unchanged production tables
- Requires a production `manifest.json` as the comparison baseline
- Dramatically reduces CI build time and warehouse cost

**Q: How would you orchestrate dbt in production? Compare approaches.**

- Airflow: `BashOperator` or `dbt-airflow` provider. dbt is one task in a larger DAG. Good when dbt is part of a broader pipeline
- Dagster: Each dbt model becomes a software-defined asset. Best for asset-based orchestration with unified lineage
- Prefect: `prefect-dbt` integration. Good for Python-native teams
- dbt Cloud: Built-in scheduler with job definitions, Slim CI, and metadata API. Simplest option if you don't need external orchestration

---

## Section G: Performance & Optimization

**Q: A dbt run takes 3 hours. How do you diagnose and optimize?**

1. Check `target/run_results.json` for per-model timing — identify the slowest models
2. For slow models:
   - Review compiled SQL in `target/compiled/` — is the Jinja generating inefficient SQL?
   - Check materialization — should a view be a table? Should a table be incremental?
   - Profile the query in the warehouse's query planner (EXPLAIN)
   - Add clustering/sort keys, partitioning
3. Increase `threads` in `profiles.yml` to parallelize independent models
4. Use `dbt build --select tag:critical` to prioritize high-value models
5. Split into multiple dbt jobs (hourly for critical, daily for the rest)

**Q: How do you optimize incremental models for large datasets?**

- Use partition-aligned strategies (`insert_overwrite`) when possible
- Add a lookback window but keep it as small as data latency allows
- Use `unique_key` as a list for composite keys (avoids surrogate key computation)
- Ensure the incremental filter column is indexed/clustered in the warehouse
- Monitor for "incremental drift" — schedule periodic full refreshes

**Q: What is the impact of `threads` in `profiles.yml`?**

- Controls max parallel model executions
- Independent models (no dependency between them) run concurrently up to this limit
- Too low: underutilizes warehouse capacity, slow builds
- Too high: can overwhelm the warehouse with concurrent queries, cause queuing
- Rule of thumb: start with 4-8 for dev, 8-16 for prod, tune based on warehouse capacity

---

## Section H: Scenario-Based & System Design Questions

**Q: Design a dbt project for an e-commerce platform with orders, customers, products, and payments.**

Expected answer structure:
```
models/
├── staging/
│   ├── stg_ecommerce__orders.sql        (view)
│   ├── stg_ecommerce__customers.sql     (view)
│   ├── stg_ecommerce__products.sql      (view)
│   └── stg_ecommerce__payments.sql      (view)
├── intermediate/
│   ├── int_orders_enriched.sql          (ephemeral)
│   ├── int_customer_order_summary.sql   (ephemeral)
│   └── int_payment_methods_pivoted.sql  (ephemeral)
└── marts/
    ├── fct_orders.sql                   (incremental, merge)
    ├── dim_customers.sql                (table)
    ├── dim_products.sql                 (table)
    └── fct_daily_revenue.sql            (table)
```

Discuss: materialization choices, testing strategy, source freshness, snapshot for customer SCD, seeds for country codes.

**Q: You're migrating a legacy Oracle stored procedure pipeline to dbt. What's your approach?**

1. Map each stored procedure to understand its inputs, outputs, and logic
2. Identify the dependency order (which procs call which)
3. Create staging models for each source table the procs read from
4. Translate procedural logic to declarative SQL (CTEs replace temp tables, CASE replaces IF/ELSE)
5. Handle imperative patterns: loops become set-based operations, cursors become window functions
6. Create intermediate models for complex multi-step logic
7. Add tests at each layer to validate parity with the legacy output
8. Use `audit_helper.compare_relations` to verify the migration produces identical results
9. Run both systems in parallel during validation period

**Q: How would you implement data mesh principles using dbt?**

- dbt Mesh (1.6+): Cross-project `ref()` allows teams to own their own dbt projects while referencing each other's models
- Each domain team owns a dbt project with their staging, intermediate, and mart models
- Public models are exposed via `access: public` in YAML — these are the "data products"
- Model contracts enforce the interface between teams
- Model versions allow non-breaking changes to public models
- A shared `packages.yml` provides common macros and conventions

**Q: Your incremental model is producing duplicates. How do you debug?**

1. Check if `unique_key` is truly unique in the source data
2. Verify the incremental strategy matches your warehouse (e.g., `merge` requires warehouse support)
3. Check the compiled SQL in `target/run/` — is the MERGE statement correct?
4. Look for race conditions: is the source being updated while dbt runs?
5. Check for NULL values in `unique_key` columns (NULLs don't match in MERGE)
6. Add a `unique` test on the key column to catch duplicates early
7. If all else fails, `--full-refresh` and investigate the root cause

---

## Section I: dbt Cloud & Metadata

**Q: What are the key differences between dbt Core and dbt Cloud?**

| Aspect | dbt Core | dbt Cloud |
|--------|----------|-----------|
| Cost | Free, open-source | Paid SaaS |
| Execution | CLI, self-managed | Managed environment |
| Scheduling | External orchestrator | Built-in scheduler |
| CI/CD | Self-configured | Built-in Slim CI |
| IDE | Your editor | Browser-based IDE |
| Metadata | Local `manifest.json` | Discovery API, Explorer |

**Q: What is the dbt Semantic Layer?**

- Centralized metric definitions that BI tools can query directly
- Metrics are defined in YAML with dimensions, measures, and time grains
- Powered by MetricFlow (acquired by dbt Labs)
- Ensures "one source of truth" for business metrics across all BI tools
- Eliminates metric definition drift between Looker, Tableau, and ad-hoc SQL

**Q: What are exposures and why do they matter?**

- Exposures document downstream consumers (dashboards, ML models, applications) that depend on dbt models
- They create lineage edges from dbt models to external systems
- Enable impact analysis: "if I change this model, which dashboards break?"
- Defined in YAML with owner, URL, maturity, and `depends_on`

---

## Section J: Rapid-Fire / Short Answer

| Question | Key Points |
|----------|-----------|
| What file configures warehouse connections? | `profiles.yml` (outside project, NOT in version control) |
| What file configures the dbt project? | `dbt_project.yml` (project root, in version control) |
| What file declares packages? | `packages.yml` (project root) |
| How do you install packages? | `dbt deps` |
| How do you load CSV seed files? | `dbt seed` |
| How do you check source freshness? | `dbt source freshness` |
| What does `{{ this }}` refer to? | The current model's existing database object |
| What does `is_incremental()` return? | True if: incremental materialization + table exists + not full-refresh |
| How do you run only one model? | `dbt run --select model_name` |
| How do you run a model and all its dependents? | `dbt run --select model_name+` |
| How do you full-refresh an incremental model? | `dbt run --select model_name --full-refresh` |
| Where is compiled SQL stored? | `target/compiled/` |
| What is `manifest.json`? | Complete project graph — models, tests, sources, lineage |
| What is the default materialization? | `view` |
| Can dbt do streaming? | No. dbt is batch/micro-batch only. Use Flink/Spark Streaming for real-time |
| What language does dbt use? | SQL + Jinja2 templating (Python models available in 1.3+ but SQL is primary) |

---

## Section K: Take-Home / Coding Challenges

### Challenge 1: Write an Incremental Model

*"Write a dbt model for a `fct_page_views` fact table. It should be incremental, handle late-arriving data with a 3-hour lookback, use merge strategy, and partition by day."*

### Challenge 2: Write a Macro

*"Write a macro called `safe_divide(numerator, denominator, default)` that returns the division result or a default value when the denominator is zero or null."*

```sql
{% macro safe_divide(numerator, denominator, default=0) %}
    CASE
        WHEN {{ denominator }} IS NULL OR {{ denominator }} = 0
        THEN {{ default }}
        ELSE CAST({{ numerator }} AS DECIMAL(18,6)) / {{ denominator }}
    END
{% endmacro %}
```

### Challenge 3: Debug a Failing Test

*"The `unique` test on `order_id` in `fct_orders` is failing. Walk me through your debugging process."*

Expected steps:
1. Run `dbt test --select fct_orders` to see the failure details
2. Check the compiled test SQL in `target/compiled/` — it's a `GROUP BY order_id HAVING COUNT(*) > 1`
3. Run the compiled SQL manually to see which `order_id` values are duplicated
4. Trace upstream: is the duplication in `stg_orders` or introduced by a join in an intermediate model?
5. If it's a fan-out join, add a dedup CTE or fix the join condition
6. If it's in the source, add a dedup in staging using `ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) = 1`

### Challenge 4: Design a Testing Strategy

*"You're building a financial reporting mart. What tests would you put in place?"*

Expected answer:
- Primary keys: `unique` + `not_null` on every key
- Foreign keys: `relationships` between facts and dimensions
- Reconciliation: singular test comparing mart totals to source totals (within tolerance)
- Range checks: `dbt_expectations.expect_column_values_to_be_between` on monetary amounts
- Row count: `dbt_expectations.expect_table_row_count_to_be_between` to catch data drops
- Freshness: source freshness checks with tight thresholds
- Contracts: enforced on all mart models to prevent schema drift

---

*Back to: [README.md](./README.md) — Guide index and quick mental model.*
