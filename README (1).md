# Fremont Rocket Company — Azure Data Platform

An end-to-end, config-driven data platform that takes Fremont Rocket's business data from Excel files on SharePoint through a medallion architecture into a dimensional model that powers finance and sales reporting in Power BI.

Built as a data engineering assessment. The brief was a *scalable* platform, not just a solution sized to today's data — the design notes below explain where I built for scale and where I'd deliberately start simpler.

---

## The Problem

Fremont Rocket had no data platform. Everything lived in six Excel and CSV files on SharePoint, maintained by different people, with no history, no validation, and no single source of truth. The CFO couldn't answer basic profitability questions without someone manually stitching workbooks together.

| Source file | Contents |
|---|---|
| `Sales_Data.csv` | Sales transactions |
| `Customer_Data.csv` | Customer master |
| `Sales_People.csv` | Salesperson master |
| `Employees.xlsx` | Employee records |
| `Production_Invoice.xlsx` | Production cost invoices |
| `Energy_Prices.xlsx` | Energy cost reference data |

Roughly 9,165 rows across all sources at time of assessment.

---

## Architecture

```
SharePoint (Graph API)
        │
        ▼
   Azure Data Factory ──── ingestion + orchestration
        │
        ▼
   ADLS Gen2 / Delta Lake
        │
   ┌────┴──────────────────────────┐
   │  BRONZE   raw, as-landed      │
   │  SILVER   cleaned, conformed  │
   │  GOLD     star schema          │
   └────┬──────────────────────────┘
        │  (Databricks — PySpark notebooks)
        ▼
    Azure SQL Database ──── serving layer
        │
        ▼
      Power BI ──── CFO / sales dashboards
```

### Layer responsibilities

**Bronze** — Files land exactly as received. No transformation, no filtering. Ingestion metadata added (`source_file`, `ingested_at`, `batch_id`) so any row can be traced back to the file and run that produced it. This layer is what makes reprocessing possible.

**Silver** — Cleaned and conformed. Data types enforced, column names standardised, duplicates removed, referential checks applied, business rules validated. One clean table per business entity.

**Gold** — Kimball-style dimensional model. Surrogate keys generated, dimensions loaded before facts, facts written with surrogate keys only.

### Dimensional model

| Table | Type | Notes |
|---|---|---|
| `dim_date` | Static | Pre-populated calendar |
| `dim_customer` | SCD Type 1 | Overwrite on change |
| `dim_salesperson` | SCD Type 2 | History tracked — territory and role changes affect commission attribution |
| `dim_rocket` | SCD Type 2 | History tracked — product spec and pricing changes matter for margin analysis |
| `dim_employee` | SCD Type 1 | |
| `fact_sales` | Transactional | Grain: one row per sale line |
| `fact_production_cost` | Transactional | Grain: one row per production invoice line |

---

## Design Decisions

### Why Azure SQL as the serving layer, not a Databricks SQL Warehouse

At ~9,165 rows, distributed query compute is a freight train carrying a briefcase. Azure SQL handles this volume without effort, costs a fraction of an always-on warehouse, uses SQL syntax the client's team already knows, and has a first-class Power BI connector. Databricks earns its place in the *transformation* layer; it doesn't need to own serving too.

### Why config-driven, not one notebook per source

Every source is described by a JSON config — path, format, schema, primary key, transformation rules, SCD type, target table. The notebooks are generic and read the config. Adding a seventh source is a config entry, not a new notebook. At six sources this is mild over-engineering; at sixty it's the only thing that keeps the platform maintainable.

### Why medallion at all for a company this small

The brief asked for a scalable platform. Bronze costs almost nothing to store and buys the ability to reprocess when a business rule is discovered to be wrong six months in — which, on a first-ever data platform, it will be.

**Honest note on right-sizing:** for the current data volume alone, ADF → Azure SQL → Power BI with no Databricks and no lake would be sufficient and cheaper. The medallion design here is the target state. See [Phased rollout](#phased-rollout) for how I'd actually sequence it.

---

## Data Quality & Reliability

**ABC validation (Audit, Balance, Control)** — Every layer transition records row counts in and out, and reconciles key measure sums. A Silver load that drops rows without an explained reason fails the run rather than quietly under-reporting revenue.

**Row hash idempotency** — Each row carries a hash of its business columns. Re-running a pipeline over the same source produces the same hashes and no duplicate inserts. Pipelines are safe to re-run after a partial failure.

**Watermarking** — Silver and Gold loads read incrementally from the previous layer using a watermark column, so a full history reload isn't required on every run.

**Central audit table** — One table records every pipeline run: batch ID, source, layer, start and end time, row counts, status, error message. This is where you look first when the CFO says a number looks wrong.

**Secrets** — All credentials live in Azure Key Vault and are referenced by ADF linked services. Nothing hardcoded, nothing in Git.

---

## Orchestration

ADF handles ingestion and triggering; Databricks Workflows handle layer dependencies.

- **Trigger** — Scheduled daily; SharePoint file-arrival trigger also supported
- **Ingestion** — ADF copies each source into Bronze in parallel via a metadata-driven `ForEach`
- **Transformation** — Databricks Workflow with explicit task dependencies: Bronze → Silver → Gold, dimensions before facts
- **Failure handling** — Any task failure stops downstream tasks, writes to the audit table, and alerts

---

## Repository Structure

```
├── notebooks/
│   ├── bronze/          # Ingestion from raw landing
│   ├── silver/          # Cleansing and conforming
│   ├── gold/            # Dimension and fact loads
│   └── utils/           # Shared transform, hash, and audit functions
├── config/              # Per-source JSON configs
├── adf/                 # Linked services, datasets, pipeline definitions
├── sql/                 # Azure SQL DDL, staging and serving schemas
├── powerbi/             # Report file and data model
└── docs/                # Architecture diagram, data dictionary
```

> Adjust this section to match your actual folder names before pushing.

---

## Setup

**Prerequisites** — Azure subscription with ADF, ADLS Gen2, Databricks, Azure SQL, and Key Vault provisioned. An app registration with Microsoft Graph `Files.Read.All` for SharePoint access.

1. Create the Key Vault secrets: SharePoint tenant ID, client ID, client secret, Azure SQL credentials
2. Deploy the ADF linked services and pipelines from `adf/`
3. Run the DDL in `sql/` to create staging and serving schemas
4. Import the notebooks into your Databricks workspace and attach to a small cluster (2 workers is plenty at this scale)
5. Update the source paths in `config/` to point at your SharePoint site
6. Run the pipeline manually once end-to-end, then enable the schedule
7. Point Power BI at the Azure SQL serving schema

---

## Phased Rollout

| Phase | When | Stack |
|---|---|---|
| **1 — Now** | Current volume, small team, first platform | ADF → Azure SQL → Power BI. No lake, no Databricks. Low cost, low complexity, gets the CFO reliable numbers fast. |
| **2 — Growth** | More sources, transformation logic outgrows SQL | Add ADLS + Databricks + medallion. Keep Azure SQL as the serving layer. |
| **3 — Scale** | Millions of rows, 50+ users, multiple departments | Move serving to Databricks SQL Warehouse or Snowflake. Departmental Gold marts. Unity Catalog governance. |

---

## Possible Extensions

**Multi-currency** — Add a conformed `dim_currency` plus a `fact_exchange_rate` table at daily grain (currency, date, rate). Facts store the transaction amount, the currency key, and the USD amount converted at the *transaction date's* rate. Rates belong in a fact, not a dimension — a slowly changing dimension can't represent a value that changes every single day, and using today's rate on a two-year-old sale silently corrupts historical margin.

**Streaming** — Only if the business asks for a live sales dashboard or real-time budget alerts. At current scale, nightly batch is the right answer. The path would be Event Hubs → Structured Streaming with micro-batches, Bronze/Silver/Gold unchanged, and batch retained alongside for end-of-day corrections and commission adjustments.

**Governance** — Unity Catalog with column-level security and lineage becomes worthwhile once there are multiple departments or any regulated data in scope.
