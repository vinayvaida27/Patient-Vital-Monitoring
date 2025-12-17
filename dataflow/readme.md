## 🧠 Code Explanation: Real-Time Patient Vitals Pipeline (Beginner Friendly)

This section explains the Python script for the streaming data pipeline **step by step**, ideal for beginners. The pipeline is implemented using **Apache Beam** and runs on **Google Cloud Dataflow**.

---

### 📦 Step 1: Import Required Libraries

```python
import os
import json
from dotenv import load_dotenv
import apache_beam as beam
from apache_beam.options.pipeline_options import PipelineOptions, StandardOptions
from apache_beam.transforms.window import FixedWindows
```

- `os` & `json`: Work with environment variables and JSON data.
- `load_dotenv`: Loads configuration from a `.env` file.
- `apache_beam`: Main data processing library.
- `PipelineOptions`: Configure the pipeline settings.
- `FixedWindows`: Divides data into time-based chunks (e.g. every 60 seconds).

---

### ⚙️ Step 2: Load Environment Variables

```python
load_dotenv()
PROJECT_ID = os.getenv("GCP_PROJECT")
...
```

- Reads credentials and configurations from `.env` so we don’t hardcode them.
- These include project ID, storage paths, subscription names, etc.

---

### 🛠️ Step 3: Set Beam Pipeline Options

```python
pipeline_options = PipelineOptions(
    project=PROJECT_ID,
    streaming=True,
    ...
)
pipeline_options.view_as(StandardOptions).streaming = True
```

- Tells Apache Beam to run in **streaming mode** (for real-time data).
- Specifies GCP project, region, temp, and staging locations.

---

### 🧰 Step 4: Define Helper Functions

#### ✅ `parse_json()`

```python
def parse_json(message):
    try:
        return json.loads(message)
    except:
        return None
```

- Converts a string to a Python dictionary.
- Returns `None` if it’s not valid JSON.

#### ✅ `is_valid_record()`

```python
def is_valid_record(record):
    ...
```

- Validates the data by:
  - Checking required fields exist
  - Ensuring values are within expected ranges

#### ✅ `enrich_record()`

```python
def enrich_record(record):
    ...
```

- Calculates a **risk score** and **risk level** (Low, Moderate, High).
- Formula:
  $$
  \text{risk\_score} =
  \left( \frac{\text{heart\_rate}}{200} \times 0.4 \right) +
  \left( \frac{\text{temperature}}{40} \times 0.3 \right) +
  \left( 1 - \frac{\text{spo2}}{100} \right) \times 0.3
  $$

---

### 🔄 Step 5: Define Apache Beam Pipeline

```python
with beam.Pipeline(options=pipeline_options) as p:
```

- This is the main container where your pipeline steps are defined.

---

### 🥉 Bronze Layer: Raw Ingestion from Pub/Sub

```python
bronze_data = (
    p
    | "Read from PubSub" >> beam.io.ReadFromPubSub(subscription=PUBSUB_SUBSCRIPTION)
    | "Decode to string" >> beam.Map(lambda x: x.decode("utf-8"))
    | "Window Bronze Data" >> beam.WindowInto(FixedWindows(60))
)
```

- Reads raw messages from Pub/Sub.
- Decodes bytes into strings.
- Batches messages into **60-second windows**.

```python
bronze_data | "Write Bronze to GCS" >> beam.io.WriteToText(
    BRONZE_PATH + "raw_data", file_name_suffix=".json"
)
```

- Writes raw messages to GCS for traceability (Bronze layer).

---

### 🥈 Silver Layer: Clean + Enrich

```python
silver_data = (
    bronze_data
    | "Parse JSON" >> beam.Map(parse_json)
    | "Filter Invalid Records" >> beam.Filter(is_valid_record)
    | "Enrich with Risk Score" >> beam.Map(enrich_record)
    | "Window Silver Data" >> beam.WindowInto(FixedWindows(60))
)
```

- Parses the raw string into JSON.
- Filters out bad/incomplete records.
- Adds risk score and risk level.
- Groups them into 1-minute windows again.

```python
silver_data | "Write Silver to GCS" >> beam.io.WriteToText(
    SILVER_PATH + "cleaned_data", file_name_suffix=".json"
)
```

- Stores the cleaned/enriched data into GCS (Silver layer).

---

### 🥇 Gold Layer: Aggregate + Write to BigQuery

#### Helper Functions

```python
def extract_for_aggregation(record):
    return (record["patient_id"], record)
```

- Groups data by `patient_id`.

```python
def aggregate_records(key_values):
    ...
```

- Computes:
  - Average heart rate
  - Average SpO₂
  - Average temperature
  - Highest observed risk level

#### Beam Steps

```python
gold_data = (
    silver_data
    | "Key by patient_id" >> beam.Map(extract_for_aggregation)
    | "Group by patient_id" >> beam.GroupByKey()
    | "Aggregate per patient" >> beam.Map(aggregate_records)
)
```

- Aggregates the enriched data for each patient within the window.

```python
gold_data | "Write Gold to BigQuery" >> beam.io.WriteToBigQuery(
    BIGQUERY_TABLE,
    write_disposition=beam.io.BigQueryDisposition.WRITE_APPEND,
    create_disposition=beam.io.BigQueryDisposition.CREATE_IF_NEEDED
)
```

- Writes the final metrics to BigQuery (Gold layer).

---

### ✅ Summary

| Layer   | Description                            | Output              |
|---------|----------------------------------------|---------------------|
| Bronze  | Raw Pub/Sub data                       | GCS (raw JSON)      |
| Silver  | Cleaned + risk-enriched data           | GCS (cleaned JSON)  |
| Gold    | Aggregated analytics per patient       | BigQuery table      |

This architecture allows for traceable, validated, and analytics-ready streaming data pipelines for real-time patient monitoring on GCP.
