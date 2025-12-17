# Patient-Vital-Monitoring  
Real-Time Patient Vitals Streaming & Analytics Pipeline (GCP)

This project implements a real-time patient vitals monitoring system on Google Cloud Platform using a **streaming medallion architecture**:

> **Bronze → Silver → Gold**

Live vitals are streamed from a Python simulator into Pub/Sub, processed with Apache Beam on Dataflow, validated and enriched, and finally aggregated into BigQuery for analytics and dashboards.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Data Model](#data-model)
- [Risk Scoring Logic](#risk-scoring-logic)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Simulator](#running-the-simulator)
- [Running the Streaming Pipeline](#running-the-streaming-pipeline)
- [Monitoring & Outputs](#monitoring--outputs)
- [Troubleshooting](#troubleshooting)
- [Security & Best Practices](#security--best-practices)
- [Use Cases](#use-cases)
- [Author](#author)
- [License](#license)

---

## Architecture Overview

End-to-end data flow:

```text
Patient Vitals Simulator (Python)
        ↓
Google Pub/Sub (Streaming Ingestion)
        ↓
Apache Beam (Python SDK) on Dataflow
        ↓
Bronze Layer (Raw JSON → GCS)
        ↓
Silver Layer (Validated & Enriched → GCS)
        ↓
Gold Layer (Aggregated Metrics → BigQuery)
        ↓
Analytics / Dashboards (Looker / Tableau / etc.)
```

The pipeline runs in **streaming mode** with **windowed aggregations** and exactly-once semantics enabled.

---

## Features

- Real-time ingestion of patient vitals using **Google Pub/Sub**
- Streaming processing with **Apache Beam** on **Google Cloud Dataflow**
- **Medallion architecture**
  - 🥉 **Bronze** – Raw Pub/Sub messages stored in GCS
  - 🥈 **Silver** – Validated and enriched records in GCS
  - 🥇 **Gold** – Aggregated patient metrics in BigQuery
- Data validation and quality checks
- Risk score calculation and classification
- Error injection simulator for testing data quality logic
- Analytics-ready BigQuery tables

---

## Tech Stack

- **Language:** Python 3
- **Streaming:** Google Pub/Sub
- **Processing:** Apache Beam (Python SDK)
- **Runner:** Google Cloud Dataflow (Streaming)
- **Storage:**
  - Google Cloud Storage (Bronze / Silver)
  - BigQuery (Gold)
- **Configuration:** `.env` via `python-dotenv`
- **Infrastructure:** Google Cloud Platform
- **Version Control:** GitHub

---

## Repository Structure

```text
Patient-Vital-Monitoring/
├─ dataflow/
│  └─ streaming_medallion_pipeline.py
├─ simulator/
│  └─ patinet_vitals_simulator.py
├─ .env                    # Environment variables (not committed)
├─ requirements.txt
└─ README.md
```

> Note: The simulator filename is `patinet_vitals_simulator.py` as provided.

---

## Data Model

### Incoming vitals (Pub/Sub)

```json
{
  "patient_id": "P001",
  "timestamp": "2025-01-01T12:00:00Z",
  "heart_rate": 82.5,
  "spo2": 97.2,
  "temperature": 37.1,
  "bp_systolic": 120,
  "bp_diastolic": 80
}
```

### Silver layer additions

- `risk_score`
- `risk_level` (Low / Moderate / High)

### Gold layer aggregates (BigQuery)

- `patient_id`
- `avg_heart_rate`
- `avg_spo2`
- `avg_temperature`
- `max_risk_level`

---

## Risk Scoring Logic

```text
risk_score =
  (heart_rate / 200) * 0.4 +
  (temperature / 40) * 0.3 +
  (1 - spo2 / 100) * 0.3
```

Risk levels:

- **Low**: `< 0.3`
- **Moderate**: `0.3 – 0.6`
- **High**: `> 0.6`

---

## Prerequisites

- Google Cloud Project with billing enabled
- Enabled APIs:
  - Pub/Sub
  - Dataflow
  - BigQuery
  - Cloud Storage
- Python 3.8+
- `gcloud` CLI authenticated or service account credentials
- GCS buckets for Bronze, Silver, temp, and staging
- Pub/Sub topic and subscription
- BigQuery dataset/table

---

## Installation

```bash
git clone https://github.com/<your-org-or-user>/Patient-Vital-Monitoring.git
cd Patient-Vital-Monitoring

python3 -m venv .venv
source .venv/bin/activate

pip install -r requirements.txt
```

Example `requirements.txt`:

```text
apache-beam[gcp]
google-cloud-pubsub
python-dotenv
```

---

## Configuration

Create a `.env` file in the project root:

```dotenv
GCP_PROJECT=patient-vital
REGION=us-central1

PUBSUB_TOPIC=patient_vitals_topic
PUBSUB_SUBSCRIPTION=projects/patient-vital/subscriptions/patient_vitals_subscription

BRONZE_PATH=gs://patient-vitals-streaming-vaida/bronze/
SILVER_PATH=gs://patient-vitals-streaming-vaida/silver/
TEMP_LOCATION=gs://patient-vitals-streaming-vaida/temp/
STAGING_LOCATION=gs://patient-vitals-streaming-vaida/staging/

BIGQUERY_TABLE=patient-vital.healthcare.patient_risk_analytics

PATIENT_COUNT=20
STREAM_INTERVAL=2
ERROR_RATE=0.1
```

---

## Running the Simulator

```bash
cd simulator
python patinet_vitals_simulator.py
```

The simulator continuously publishes vitals (with optional errors) to Pub/Sub.

---

## Running the Streaming Pipeline

### Run on Dataflow (recommended)

```bash
cd dataflow

python streaming_medallion_pipeline.py \
  --runner DataflowRunner \
  --project $GCP_PROJECT \
  --region $REGION \
  --staging_location $STAGING_LOCATION \
  --temp_location $TEMP_LOCATION \
  --streaming
```

### Run locally (optional)

```bash
python streaming_medallion_pipeline.py --runner DirectRunner
```

---

## Monitoring & Outputs

- **Dataflow Console**: job graph, throughput, latency, logs
- **Bronze (GCS)**: raw JSON messages
- **Silver (GCS)**: validated & enriched records
- **Gold (BigQuery)**: aggregated patient metrics

Example query:

```sql
SELECT *
FROM `patient-vital.healthcare.patient_risk_analytics`
ORDER BY patient_id;
```

---

## Troubleshooting

- No data:
  - Verify simulator is running
  - Check Pub/Sub subscription
  - Confirm IAM permissions
- Pipeline startup failures:
  - Validate GCS paths
  - Ensure APIs are enabled
- Too many filtered records:
  - Review validation logic in `is_valid_record()`

---

## Security & Best Practices

- Secrets stored in `.env` (not committed)
- IAM-based access control
- Windowed aggregation for scalability
- Medallion architecture for traceability and data quality

---

## Use Cases

- Real-time patient monitoring
- Clinical risk detection
- Streaming healthcare analytics
- Public health surveillance
- Reference Dataflow streaming architecture

---

## Author

**Vinay Vaida**  
Data Engineer | Cloud & Streaming Systems  
GitHub: https://github.com/vinayvaida27

---

## License

MIT / Apache 2.0 (choose one and add a LICENSE file)
