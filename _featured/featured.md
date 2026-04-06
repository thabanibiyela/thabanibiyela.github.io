---
layout: default
title: "Featured Project"
permalink: /featured
---


# From Spreadsheets to the Cloud: Automating Revenue Accounting

---

## Overview

Large-scale digital marketplaces generate enormous volumes of financial data — and the accounting processes that support them need to keep up. This project examined that challenge first-hand, diagnosing a fragmented, spreadsheet-heavy operation inside a platform processing hundreds of millions of transactions annually, and delivering a working cloud-based solution to modernise and automate it.

The project was implemented within the platform accounting team at a leading online food delivery platform operating across 17 countries and processing 879 million orders in 2024 alone. Each order generates multiple financial microtransactions — all of which must be accurately captured, enriched, reconciled, and reported to the general ledger.

The team's existing system relied on a combination of a bespoke ETL script, a customised accounting integration, and extensive manual spreadsheet work to fulfil its responsibilities across journal entries, reconciliations, audit support, and stakeholder reporting. Manual processes increased turnaround times, introduced risk of error, and were not scalable given the size and complexity of the datasets involved.

### Approach

Following an internal analysis, a gap analysis was performed to map the team's strategic objectives to the current IT architecture. The resulting solution was delivered across three work packages:

- **Project A — Master Data Harmonisation:** Migration of data storage and processing to Google BigQuery using DBT
- **Project B — Stakeholder Reporting:** Interactive business intelligence dashboards built in Looker Studio
- **Project C — Automation Platform:** A Python command line application to automate journal entries, reconciliations, and audit support

### Results

| Process | Before | After |
|---|---|---|
| Journal entry (per account) | ~60 minutes | < 5 minutes |
| Reconciliation (per account) | ~180 minutes | < 5 minutes |
| Annual labour saving | — | 350+ hours |


**Stack:** Python · SQL (DBT) · JavaScript · Google BigQuery · Google App Scripts · Google Looker Studio · JupyterLab · Git

💻 [View the code on GitHub](#)

---

## The Data Model

The data model is the backbone of the solution. Built in **Google BigQuery** and powered by **DBT (Data Build Tool)**, it transforms raw event-level platform transaction data into structured, auditable financial accounting data through five sequential layers.

```
Transactions → Staging → Intermediate → Enrichment → Aggregation → Presentation
```

### Layer 0 — Event Level Data
Platform transaction data is extracted in real time from the order management system and populated into BigQuery.

### Layer 1 — Staging
Basic transformations are applied to the raw data — renaming, type casting, basic computations, and minimal joins. Only dimensions relevant to the platform accounting team are selected, keeping the model lean. Four sub-groups of staging models are maintained: order-level data, invoice-level data, restaurant master data, and other master data.

### Layer 2 — Intermediate
Staging entities are integrated to produce the inputs required for enrichment. Each intermediate model corresponds to a specific revenue stream or business process — for example, consumer delivery fee revenue and weekly invoiced partner revenue each have dedicated intermediate models.

### Layer 3 — Enrichment
Accounting master data is used to enrich intermediate models with the financial accounting dimensions required by the accounting system: general ledger account numbers, cost centres, revenue categories, and spend categories.

### Layer 4 — Aggregation
Enriched data is grouped by the minimum dimensions required by downstream processes. This step balances the need for granularity (important for audit and compliance) against the practical constraints of query performance and downstream system compatibility.

### Layer 5 — Presentation
Transformed and aggregated data is structured for easy consumption by both the CLI application and Looker Studio dashboards.

### Performance Optimisation
By default, DBT materialises models as database views — SQL statements that compute dynamically on query. While this keeps data consistent with the underlying source, it results in slow query times across a deeply layered model. To meet the project's latency requirement of five minutes or less, the aggregation layer is materialised as a "physical" table. The performance improvement was dramatic: querying a partner invoicing view for a given period took over eight minutes, while the equivalent table query completed in one second.

### Testing & Data Integrity
Data checks are written as SQL SELECT statements and stored in the DBT project's test directory. A **continuous monitoring workflow** runs these checks daily in production, aborting the materialisation job and alerting the team if any check fails — ensuring only verified data reaches downstream consumers.

💻 [View the DBT project on GitHub](#)

---

## The CLI Application

The command line application (`gforevpy`) is a Python-based automation platform that allows the platform accounting team to interact with BigQuery data and automate the production of outputs for journal entries, account reconciliations, and audit support.

### Architecture
The application retrieves data from BigQuery via the Python BigQuery Client and performs the necessary calculations and data manipulations before generating outputs. It is structured as a modular Python package, with each package performing a defined scope of tasks and delegating out-of-scope responsibilities to other packages. Object-oriented design principles ensure uniform communication interfaces across the package hierarchy, making the codebase extensible and maintainable.

### Getting Started

Install the application in a Python virtual environment:

```bash
pip install .
```

Authorise Google Cloud access:

```bash
gcloud init
```

### Usage

```bash
gforevpy <process> <year> <month> <id>
```

| Argument | Options | Description |
|---|---|---|
| `process` | `journal`, `reconciliation`, `audit` | The accounting process to run |
| `year` | 4-digit integer | The accounting year |
| `month` | 1–12 | The accounting month |
| `id` | Integer | Journal process ID or GL account number |

**Example — Journal entry for August 2025, Promoted Placement Revenue:**
```bash
gforevpy journal 2025 8 3
```

**Example — Reconciliation for August 2025, account 4900:**
```bash
gforevpy reconciliation 2025 8 4900
```

### Key Features

**Journal Entry**
The application automates the production of two outputs. The **integration file** contains the financial data formatted for direct ingestion by the accounting system. The **review workbook** is an audit requirement that documents posting logic, reconciliations across revenue categories, and the results of data quality and plausibility checks — all generated automatically and ready for the preparer to review and sign off.

**Reconciliations**
Each month, account balances in the accounting system are reconciled to platform source data. The application automates all data processing steps and produces a structured workbook presenting month-by-month reconciliation summaries, variance analysis, and data quality check results — reducing a process that previously took up to three hours to under five minutes.

**Audit Support** *(in development)*
The audit support module is currently under development and will allow the team to generate order- or invoice-level audit files on demand for any account and period.

### Testing
Unit tests cover all isolated methods that do not connect to external systems. Integration tests verify that the application can successfully interface with BigQuery and produce the expected outputs. User acceptance testing was performed iteratively throughout development against the functional requirements defined at the outset of the project.

💻 [View the CLI application on GitHub](#)

---

## App Scripts

Google App Scripts — a JavaScript-based platform for automating tasks across Google products — powers two automated workflows that keep the system running reliably without manual intervention.

### Continuous Monitoring Workflow

Each morning, a script performs data integrity checks on the BigQuery dataset:

1. Assertions written as SQL SELECT statements are evaluated against the latest data
2. **If all checks pass:** the aggregated tables are materialised from their views and the team receives a notification confirming the job results
3. **If any check fails:** the materialisation job is aborted and the team is sent a detailed test report for investigation

This ensures that only tested and verified data is ever exposed to downstream consumers — the CLI application and Looker Studio dashboards.

```javascript
// Example: DDL statement executed by App Scripts to materialise the partner invoicing table
CREATE OR REPLACE TABLE redacted.r2r_revenue_test.aggregated_qXX_partner_invoicing_table
OPTIONS() AS (
  WITH tb AS ( SELECT * FROM redacted.r2r_revenue_test.aggregated_qXX_partner_invoicing_view )
  SELECT * FROM tb
);
```

### Accounting System Data Extraction

A second workflow automates the extraction of financial data from the accounting system (Workday) to BigQuery. This eliminates the need for manual data exports and ensures that reconciliation and reporting data is always up to date.

**Latency targets:**
- First 5 working days of the month (peak month-end period): maximum 30 minutes
- All other periods: maximum 24 hours

---

## Stakeholder Reporting — Looker Studio

As custodians of platform accounting data, the team is responsible for providing stakeholders with timely insight into account balances, variance drivers, and key performance indicators. Previously, this was a reactive, manual process — stakeholders would raise ad hoc requests by email or instant message, and the team would process and prepare a response from scratch each time.

The stakeholder reporting component of the project replaces this with **interactive, on-demand dashboards** built in Google Looker Studio, connected directly to the BigQuery data model.

### Dashboard Design
Requirements were gathered by analysing historical stakeholder data requests and conducting walkthrough sessions to understand how financial insights are consumed and acted upon.

Each dashboard provides:
- **Time series analysis** of account balances across the reporting period
- **Monthly variance analysis** with commentary support
- **Variance split** — attributing total variance to its component drivers (e.g. volume vs. price effects)
- **Geographic breakdown** by market, with country-level filtering

---

📄 [Read the full report](#) · 💻 [View the code on GitHub](#) · [Back to Portfolio](index.md)