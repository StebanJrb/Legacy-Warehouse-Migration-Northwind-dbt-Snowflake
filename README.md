# Incremental Warehouse Migration — Legacy Postgres + MongoDB to dbt + Snowflake

![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white)
![dbt](https://img.shields.io/badge/dbt-FF694B?style=flat&logo=dbt&logoColor=white)
![Snowflake](https://img.shields.io/badge/Snowflake-29B5E8?style=flat&logo=snowflake&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat&logo=postgresql&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=flat&logo=mongodb&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![Apache Airflow](https://img.shields.io/badge/Apache%20Airflow-017CEE?style=flat&logo=apacheairflow&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat&logo=githubactions&logoColor=white)

## Overview

This is deliberately scoped like a real migration ticket, not a capstone audit: a subscription-business reporting warehouse is being modernized model by model, over several sprints, while leadership keeps using the legacy numbers every week in the meantime — so nothing can break, nothing can go stale, and nothing gets cut over until it's proven correct. The added realism most "migration" portfolio projects skip: the legacy logic doesn't live in one clean system. Part of it is in PostgreSQL stored procedures; part of it depends on customer engagement data that a *different team* maintains in MongoDB.

## The Problem

A subscription SaaS company's weekly leadership report — churn risk by segment, commission payouts, customer health scores — is computed by a legacy PostgreSQL warehouse with business logic split across stored procedures. Migrating this to dbt + Snowflake is a real, ongoing initiative, but with constraints that make it a genuine day-to-day engineering problem instead of a one-shot rewrite:

- **You cannot do a big-bang cutover.** The legacy warehouse still runs and still feeds the exec dashboard while migration is in progress — leadership does not stop needing Monday's numbers because a migration is underway. Each migrated model has to run in **shadow mode** (both old and new producing output) before anything is allowed to cut over.
- **The source of truth isn't all in Postgres.** Customer health scoring depends on an `engagement_scores` collection maintained in **MongoDB** by the Growth Engineering team — updated by their own service, with a schema that includes nested experiment-assignment data the legacy stored procedure joins against via a nightly flat-file export nobody remembers setting up. Migrating this model means first understanding and properly extracting from a system your team doesn't own.
- **The business logic is genuinely undocumented** — a stored procedure computing commission tiers has conditional logic that only makes sense after a conversation with whoever's still around from Finance who remembers why a tier boundary is `$47,500` and not a round number.

This is the actual shape of migration work at most companies: incremental, cross-team, partially undocumented, and constrained by "the business still depends on this being right every single day."

## Data Source

- **Legacy relational side:** The **Northwind** sample schema, extended with realistic stored procedures for churn-risk segmentation and tiered commission calculation — deliberately including a couple of "why is this threshold here" quirks that have to be resolved by reading the logic carefully, not guessed at.
- **Legacy MongoDB side:** A seeded `engagement_scores` collection (per-customer engagement score, experiment arm, feature-flag history as nested arrays) representing data owned by another team — the project includes the extraction script that pulls this into a staging layer, since dbt itself can't query Mongo directly.

## Architecture

- **Legacy System:** PostgreSQL (extended Northwind + stored procedures) **and** MongoDB (`engagement_scores`, external system boundary).
- **Extraction Layer (new, not in the original scope):** A Python job (`pymongo`) that incrementally pulls the MongoDB collection into a Snowflake staging table on a schedule — this is its own small pipeline, not a one-time export, because Growth Engineering's data keeps changing.
- **Migration Mapping:** One model migrated at a time, each documented with its legacy source (stored procedure logic or Mongo fields), written for a teammate who's never seen the original.
- **Target Stack:** dbt Core (staging → intermediate → marts) on Snowflake.
- **Shadow Mode & Parity (the core of the project):** Every migrated model runs in parallel with its legacy equivalent for a minimum validation window. A nightly comparison job diffs old-vs-new output and only marks a model "cutover-eligible" after N consecutive days of exact parity — this is the actual mechanism, not a single one-off test.
- **Cutover:** A feature-flag config (`migration_status.yml`) controls which models the BI layer reads from legacy vs. new — allowing per-model, reversible cutover instead of an all-or-nothing switch.
- **Infrastructure:** Terraform provisioning the Snowflake objects and the scheduled extraction job's infrastructure.
- **CI/CD:** GitHub Actions running dbt tests plus the parity comparison on every PR that touches a migrated model.
- **Orchestration:** Airflow DAGs for the nightly dbt run, the Mongo extraction job, and the shadow-mode comparison — three separate concerns, scheduled independently.

## Tech Stack

- PostgreSQL 15 (legacy relational source)
- **MongoDB** + `pymongo` (legacy source owned by another team)
- dbt Core (staging/intermediate/marts)
- Snowflake (target warehouse)
- Terraform (infrastructure as code)
- Apache Airflow 2.x (extraction, dbt runs, shadow comparison)
- GitHub Actions (CI/CD, parity checks on every PR)
- Docker & Docker Compose

## What This Demonstrates

- Realistic, incremental migration practice — shadow mode and feature-flagged cutover, not a rewrite-and-pray approach
- Handling a legacy source that spans two heterogeneous systems (Postgres + a Mongo collection owned by another team), which is far more common in practice than a single clean relational legacy system
- Extraction engineering as its own ongoing concern, not a one-time script
- Reverse-engineering undocumented business logic and being explicit about what had to be inferred vs. confirmed
- CI/CD that treats "does the new model match the old one" as a mandatory, automated gate — not a manual sign-off

## Status
🚧 In Development

## Project Structure
```
├── legacy/
│   ├── stored_procedures/
│   ├── extended_northwind_schema.sql
│   └── migration_mapping.md
├── extraction/
│   └── mongo_engagement_extractor.py    # ongoing, scheduled — not a one-time export
├── dbt_project/
│   ├── models/
│   │   ├── staging/
│   │   ├── intermediate/
│   │   └── marts/
│   ├── tests/
│   │   └── shadow_mode/                 # nightly parity comparison logic
│   └── macros/
├── config/
│   └── migration_status.yml             # per-model cutover flags
├── terraform/
├── dags/
│   ├── nightly_dbt_run_dag.py
│   ├── mongo_extraction_dag.py
│   └── shadow_mode_comparison_dag.py
├── .github/workflows/
├── docker-compose.yml
└── README.md
```
