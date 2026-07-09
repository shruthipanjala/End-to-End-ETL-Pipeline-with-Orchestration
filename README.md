# Healthcare Data Pipeline 🏥📊

An orchestrated ETL pipeline that pulls real public healthcare data from three government sources, cleans and models it into a proper star schema, validates it with automated data quality checks, and runs on a daily schedule.
---

## Why I built this

A lot of "data engineering" portfolio projects I saw (including my earlier ones) were really just a Python script that reads a CSV, does a `groupby`, and writes another CSV. That's not really data engineering there's no orchestration, no data quality enforcement, and no real modeling.

I wanted to build something that actually looks like what a data engineering team ships: multiple real (messy, occasionally inconsistent) data sources, a proper transformation layer with tests, automated quality gates that would actually catch a broken pipeline before bad data reaches anyone downstream, and a schedule it runs on without me babysitting it.

I picked healthcare because the public data is genuinely interesting to join together — hospital quality, drug safety, and chronic disease prevalence all connect at the state level, and I wanted to see if that connection actually showed anything.

## What it does

Every day, the pipeline:

1. **Extracts** data from three public APIs:
   - CMS Hospital General Information — quality ratings for every US hospital
   - openFDA — real reported drug adverse events
   - CDC Chronic Disease Indicators — state-level disease prevalence
2. **Loads** raw data into staging tables in Postgres
3. **Transforms** it with dbt into a proper star schema (`dim_state`, `fact_hospital_quality`, `fact_chronic_disease`, `fact_drug_events`)
4. **Validates** it with Great Expectations — checks for null keys, valid rating ranges, valid state codes, and unexpected drops in row count
5. **Alerts** if anything fails, instead of silently producing bad data

## Architecture

```
CMS API ──┐
FDA API ──┼──► Extract (Python) ──► raw.* (Postgres) ──► dbt staging ──► dbt marts (star schema) ──► Great Expectations gate ──► (dashboard)
CDC API ──┘

                                    All orchestrated by Airflow, daily schedule, retries + failure alerting
```

I used **Airflow** for orchestration instead of just chaining cron jobs because I wanted real dependency management between tasks and visibility when something breaks — one government API silently changing a field name shouldn't take down the whole pipeline undetected. I used **dbt** for transformation instead of doing it all in pandas because I wanted version-controlled, testable SQL with lineage tracking, which is much closer to how data teams actually build warehouses in practice.

## Tech Stack

- **Orchestration:** Apache Airflow (Docker)
- **Transformation:** dbt
- **Warehouse:** PostgreSQL
- **Data quality:** Great Expectations
- **Extraction:** Python (requests, pandas)
- **Deployment:** Docker Compose

## Data Quality Results

This is the part I actually care about most — a pipeline that "runs" but doesn't check its own output isn't worth much:

- `[X]` records flagged by Great Expectations across `[X]` pipeline runs (mostly `[e.g. "invalid state codes from the FDA source"]`)
- dbt tests: `[X]` tests defined across staging/mart models, `[X]%` passing consistently
- Pipeline has run successfully on schedule for `[X]` consecutive days as of this writing
- One real schema-drift incident caught: `[describe if you hit one — e.g. "CDC renamed a field mid-project, quality gate caught the resulting null spike before it reached the marts"]`

*(Full Great Expectations validation results are in `/great_expectations/uncommitted/data_docs/`)*

## What I'd do differently / next

- **Incremental loads** — right now every run does a full refresh of each source. For a real production pipeline I'd move to incremental extraction (only pulling new/changed records) to reduce load on the source APIs and speed up runs
- **A real warehouse instead of local Postgres** — I'd move this to BigQuery or Snowflake to actually demonstrate cloud-warehouse patterns (partitioning, clustering) at a scale local Postgres doesn't require
- **Better schema-drift handling** — right now a renamed field from a source API breaks the extractor with a KeyError. I'd want defensive schema validation on the raw extract step itself, before it ever reaches staging
- **A dashboard layer** — the star schema is ready for one, but I haven't built the BI layer on top yet

## Running it locally

```bash
git clone [your-repo-url]
cd healthcare-data-pipeline
docker compose up -d
```

This spins up Postgres + Airflow. Once Airflow is up (`localhost:8080`), trigger the `healthcare_pipeline` DAG manually for a first run, or let the daily schedule handle it.

To explore the dbt models and lineage graph:
```bash
cd dbt_project
dbt docs generate
dbt docs serve
```

To run the data quality checks standalone:
```bash
great_expectations checkpoint run healthcare_checkpoint
```

## A note on this being a portfolio project

The data is real and public, but this isn't feeding any actual clinical or business decision — it's a demonstration of orchestration, modeling, and data-quality practices using real-world messy government data rather than a clean sample dataset. Happy to walk through the dbt lineage graph or the Great Expectations setup in more depth — those are the two pieces I'd most want to talk through.
