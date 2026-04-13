# 1. The Core Philosophy & Paradigm Shift

---

## 1.1 ETL vs. ELT — The Architectural Inversion

If you've built pipelines with PySpark or Flink, you're intimately familiar with the ETL pattern: data is **Extracted** from sources, **Transformed** in a processing engine (your Spark cluster, your Flink job), and then **Loaded** into a target store. The transformation happens *outside* the warehouse, in application code you own and operate.

ELT inverts the middle step:

| Dimension | ETL | ELT |
|-----------|-----|-----|
| Transform location | External compute (Spark, Flink, custom code) | Inside the warehouse |
| Transform language | Python, Scala, Java | SQL (+ Jinja templating) |
| Compute scaling | You manage cluster sizing | Warehouse handles it |
| Data movement | Data moves to compute | Compute comes to data |
| Schema enforcement | At transform time | At load time (schema-on-read) |
| Reprocessing cost | Re-run entire pipeline | Re-run a SQL model |

The ELT shift became viable because modern cloud warehouses (Snowflake, BigQuery, Redshift, Databricks SQL) offer:
- Elastic, on-demand compute that scales independently of storage
- Columnar storage with aggressive compression
- MPP (Massively Parallel Processing) query engines that handle terabyte-scale transforms natively

The key insight: **if your warehouse can run the transformation faster and cheaper than an external cluster, why move the data out?**

### Where ETL Still Wins

ELT is not a universal replacement. ETL remains superior when:
- Transformations require complex procedural logic (ML feature engineering, graph traversals)
- You need streaming/real-time processing (Flink's wheelhouse)
- Source systems require heavy cleansing before loading (PII scrubbing, format normalization)
- You're working with unstructured data that needs parsing before it can be tabular

The pragmatic answer: most organizations use **both**. ELT handles the analytical transformation layer; ETL handles ingestion and complex pre-processing.

---

## 1.2 dbt as the "T" in ELT

dbt occupies a very specific niche: it is **exclusively** the transformation layer. It does not extract. It does not load. It assumes raw data already exists in your warehouse.

```
  Fivetran / Airbyte / Stitch / Custom Scripts
  ─────────────────────────────────────────────
           Extract & Load (EL)
  ─────────────────────────────────────────────
                    │
                    ▼
            ┌──────────────┐
            │   RAW LAYER   │  (landing zone in your warehouse)
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │     dbt       │  ← SELECT-only transformations
            └──────┬───────┘
                   │
                   ▼
            ┌──────────────┐
            │  MART LAYER   │  (consumption-ready tables/views)
            └──────────────┘
                   │
                   ▼
            Looker / Tableau / Metabase / Downstream consumers
```

What dbt actually does at execution time:

1. **Reads** your `.sql` model files (which are Jinja-templated SELECT statements)
2. **Compiles** Jinja into pure SQL
3. **Resolves** the dependency graph (DAG) by parsing `ref()` and `source()` calls
4. **Executes** the compiled SQL against your warehouse in dependency order
5. **Materializes** results as views, tables, or incremental inserts depending on configuration

dbt never issues `INSERT INTO ... VALUES`, `UPDATE`, or `DELETE` on your behalf in the traditional sense. Every model is a `SELECT` statement. dbt wraps that SELECT in the appropriate DDL/DML (`CREATE VIEW AS`, `CREATE TABLE AS`, or `MERGE INTO` for incrementals).

### The Analogy for Spark/Flink Engineers

Think of dbt models as **lazy transformations in a DAG** — conceptually identical to Spark's logical plan or Flink's dataflow graph. The difference:
- In Spark, the DAG is defined in Python/Scala and executed on a Spark cluster
- In dbt, the DAG is defined in SQL files with `ref()` calls and executed on the warehouse's query engine

The `ref()` function is dbt's equivalent of a DataFrame reference. When you write `{{ ref('stg_orders') }}`, you're declaring a dependency, just like `orders_df = spark.read.table("stg_orders")` declares one in PySpark.

---

## 1.3 Software Engineering Best Practices for SQL

This is dbt's most transformative contribution. Before dbt, SQL transformations in most organizations looked like:
- Hundreds of stored procedures with no version control
- Copy-pasted SQL across scripts with no DRY principle
- No testing beyond "does it run without errors?"
- Documentation living in someone's head or a stale Confluence page
- Deployments via manual execution or fragile shell scripts

dbt introduces the full software engineering lifecycle to SQL:

### Version Control

Every dbt project is a Git repository. Models, tests, macros, and configurations are all files that go through pull requests, code reviews, and branch-based development.

```
my_dbt_project/
├── models/
│   ├── staging/
│   │   ├── stg_orders.sql
│   │   └── stg_customers.sql
│   └── marts/
│       └── fct_order_summary.sql
├── tests/
├── macros/
├── dbt_project.yml
└── profiles.yml
```

### CI/CD Integration

dbt projects integrate naturally with CI/CD pipelines:
- **On PR**: `dbt build --select state:modified+` runs only changed models and their downstream dependents against a CI schema
- **On merge to main**: Full `dbt build` against production
- dbt Cloud offers built-in CI with "Slim CI" — only building what changed

### DRY (Don't Repeat Yourself)

Three mechanisms enforce DRY:
1. **`ref()` function**: Reference models by name, never hardcode schema/table names
2. **Macros**: Reusable Jinja functions for repeated SQL patterns
3. **Packages**: Community-maintained macro libraries (e.g., `dbt_utils`, `dbt_expectations`)

### Testing as a First-Class Citizen

Every model can have declarative tests defined in YAML:

```yaml
models:
  - name: fct_order_summary
    columns:
      - name: order_id
        tests:
          - unique
          - not_null
      - name: total_amount
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 1000000
```

These compile to SQL queries that return rows only when the assertion fails. Zero rows = test passes.

### Auto-Generated Documentation

dbt generates a full documentation site from your YAML descriptions and model SQL, including:
- An interactive DAG visualization
- Column-level descriptions
- Test coverage
- Source freshness information

Run `dbt docs generate && dbt docs serve` and you have a living data catalog.

---

## 1.4 The Mental Model Shift

Coming from imperative pipeline code (Python/Scala), the biggest shift is moving to **declarative transformations**:

| Imperative (Spark/Flink) | Declarative (dbt) |
|--------------------------|-------------------|
| `df.filter(col("status") == "active")` | `WHERE status = 'active'` |
| `df.groupBy("customer_id").agg(sum("amount"))` | `GROUP BY customer_id` |
| `df.join(other_df, "key")` | `JOIN {{ ref('other_model') }} USING (key)` |
| You define *how* to compute | You define *what* to compute |
| You manage parallelism | Warehouse manages parallelism |
| You handle state & checkpointing | Warehouse handles it |

You're not giving up power — you're delegating execution to an engine purpose-built for analytical queries. Your job shifts from "how do I process this efficiently?" to "how do I model this data correctly?"

---

## 1.5 dbt Core vs. dbt Cloud

| Aspect | dbt Core | dbt Cloud |
|--------|----------|-----------|
| Cost | Free, open-source | Paid SaaS |
| Execution | CLI on your machine / CI runner | Managed execution environment |
| Scheduling | External (Airflow, Dagster, Prefect) | Built-in scheduler |
| IDE | Your editor + terminal | Browser-based IDE |
| CI/CD | You configure it | Built-in Slim CI |
| Metadata | Local `manifest.json` | Hosted metadata API (Discovery API) |
| Best for | Teams with existing orchestration | Teams wanting managed analytics engineering |

Both use the same project structure, same SQL, same Jinja. dbt Cloud is a convenience layer, not a different product.

---

*Next: [02-project-anatomy.md](./02-project-anatomy.md) — Every component of a dbt project, dissected.*
