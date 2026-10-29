# `analytics`: the dbt project

**🌐 Live dbt docs:** [joanasoares0.github.io/dbt-duckdb-pipeline](https://joanasoares0.github.io/dbt-duckdb-pipeline/)

![Project 1: Local Pipeline](../img/project_1.png)

This is the dbt project behind the **Automated Job Postings Pipeline (v1)**. It turns 12 months of raw job-posting parquet files into a small, tested and documented DuckDB warehouse.

> 🚀 **v2 is coming on Databricks.** The same modeling approach will be rebuilt on a lakehouse, with more advanced sources, incremental models and orchestration.

---

## ✨ What it does

- **Reads raw parquet directly.** The `raw.job_postings` source points at `read_parquet('../data/raw/raw_job_postings_*.parquet')` via `external_location`, so there is no load step.
- **Cleans once, reuses everywhere.** `job_postings` keeps the nine useful columns and drops the scraper's error rows. Every mart builds on this one model with `ref()`.
- **Answers real questions:**
  - *How is demand trending for each role?* → `jobs_per_month`
  - *How many postings are remote each month?* → `monthly_summary`
  - *Where are the jobs?* → `top_locations`
- **Tests the grain of each mart** with `unique` and `not_null`.
- **Documents everything.** Model and column descriptions live in `_schema.yml` and are persisted to DuckDB with `persist_docs`.

---

## 🧱 Models

| Model | Materialization | Grain | Description |
|---|---|---|---|
| `job_postings` | view | one row per posting scrape | Cleaned postings: 9 named columns, rows with `error IS NOT NULL` removed |
| `jobs_per_month` | table | month × role | Posting counts per search role per month |
| `monthly_summary` | table | month | Total postings, remote postings and `pct_remote` |
| `top_locations` | table | location | Top 10 locations by posting count |

Models are materialized as **tables** by default (`dbt_project.yml`). The staging model overrides this to a **view**.

### Lineage

```
raw.job_postings (source)
   └── job_postings
         ├── jobs_per_month ──┐
         ├────────────────────┴── monthly_summary
         └── top_locations
```

---

📖 You can browse the interactive lineage on the **[hosted dbt docs](https://joanasoares0.github.io/dbt-duckdb-pipeline/)**.

---

## ✅ Tests

| Model | Column | Tests |
|---|---|---|
| `job_postings` | `job_title` | `not_null` |
| `monthly_summary` | `month` | `not_null`, `unique` |
| `top_locations` | `job_location` | `not_null`, `unique` |

`dbt build` runs the models and tests together in DAG order. A model whose test fails blocks everything downstream of it.

---

## ▶️ Running

From this folder, after downloading the data at the repo root (`uv run python scripts/download_data.py`):

```bash
uv run dbt deps           # install dbt_utils
uv run dbt build          # run models + tests (uses your ~/.dbt/profiles.yml)
uv run dbt docs generate && uv run dbt docs serve   # lineage + docs site
```

CI runs the same build against the `prod` target:

```bash
uv run dbt build --profiles-dir ../ci   # writes ../data/prod.duckdb
```

---

## 📂 Structure

```
analytics/
├── models/
│   ├── _sources.yml        # raw.job_postings → parquet files
│   ├── _schema.yml         # descriptions, tests, persist_docs
│   ├── job_postings.sql    # staging view
│   ├── jobs_per_month.sql  # mart
│   ├── monthly_summary.sql # mart
│   └── top_locations.sql   # mart
├── dbt_project.yml
└── packages.yml            # dbt-labs/dbt_utils
```

← Back to the [project README](../README.md).
