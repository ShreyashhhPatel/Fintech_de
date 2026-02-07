# Fintech Transactions Data Engineering Pipeline (Databricks)

## Overview
This project demonstrates an end-to-end **production-style data engineering pipeline** built on **Databricks** using **Delta Lake** and a **Medallion Architecture (Bronze → Silver → Gold)**.

The goal of this project is to show practical, hands-on understanding of:
- Incremental ingestion patterns
- Schema drift handling
- Data quality enforcement
- Idempotent MERGE logic
- Analytics-ready data modeling

This project was intentionally built to work in a **restricted Databricks Community / Spark Connect environment**, where direct object storage access (DBFS FileStore, local FS) is disabled. The design choices reflect real-world constraints and solutions.

---

## Architecture

**Medallion Layers**

- **Bronze (Raw Landing)**: Raw fintech transaction events ingested as JSON
- **Silver (Clean & Trusted)**: Typed, validated, deduplicated transactions maintained incrementally
- **Gold (Analytics)**: Business-level aggregations for reporting and dashboards

```
Raw Events
   ↓
Bronze (raw JSON + metadata)
   ↓
Silver (validated, deduped, schema-enforced)
   ↓
Gold (KPIs, merchant performance, risk metrics)
```

---

## Dataset Description

The pipeline simulates **fintech transaction events**, including:

- transaction_id
- event_time
- account_id
- customer_id
- merchant_id
- amount
- currency
- txn_type (card_purchase, transfer, refund, atm_withdrawal)
- status (approved, declined, reversed)
- risk_score

Schema drift is simulated by introducing optional attributes over time:
- merchant_category

Invalid records (negative amounts, missing fields) are intentionally injected to test data quality handling.

---

## Bronze Layer

### Purpose
- Preserve **raw, immutable data**
- Allow reprocessing and schema evolution
- Capture ingestion metadata

### Table

**fintech_de.bronze_transactions_raw**

| Column | Type |
|------|------|
| raw_payload | STRING |
| ingest_time | TIMESTAMP |
| batch_id | STRING |

### Ingestion Strategy

- Raw events are generated in batches
- Each event is stored as a JSON string
- No transformations are applied in Bronze

This mirrors real-world ingestion via APIs, CDC tools, or message queues.

---

## Structured Bronze

### Purpose
- Parse JSON payloads into columns
- Materialize evolving schemas
- Retain raw payload for lineage and recovery

### Table

**fintech_de.bronze_transactions**

Key characteristics:
- JSON parsed using inferred schema
- Schema drift preserved
- Raw payload retained

---

## Silver Layer

### Purpose
- Create a **trusted, analytics-ready dataset**
- Enforce data quality rules
- Handle late-arriving and duplicate data

### Data Quality Rules

A record is considered **valid** if:
- transaction_id is not NULL
- event_time is a valid timestamp
- amount > 0
- currency is not NULL

Invalid records are excluded from Silver (optionally quarantined).

---

### Silver Table Schema

**fintech_de.silver_transactions**

| Column | Type |
|------|------|
| transaction_id | STRING |
| event_time | TIMESTAMP |
| account_id | STRING |
| customer_id | STRING |
| merchant_id | STRING |
| amount | DECIMAL(18,2) |
| currency | STRING |
| txn_type | STRING |
| status | STRING |
| risk_score | DOUBLE |
| ingest_time | TIMESTAMP |
| batch_id | STRING |
| raw_payload | STRING |
| merchant_category | STRING |

---

## Incremental MERGE Logic (Silver)

### Why MERGE?

- Prevent duplicate records
- Handle late-arriving updates
- Enable idempotent re-runs

### Merge Strategy

- **Primary key**: transaction_id
- **Conflict resolution**: newest ingest_time wins
- **Schema drift handling**: optional fields derived from raw_payload

### Key Principle

Delta Lake does **not** auto-evolve schemas during MERGE. Columns such as `merchant_category` must be explicitly added using `ALTER TABLE` before MERGE operations.

---

## Gold Layer

### Purpose
- Deliver business-ready metrics
- Power dashboards and analytics
- Isolate analytical logic from raw data complexity

---

### Gold Table 1: Daily KPIs

**fintech_de.gold_daily_kpis**

Metrics:
- Daily transaction count
- Total transaction amount
- Approval rate
- Average risk score

Used by:
- Finance teams
- Executive dashboards

---

### Gold Table 2: Merchant Performance

**fintech_de.gold_merchant_performance**

Metrics:
- Transaction volume by merchant
- Revenue by merchant
- Risk exposure by merchant category

Used by:
- Payments teams
- Merchant operations

---

### Gold Table 3: Risk Summary

**fintech_de.gold_risk_summary**

Metrics:
- Risk distribution by country
- Risk distribution by transaction channel (if available)

Used by:
- Fraud detection teams
- Risk analysts

---

## Key Engineering Concepts Demonstrated

- Medallion Architecture (Bronze / Silver / Gold)
- Incremental, idempotent MERGE patterns
- Schema drift handling via raw payload retention
- Explicit schema evolution using ALTER TABLE
- Data quality validation
- Production-safe SQL practices (no UPDATE SET *)

---

## How to Explain This Project in Interviews

> “I built an end-to-end fintech data pipeline on Databricks using Delta Lake and a medallion architecture. Raw JSON events are ingested into Bronze, validated and deduplicated into Silver using incremental MERGE logic, and aggregated into Gold tables for analytics. The pipeline is idempotent, schema-drift tolerant, and production-ready.”

---

## Repository Structure (Suggested)

```
fintech-databricks-pipeline/
├── notebooks/
│   ├── 01_bronze_ingestion
│   ├── 02_bronze_structured
│   ├── 03_silver_merge
│   ├── 04_gold_kpis
├── diagrams/
│   └── medallion_architecture.png
├── README.md
```

---

## Tools & Technologies

- Databricks (Community Edition)
- Apache Spark
- Delta Lake
- Spark SQL

---

## Final Notes

This project focuses on **realistic engineering trade-offs**, not toy examples. The implementation reflects constraints commonly encountered in managed cloud environments and demonstrates how production data pipelines are designed, validated, and maintained.

---

*End of document*

