

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


Two fact tables sharing conformed `dim_date` and `dim_rocket`, which is what lets the CFO put revenue and production cost side by side on a single rocket model or a single month.

| Table | Type | Notes |
|---|---|---|
| `fact_sales` | Transactional | Grain: one row per rocket sale. Measures: `sale_price`, `commission_paid`. Also carries commission payout detail (`commission_paid_date`, `paid_to`) |
| `fact_production_expense` | Transactional | Grain: one row per invoice line. Measure: `amount`. Payee resolved by `party_type` to either an employee or a vendor |
| `dim_date` | Static | Pre-populated calendar — year, quarter, month, day-of-week, weekend flag. Conformed across both facts |
| `dim_rocket` | SCD Type 1 | Product master — fuel type, payload, fuel consumption, estimated lifetime launches, status. Conformed across both facts |
| `dim_customer` | SCD Type 1 | Purchasing organisations — industry, founding year, contact |
| `dim_salesperson` | SCD Type 1 | Sales staff and sales region. `is_active` is a soft-delete flag, not history |
| `dim_employee` | SCD Type 1 | Internal staff paid against production expenses. Retains `original_employee_id` from source |
| `dim_vendor` | SCD Type 1 | External suppliers paid against production expenses. Retains `original_vendor_id` from source |
| `dim_description` | SCD Type 1 | Invoice narrative and `expense_category`, keyed on `invoice_id` |
| `dim_service_contract` | SCD Type 1 | Post-sale service agreements, hanging off `fact_sales.sale_id` |
| `energy_prices` | Reference | Daily fuel and material prices (liquid hydrogen, APCP, uranium-235) with a `forecast_source` flag distinguishing actuals from forecasts. Sits outside the star — consumed for cost modelling, not aggregated against a business event |

### Notes on the model

**Polymorphic payee on production expense.** A production invoice can be paid to an internal employee or an external vendor. Rather than splitting into two fact tables or forcing a single "party" dimension over two very different entities, the fact carries `party_type` alongside nullable `employee_id` and `vendor_id`. Exactly one of the two is populated per row, enforced in the Gold load. The trade-off is a conditional join in reporting, in exchange for one clean cost fact that totals correctly without a union.

**Description as a dimension, not fact columns.** Invoice text and expense category are keyed on `invoice_id` and kept out of the fact. Categorisation logic can be corrected in one place without rewriting fact rows, and free-text notes stay off the high-row-count table.

**Service contract keyed to the sale.** A contract is a child of a sale, so it references `sale_id` directly. This makes attach-rate and post-sale revenue questions answerable without a bridge table.

**Source IDs preserved.** `original_employee_id` and `original_vendor_id` sit alongside the generated surrogate keys, which is what makes reconciliation against source files possible when a number is disputed.

**Current state is SCD Type 1.** Dimensions overwrite on change, which is correct for the current scope. The first place Type 2 would be needed is `dim_salesperson` — when a rep changes territory, Type 1 behaviour re-attributes their historical commission to the new region. If territory-level historical reporting is ever required, that's the dimension to convert first, and `dim_rocket` second if spec changes need to be reflected in historical margin.

---
<img width="1758" height="1016" alt="Untitled" src="https://github.com/user-attachments/assets/f65bb2c7-ce5d-47f2-9c5f-a4f4abad1aa8" />


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
