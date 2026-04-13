# dbt (Data Build Tool) — The Definitive Engineering Reference

> Written for experienced Data Engineers transitioning from distributed systems (PySpark, Flink), relational databases (Oracle SQL), and NoSQL (MongoDB) into the modern analytics engineering paradigm.

---

## Guide Structure

| # | Document | Covers |
|---|----------|--------|
| 1 | [01-core-philosophy.md](./01-core-philosophy.md) | ETL vs ELT, dbt's role as the "T", software engineering for SQL |
| 2 | [02-project-anatomy.md](./02-project-anatomy.md) | Models, ref(), materializations, tests, snapshots, seeds, macros, Jinja, packages |
| 3 | [03-workflow-and-cli.md](./03-workflow-and-cli.md) | Dev loop, CLI commands, profiles.yml, environment management |
| 4 | [04-use-cases-and-ecosystem.md](./04-use-cases-and-ecosystem.md) | Data quality, lineage, modular modeling, modern data stack integration |
| 5 | [05-interview-prep.md](./05-interview-prep.md) | Comprehensive interview Q&A — concepts, scenarios, coding challenges, rapid-fire |

---

## Quick Mental Model

```
                    ┌─────────────────────────────────────────┐
                    │         YOUR DATA WAREHOUSE              │
                    │  (Snowflake / BigQuery / Redshift / etc) │
                    └──────────────┬──────────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────────┐
                    │              dbt                          │
                    │  SELECT-only transformations              │
                    │  Version-controlled SQL + Jinja           │
                    │  DAG-based execution                      │
                    │  Built-in testing & documentation         │
                    └──────────────┬──────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                     ▼
         Staging Models      Intermediate Models    Mart Models
         (1:1 source map)    (business logic)       (consumption-ready)
```

dbt does not extract or load data. It assumes data is already in your warehouse. It compiles Jinja-templated SQL into pure SQL, resolves a DAG of dependencies, and executes transformations inside the warehouse's compute engine.

---

## How to Use This Guide

- Read **01** first if you need the philosophical grounding (ETL→ELT shift, why dbt exists).
- Jump to **02** if you want the hands-on anatomy of every dbt component.
- Use **03** as your daily CLI and workflow reference.
- Consult **04** when architecting a new project or evaluating dbt's fit in your stack.

Each document is self-contained but cross-references the others where relevant.
