# ADR-0001: Build a two-tier platform: raw files in Cloud Storage, a BigQuery warehouse on top

- **Status:** accepted
- **Date:** 2026-10-08
- **Author:** Till Lüder
- **Lesson:** L01

## Context

Today Albert's Marketplace has no analytical architecture. One PostgreSQL
database serves the web shop, and the analysts query it directly or use a
nightly CSV export ([data flows](../data-flows.md), flows 1 to 4). This causes
three problems:

- **Analytics competes with checkout.** Analyst queries run on the tables the
  shop writes to.
- **Sensitive data leaves the shop every night.** The export contains
  `customer.pwd` (12-character values with no hash format) and
  `payment_details.card_no` and `cvv`. PCI DSS forbids storing a CVV after
  authorisation.
- **Revenue has no single definition.** It is computed from live queries and
  from last night's CSV, with each spreadsheet's own logic, and the reports
  disagree.

What the platform must serve:

- **Analysts and revenue reports:** SQL and spreadsheets. They need one trusted
  revenue figure, not live data.
- **The ML team:** units sold per product, per country, per day, refreshed
  daily.

Facts, from the v1.0.0 export (`legacy_csv`) unless stated:

| Fact | Figure |
|---|---|
| Size | 27 tables, 197 MB of CSV |
| Orders | 222,644 orders and 408,560 order lines, 15 Sep 2023 to 14 Sep 2026: about 200 orders a day |
| Countries | 6, from `shipping_details.country` |
| Revenue (Σ `payment.amount`) | 39.2 M |
| Data types | Tables now. The company repo also runs a Kafka clickstream (JSON events) and enables PostgreSQL logical decoding (CDC), both arriving in L03. Product and review images are referenced by `product_images` and `review_images` |
| Team | Four MSc students. Strongest in SQL and Python, new to cloud operations |
| Platform | Google Cloud, set by the course: Cloud Storage and BigQuery, region `europe-west1` |

In this ADR, **two-tier** means raw files kept in object storage (the lake tier)
plus a separate warehouse loaded from them. A **lakehouse** means a single copy,
stored in an open table format on object storage, that the SQL engine queries
in place.

## Options considered

| Option | For | Against |
|---|---|---|
| **Data warehouse** (BigQuery only: extracts loaded straight into tables) | One system, one access model, so governance happens in one place (column-level security, views). The analysts already work in SQL. Serverless, so nothing to operate. At 197 MB it costs almost nothing. | No raw copy outside the warehouse: a bad load or transform cannot be replayed from what the source actually sent. JSON clickstream and image references fit, but awkwardly. Every new source has to be modelled before it can land. |
| **Data lake** (Cloud Storage only: files, queried by scripts) | Cheapest storage. Takes any format, including the clickstream and images. Keeps raw history. | Analysts would need Python or Spark instead of SQL. Access control is per file or bucket, not per column, so card and personal fields cannot be hidden from some readers. Nothing defines revenue once, so the current problem stays. Without strict rules it becomes a dump of CSVs, as today. |
| **Two-tier** (raw files in Cloud Storage, curated tables in BigQuery) | Raw extracts land unchanged and immutable, so any transform can be replayed. The lake tier takes the clickstream and files without modelling them first. Analysts and ML get SQL tables in BigQuery, with one revenue definition in gold. Each tier uses the tool it is good at. | Two storage systems, and two access models to secure: Cloud Storage IAM and BigQuery. Data is stored twice, a negligible cost at this size. One more hop, so freshness is at best one load cycle behind. |
| **Lakehouse** (one copy in an open table format, e.g. Apache Iceberg on Cloud Storage, queried by BigQuery through BigLake) | A single copy serves SQL and file-based tools (Python and Spark for ML). Open format, so no lock-in. Keeps the lake's flexibility with warehouse transactions. | Adds a table format, a catalogue and file maintenance (compaction, snapshots) that the team has never run. Those mechanisms pay off at a volume and number of engines Albert's Marketplace does not have: 197 MB, and one query engine. More pieces to break before the first report works. |

## Criteria

Scored from `––` (rules it out) to `++` (strongest).

| Criterion | What Albert's Marketplace needs | Warehouse | Lake | Two-tier | Lakehouse |
|---|---|---|---|---|---|
| **Workload** | BI and ML both read from a governed copy, never from the shop | `++` | `–` | `++` | `+` |
| **Data types** | Tables now, JSON clickstream and image references from L03 | `+` | `++` | `++` | `++` |
| **Freshness** | Daily is enough: the reports use a nightly file today, and the ML grain is one day | `+` | `+` | `+` | `+` |
| **Team and skills** | SQL-first team and analysts, new to cloud operations | `++` | `––` | `+` | `–` |
| **Cost** | 197 MB, so storage is negligible; billed per byte a query scans | `+` | `+` | `+` | `+` |
| **Governance** | Drop cards and passwords, restrict personal columns, one revenue definition | `++` | `––` | `+` | `+` |

**What decided it:**

- **Lake out:** it fails *team and skills* and *governance*, the two problems
  the analysts and the CVV export cause today.
- **Lakehouse out:** it buys one copy and multi-engine access at the price of
  table-format operations the team cannot yet run. That trade only pays at a
  volume and number of engines this company does not have.
- **Warehouse vs two-tier:** they tie on *workload*, *freshness* and *cost*. The
  warehouse wins on *team and skills* and *governance* (one system). Two-tier
  wins on *data types*: the clickstream and CDC streams arrive in L03 as JSON
  events, and an immutable raw tier lets us land them before modelling them and
  replay any transform. We choose two-tier and close its governance gap by
  design: card data and passwords are never extracted, and only the pipeline's
  service account can read the raw bucket.
- **Freshness and cost** score the same for every option, so they did not
  decide anything.

## Decision

A two-tier platform. Each source lands as raw, immutable Parquet files in a
Cloud Storage bucket (**bronze**). dbt builds cleaned tables (**silver**) and
business tables (**gold**) in BigQuery. Analysts, reports and the ML team read
only gold, and nothing reads the shop database except the ingestion job.

## Consequences

**Easier**

- **The shop is protected:** analyst queries no longer touch checkout's tables.
- **Revenue is defined once,** in a gold model with tests. The reports stop
  disagreeing because there is only one place the figure comes from.
- **Cards and passwords stay out:** `payment_details.card_no`, `cvv` and
  `customer.pwd` are excluded at extraction and never reach the platform.
- **A transform bug is recoverable:** rebuild silver and gold from bronze.

**Harder**

- **Two systems to secure and keep consistent:** bucket IAM and BigQuery
  permissions.
- **Analysts lose live access.** Their data is up to one day old, and they
  learn to query gold instead of production tables.
- **A pipeline to run and monitor** that did not exist before.

**To watch**

- **Raw bucket access:** if analysts get read access to bronze, the governance
  argument for this choice collapses.
- **BigQuery bytes scanned:** partition gold by date from the start (L04, L09).
- **Country of an order:** it is not on the order. It comes from the buyer's
  shipping address, and the export holds today's default address, not the one
  used at order time. The star schema has to state this.
- **The table count:** the brief says 25 tables, the export holds 27. Find out
  which ones are missing from the brief before ingestion (L02).
- **When to revisit:** a new ADR should reconsider the lakehouse if the
  clickstream makes storing data twice expensive, or if the ML team needs to
  read the same tables from Python or Spark without going through BigQuery.
