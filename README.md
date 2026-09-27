# 🐋 BTC Whale Alert & Large Transfer Tracker (AWS S3 + Snowflake + dbt)

An automated Web3 data engineering pipeline that ingests raw Bitcoin blockchain transaction logs from AWS S3 into Snowflake, unpivots multi-output UTXO recipient structures, and models high-value address movements (`whale_alerts`) using dbt with automated GitHub Actions CI/CD testing.

---

## 📌 Project Overview

Unlike account-based blockchains (like Ethereum), Bitcoin operates on an **Unspent Transaction Output (UTXO)** model where a single transaction can send funds to multiple recipient addresses simultaneously.

This project ingests raw UTXO transaction logs, normalizes complex multi-output arrays into individual transfer records, and creates a high-priority analytics layer that isolates **"Whale" transactions** (high-value BTC transfers) for downstream security, market monitoring, and alert engines.

---

## 🏗️ Data Architecture & Pipeline Lineage

[ AWS S3 Bucket (Raw BTC JSON/CSV Logs) ]
│
▼  (Snowflake COPY INTO Task)
RAW.BTC.STAGED_BTC_TRANSFERS
│
▼
stg_btc (View)
│
▼
stg_btc_transactions (Ephemeral)
│
▼
stg_btc_output (View)
│
▼
fct_whale_alerts (Table)

---

## 📂 Model Architecture & Layering

| Layer | Model Name | Materialization | Purpose & Rationale |
| :--- | :--- | :--- | :--- |
| **Staging** | `stg_btc` | `view` | **Raw Landed Data:** Direct wrapper over the Snowflake `COPY INTO` task target table. Standardizes field names, block numbers, transaction hashes (`tx_id`), and timestamps. |
| **Staging** | `stg_btc_transactions` | `ephemeral` | **Transformation Bridge:** An intermediate CTE-style helper model used to parse raw transaction payloads and calculate total transaction fees and input/output counts without materializing redundant storage. |
| **Staging** | `stg_btc_output` | `view` | **UTXO Unpivoting:** Explodes transactions containing multiple output addresses into individual, row-level recipient records so each address's incoming BTC amount is isolated. |
| **Marts** | `whale_alerts` | `table` | **Whale Analytics Mart:** Filters normalized transaction outputs for large transfer thresholds (e.g., transfers $\ge 100\text{ BTC}$), calculating total value moved, block context, and recipient address profiles. |

---

## 🔄 Automated Ingestion & Pipeline Highlights

1. **AWS S3 to Snowflake Ingestion:** Raw Bitcoin transaction batches are staged in AWS S3 and continuously loaded into Snowflake via automated `COPY INTO` tasks.
2. **UTXO Array Normalization:** Solves the 1-to-Many Bitcoin output problem by flattening multi-recipient arrays into discrete atomic transactions.
3. **Ephemeral Model Efficiency:** Uses `stg_btc_transactions` as an `ephemeral` model to keep Snowflake storage lean while re-using reference logic across downstream marts.

---

## 🛡️ Data Quality & Automated CI/CD

### 1. Schema Assertions (`schema.yml`)
- **Primary Key Integrity:** Enforces uniqueness and non-null constraints on surrogate keys generated across flattened outputs (`tx_id` + `output_index`).
- **Value Assertions:** Verifies that transfer amounts (`btc_amount`) are non-negative and timestamps are present.

### 2. GitHub Actions CI/CD Pipeline (`.github/workflows/dbt_ci.yml`)
Integrated into the multi-project CI/CD workflow to run automated syntax checks and data assertions on every Pull Request:

```bash
dbt run --select stg_btc stg_btc_output fct_whale_alerts 
dbt test --select stg_btc stg_btc_output fct_whale_alerts 
🚀 Quickstart Guide
1. Environment Setup
Bash
# Clone repository
git clone [https://github.com/your-username/btc-whale-tracker.git](https://github.com/your-username/btc-whale-tracker.git)
cd btc-whale-tracker

# Activate virtual environment and install dbt-snowflake
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install dbt-core==1.9.4 dbt-snowflake==1.9.4
dbt deps
2. Execution Commands
Bash
# Verify Snowflake connection
dbt debug

# Run all BTC models
dbt run --select tag:btc

# Run data quality tests
dbt test --select tag:btc
