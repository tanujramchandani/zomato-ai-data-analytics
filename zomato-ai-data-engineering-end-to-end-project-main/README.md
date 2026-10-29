# Zomato Data Engineering & AI Analytics Project

Built by Tanuj Ramchandani

This project is a personal end-to-end data engineering portfolio project that demonstrates how raw food delivery data can be ingested, transformed, orchestrated, and enriched using modern cloud and AI tools.

The pipeline follows this flow:

**Zomato-style food delivery data → Amazon S3 → Snowflake → dbt → Airflow → AI analytics**

The goal is to build a practical data platform that moves data from raw files into a warehouse, applies transformation logic, exposes clean business-ready tables, and adds AI-powered insights for review analysis and natural-language querying.

![Architecture](docs/architecture.png)

## Project overview

This project uses raw CSV files to simulate a real-world food delivery dataset and processes them through a modern data stack:

- Data is stored in Amazon S3 as a raw lake layer.
- Snowflake ingests and stores the raw data using a storage integration and staged load process.
- dbt transforms the data into staged, dimensional, and fact-based models.
- Airflow orchestrates the end-to-end workflow.
- OpenAI is used to enrich review text and enable AI-driven analytics.

## What gets built

| Layer | Where | What |
|---|---|---|
| **Source** | `data/` (local) | 4 dimension CSV files and multiple generated fact files covering restaurants, users, food, menu, orders, order items, and reviews |
| **Lake** | Amazon S3 | Raw bucket structure organized by table |
| **Bronze** | Snowflake `ZOMATO.RAW` | Load process using `COPY INTO` from S3 |
| **Silver** | Snowflake `ZOMATO.STAGING` | Clean, standardized staging views |
| **Gold** | Snowflake `ZOMATO.MARTS` | Dimensions, incremental facts, and aggregated business marts |
| **AI** | Snowflake `ZOMATO.AI` | Review enrichment, semantic search, and text-to-SQL capabilities |
| **Orchestration** | Airflow (Docker) | Daily pipeline orchestration with dependency-aware tasks |

## Tech stack

Python · Pandas · Amazon S3 · Snowflake · dbt (dbt-snowflake) · Apache Airflow 3 (Docker) · OpenAI (`gpt-4o-mini`, `text-embedding-3-small`) · Streamlit

## Repository structure

```
├── airflow/                  # Airflow 3 on Docker
│   ├── Dockerfile            #   Snowflake + OpenAI providers, dbt in its own venv
│   ├── docker-compose.yaml   #   postgres + api-server + scheduler; creds via env vars
│   ├── example.env           #   template for SNOWFLAKE_* / OPENAI_API_KEY
│   └── dags/zomato_batch.py  #   orchestrates the batch pipeline
├── zomato/                   # dbt project
│   ├── models/staging/       #   staging views and source definitions
│   ├── models/marts/         #   final dimensional and business models
│   └── macros/               #   schema naming logic
├── ai/                       # AI layer
│   ├── enrich_reviews.py     #   LLM review enrichment
│   ├── rag_chat.py           #   question answering over reviews
│   ├── text_to_sql.py        #   natural-language warehouse querying
│   └── example.env           #   AI environment template
├── snowflake/                # Snowflake setup SQL
│   ├── 01_setup.sql          #   warehouse, database, schemas, and role setup
│   ├── 02_storage_integration.sql  # S3 storage integration config
│   ├── 03_stage_and_formats.sql    # external stage + CSV file format
│   ├── 04_raw_tables.sql     #   RAW table definitions
│   └── 05_copy_into.sql      #   raw data ingestion script
├── aws/iam/                  # IAM policy and trust policy examples
├── docs/architecture.png     # architecture diagram
└── README.md                # project overview and setup guide
```

> `data/` files and generated artifacts are intentionally excluded from the repository because of size. This project is designed to use your own local dataset or a dataset downloaded from your preferred source.

## How the pipeline works

### 1 · Data lands in S3

The raw CSV files are uploaded into an S3 landing zone with a folder structure such as `raw/<table>/` for each data domain, including restaurants, users, food, menu, orders, order items, and reviews.

### 2 · S3 → Snowflake: storage integration

Snowflake reads from the bucket using a storage integration and IAM role-based access. The SQL under [`snowflake/`](snowflake/) defines the warehouse, database, schema, and staged ingestion pipeline. The AWS policy artifacts in [`aws/iam/`](aws/iam/) demonstrate the access model needed for secure S3-to-Snowflake communication.

### 3 · Load — `COPY INTO`

The raw table definitions in [`snowflake/04_raw_tables.sql`](snowflake/04_raw_tables.sql) match the source data schema. The load logic in [`snowflake/05_copy_into.sql`](snowflake/05_copy_into.sql) loads each staged file into Snowflake raw tables.

### 4 · Transform — dbt (medallion)

- **Staging (Silver)** — cleans and standardizes raw source files.
- **Dimensions (Gold)** — creates restaurant, customer, food, and date dimension tables.
- **Facts (Gold, incremental)** — builds order and order item fact models using incremental logic.
- **Marts (Gold)** — creates business-ready tables for revenue, restaurant performance, SLA analysis, and review insights.
- **Tests** — validates data quality with dbt tests such as uniqueness, null checks, and relationships.

### 5 · Orchestrate — Airflow

The DAG defined in [`airflow/dags/zomato_batch.py`](airflow/dags/zomato_batch.py) chains the workflow into a single process:

```
reload_raw  →  dbt_build_core  →  enrich_reviews  →  dbt_build_ai
(COPY from S3)  (dbt build + tests)  (AI enrichment)   (AI marts)
```

This allows the project to run in a repeatable, scheduled, or triggered manner using Dockerized Airflow.

### 6 · AI layer — three capabilities

1. **LLM enrichment** — reads review text, generates structured sentiment and topic information, and writes the enriched output back into the warehouse.
2. **RAG** — creates a review-based question-answering experience using embeddings and semantic retrieval.
3. **Text-to-SQL** — translates natural-language questions into Snowflake SQL and validates the generated query before execution.

## Project setup

```bash
# Snowflake objects and storage integration
# Run the scripts in snowflake/01_setup.sql through snowflake/05_copy_into.sql in sequence.

# dbt
cd zomato
export SNOWFLAKE_ACCOUNT=... SNOWFLAKE_USER=... SNOWFLAKE_PASSWORD=...
dbt debug && dbt build --exclude tag:ai

# Airflow
cd airflow
cp example.env .env          # fill in SNOWFLAKE_* and OPENAI_API_KEY values
docker compose build && docker compose up -d
# Open the Airflow UI and unpause the DAG before triggering it.

# AI apps
export OPENAI_API_KEY=sk-...
python ai/enrich_reviews.py
streamlit run ai/rag_chat.py      # chat with reviews
streamlit run ai/text_to_sql.py   # chat with the warehouse
```

## Notes

This project is designed as a practical learning and portfolio example for modern data engineering workflows, combining cloud storage, warehouse modeling, orchestration, and AI-powered analytics in a single pipeline.

It can be adapted for a personal portfolio, coursework, or a real business use case with your own data sources and cloud configuration.

