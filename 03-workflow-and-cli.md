# 3. The dbt Workflow & CLI

---

## 3.1 The Standard Development Loop

```
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │   Write          Compile         Run          Test       │
  │   (SQL/Jinja) → (Pure SQL) → (Execute) → (Validate) ──┐ │
  │       ▲                                                │ │
  │       │                                                │ │
  │       └──── Fix ◄──── Debug ◄──── Investigate ◄────────┘ │
  │                                                          │
  │   Then: Document → Commit → PR → Review → Deploy         │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```

### Step-by-Step

1. **Write**: Create or modify a `.sql` model with Jinja templating
2. **Compile**: `dbt compile` — renders Jinja to pure SQL (check `target/compiled/` for output)
3. **Run**: `dbt run --select my_model` — executes the compiled SQL against your dev warehouse
4. **Test**: `dbt test --select my_model` — runs assertions against the materialized result
5. **Document**: Add/update YAML descriptions, then `dbt docs generate`
6. **Commit**: Standard Git workflow — branch, commit, push, PR

For rapid iteration, `dbt build --select my_model` combines run + test in a single command.

---

## 3.2 CLI Command Reference

### Core Execution Commands

| Command | What It Does | When to Use |
|---------|-------------|-------------|
| `dbt run` | Materializes models (CREATE/REPLACE views/tables) | Deploying transformations |
| `dbt test` | Executes all tests (schema + singular) | Validating data quality |
| `dbt build` | Runs models + tests + snapshots + seeds in DAG order | Standard deployment command |
| `dbt compile` | Renders Jinja → pure SQL without executing | Debugging Jinja, reviewing compiled SQL |
| `dbt seed` | Loads CSV files from `seeds/` into warehouse | Loading reference data |
| `dbt snapshot` | Executes snapshot logic (SCD Type 2) | Capturing historical state |

### Utility Commands

| Command | What It Does |
|---------|-------------|
| `dbt deps` | Installs packages from `packages.yml` |
| `dbt clean` | Removes `target/` and `dbt_packages/` directories |
| `dbt debug` | Tests warehouse connection and project configuration |
| `dbt docs generate` | Generates documentation artifacts (`manifest.json`, `catalog.json`) |
| `dbt docs serve` | Starts a local web server to browse generated docs |
| `dbt source freshness` | Checks if source data is stale based on `freshness` config |
| `dbt ls` (or `dbt list`) | Lists resources matching a selection criteria |
| `dbt parse` | Parses the project and validates syntax without executing |
| `dbt retry` | Re-runs only the nodes that failed in the previous run |

### What Happens Under the Hood

**`dbt run` execution sequence:**
```
1. Parse all .sql and .yml files
2. Build the dependency graph (DAG)
3. Resolve ref() and source() to actual database objects
4. Compile Jinja templates to pure SQL
5. Determine execution order via topological sort
6. For each model (in order):
   a. Begin transaction
   b. Execute DDL (CREATE VIEW/TABLE AS ...)
   c. Commit transaction
   d. Log result (success/error/skip)
7. Write run results to target/run_results.json
```

**`dbt build` execution sequence:**
```
Same as above, but interleaves:
- Seeds first (if selected)
- Then models + tests in DAG order:
  Model A → Test A → Model B (depends on A) → Test B → ...
- Snapshots after models
```

This interleaving is critical: if Model A's tests fail, Model B (which depends on A) is skipped. This prevents cascading bad data.

---

## 3.3 Node Selection Syntax

dbt's selection syntax is powerful and essential for efficient development. Think of it as a query language for your DAG.

### Basic Selectors

```bash
# Single model
dbt run --select my_model

# Multiple models
dbt run --select model_a model_b model_c

# All models in a directory
dbt run --select staging

# All models in a subdirectory
dbt run --select staging.ecommerce
```

### Graph Operators

```bash
# Upstream dependencies (ancestors) — the "+" prefix
dbt run --select +fct_daily_revenue
# Runs: stg_orders → int_order_enriched → fct_daily_revenue

# Downstream dependents (descendants) — the "+" suffix
dbt run --select stg_orders+
# Runs: stg_orders → everything that depends on it

# Both directions
dbt run --select +fct_daily_revenue+

# Limit depth
dbt run --select 1+fct_daily_revenue    # Only 1 level upstream
dbt run --select fct_daily_revenue+1    # Only 1 level downstream
```

### Advanced Selectors

```bash
# By tag
dbt run --select tag:daily

# By materialization
dbt run --select config.materialized:incremental

# By source
dbt test --select source:raw_ecommerce

# By resource type
dbt ls --select resource_type:snapshot

# State-based (requires manifest from previous run)
dbt run --select state:modified       # Only changed models
dbt run --select state:modified+      # Changed models + downstream

# Intersection (AND logic)
dbt run --select tag:daily,config.materialized:table
# Models that are BOTH tagged 'daily' AND materialized as table

# Exclusion
dbt run --select marts --exclude fct_debug_table

# Union (OR logic — space-separated)
dbt run --select tag:daily tag:hourly
```

### Selection in CI/CD (Slim CI)

```bash
# Compare against production manifest and only build what changed
dbt build --select state:modified+ --defer --state ./prod-manifest/
```

`--defer` tells dbt: "for any model I'm NOT building, use the production version." This means your CI run only builds changed models but can still resolve `ref()` calls to unchanged production tables.

---

## 3.4 Managing Environments via `profiles.yml`

`profiles.yml` lives **outside** your dbt project (typically `~/.dbt/profiles.yml`) and contains warehouse connection details. It is NOT committed to version control.

```yaml
# ~/.dbt/profiles.yml
ecommerce_analytics:              # Must match 'profile' in dbt_project.yml
  target: dev                     # Default target when you run dbt

  outputs:
    dev:
      type: snowflake
      account: xy12345.us-east-1
      user: "{{ env_var('DBT_USER') }}"
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: transformer_dev
      database: analytics_dev
      warehouse: transforming_xs
      schema: "dbt_{{ env_var('DBT_USER') | lower }}"   # dbt_alice, dbt_bob
      threads: 4

    staging:
      type: snowflake
      account: xy12345.us-east-1
      user: "{{ env_var('DBT_USER') }}"
      password: "{{ env_var('DBT_PASSWORD') }}"
      role: transformer_staging
      database: analytics_staging
      warehouse: transforming_medium
      schema: analytics
      threads: 8

    prod:
      type: snowflake
      account: xy12345.us-east-1
      user: "{{ env_var('DBT_PROD_USER') }}"
      password: "{{ env_var('DBT_PROD_PASSWORD') }}"
      role: transformer_prod
      database: analytics
      warehouse: transforming_large
      schema: analytics
      threads: 16
```

### Key Concepts

**`target`**: The active environment. Switch with:
```bash
dbt run --target staging
dbt run --target prod
```

**`threads`**: Number of models dbt runs in parallel. Independent models (no dependency between them) execute concurrently up to this limit. Think of it as your DAG's parallelism degree.

**`schema`**: The default schema for models. The `generate_schema_name` macro (see Section 2.6) controls how custom schemas interact with this default.

**Environment isolation pattern:**
```
dev:    analytics_dev.dbt_alice.*        (per-developer schema)
staging: analytics_staging.analytics.*   (shared staging)
prod:   analytics.analytics.*            (production)
```

### Environment Variables

Use `env_var()` to keep secrets out of files:
```yaml
user: "{{ env_var('DBT_USER') }}"
password: "{{ env_var('DBT_PASSWORD') }}"
```

In CI/CD, these are injected as pipeline secrets. Locally, use a `.env` file or shell exports.

---

## 3.5 The `target/` Directory — Compiled Artifacts

After any dbt command, the `target/` directory contains:

```
target/
├── compiled/                    # Jinja-rendered pure SQL (for debugging)
│   └── ecommerce_analytics/
│       └── models/
│           └── marts/
│               └── fct_daily_revenue.sql
├── run/                         # SQL actually sent to warehouse (with DDL wrappers)
│   └── ecommerce_analytics/
│       └── models/
│           └── marts/
│               └── fct_daily_revenue.sql
├── manifest.json                # Full project graph (models, tests, sources, exposures)
├── run_results.json             # Results of the last run (timing, status, rows affected)
├── catalog.json                 # Column-level metadata (generated by dbt docs generate)
└── sources.json                 # Source freshness results
```

**`manifest.json`** is the most important artifact:
- Contains the entire DAG structure
- Used by `state:modified` selection for Slim CI
- Consumed by dbt Cloud's Discovery API
- Can be parsed programmatically for custom lineage tools

**Pro tip**: Always check `target/compiled/` when debugging. If your Jinja isn't producing the SQL you expect, the compiled output tells you exactly what dbt sent to the warehouse.

---

## 3.6 Hooks & Operations

### Hooks

Execute SQL before or after models run:

```yaml
# dbt_project.yml
on-run-start:
  - "CREATE SCHEMA IF NOT EXISTS {{ target.schema }}"
  - "{{ log('Starting dbt run at ' ~ run_started_at, info=True) }}"

on-run-end:
  - "GRANT SELECT ON ALL TABLES IN SCHEMA {{ target.schema }} TO ROLE analyst"

models:
  ecommerce_analytics:
    marts:
      +pre-hook:
        - "{{ log('Building: ' ~ this, info=True) }}"
      +post-hook:
        - "ALTER TABLE {{ this }} SET TAG governance.pii = 'false'"
```

**Hook types:**
- `on-run-start` / `on-run-end` — Project-level, runs once per invocation
- `pre-hook` / `post-hook` — Model-level, runs for each model

### Operations

Standalone SQL scripts that don't create models:

```sql
-- macros/grant_permissions.sql
{% macro grant_permissions() %}
    {% set sql %}
        GRANT USAGE ON SCHEMA {{ target.schema }} TO ROLE analyst;
        GRANT SELECT ON ALL TABLES IN SCHEMA {{ target.schema }} TO ROLE analyst;
    {% endset %}
    {{ run_query(sql) }}
    {{ log("Permissions granted", info=True) }}
{% endmacro %}
```

```bash
dbt run-operation grant_permissions
```

---

*Next: [04-use-cases-and-ecosystem.md](./04-use-cases-and-ecosystem.md) — Where dbt shines and how it fits the modern data stack.*
