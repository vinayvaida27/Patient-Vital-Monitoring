## 💉 Real-Time Patient Vitals Simulator (Beginner-Friendly Explanation)

This Python script simulates a **stream of patient vital signs** and publishes them to a **Google Cloud Pub/Sub** topic in real time. It mimics a medical device sending data every few seconds — **including occasional invalid or faulty records** to test downstream validation.

---

### 📦 Step 1: Import Required Libraries

```python
import time
import json
import random
import os
from google.cloud import pubsub_v1
from dotenv import load_dotenv
```

- `time`: For adding delays between messages and timestamps.
- `json`: To convert dictionaries into JSON strings.
- `random`: To simulate real-world variability in vitals and errors.
- `os`: For loading environment variables.
- `pubsub_v1`: Google Cloud Pub/Sub client for sending messages.
- `load_dotenv`: Loads values from `.env` file.

---

### ⚙️ Step 2: Load Configuration from `.env`

```python
load_dotenv()

PROJECT_ID = os.getenv("GCP_PROJECT")
TOPIC_ID = os.getenv("PUBSUB_TOPIC")
PATIENT_COUNT = int(os.getenv("PATIENT_COUNT", 20))
STREAM_INTERVAL = int(os.getenv("STREAM_INTERVAL", 2))
ERROR_RATE = float(os.getenv("ERROR_RATE", 0.1))
```

- Reads values from a `.env` file like:
  ```dotenv
  GCP_PROJECT=your-project
  PUBSUB_TOPIC=patient_vitals_topic
  PATIENT_COUNT=20
  STREAM_INTERVAL=2
  ERROR_RATE=0.1
  ```
- `ERROR_RATE = 0.1` means ~10% of records will have intentional errors.

---

### ☁️ Step 3: Initialize Pub/Sub Publisher

```python
publisher = pubsub_v1.PublisherClient()
topic_path = publisher.topic_path(PROJECT_ID, TOPIC_ID)
```

- Initializes the Pub/Sub client.
- Prepares the full topic path in the format:  
  `projects/<PROJECT_ID>/topics/<TOPIC_ID>`

---

### 🆔 Step 4: Generate Patient IDs

```python
patient_ids = [f"P{i:03d}" for i in range(1, PATIENT_COUNT + 1)]
```

- Generates IDs like `P001`, `P002`, ..., `P020` (if 20 patients).
- These IDs will be randomly picked for each message.

---

### 🧪 Step 5: Function to Generate Vitals

```python
def generate_vitals():
    ...
```

#### ✅ Valid Data (Normal Case)

```python
record = {
    "patient_id": random.choice(patient_ids),
    "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
    "heart_rate": round(random.uniform(60, 120), 1),
    "spo2": round(random.uniform(90, 100), 1),
    "temperature": round(random.uniform(36.0, 39.0), 1),
    "bp_systolic": round(random.uniform(100, 140), 1),
    "bp_diastolic": round(random.uniform(60, 90), 1)
}
```

- Generates realistic values for:
  - **Heart Rate**: 60–120 bpm
  - **SpO₂ (Oxygen Saturation)**: 90–100%
  - **Temperature**: 36.0–39.0°C
  - **Blood Pressure** (Systolic/Diastolic)

#### ❌ Error Injection (Simulated Faults)

```python
if random.random() < ERROR_RATE:
    error_type = random.choice(["missing_field", "negative_value", "out_of_range"])
    ...
```

Randomly injects **bad data** into some records:
- `missing_field`: Removes a random field (`None`)
- `negative_value`: Sets heart rate to -1
- `out_of_range`: Sets SpO₂ to 150 (invalid)

---

### 🔁 Step 6: Start Sending Messages

```python
print("Starting patient vitals simulator with error injection... Press Ctrl+C to stop.")
```

A friendly log to show it's running.

#### 🔄 Infinite Loop

```python
while True:
    vitals = generate_vitals()
    message_json = json.dumps(vitals)
    publisher.publish(topic_path, message_json.encode("utf-8"))
    print(f"Published: {message_json}")
    time.sleep(STREAM_INTERVAL)
```

- Forever loop (until manually stopped with `Ctrl + C`)
- Every `STREAM_INTERVAL` seconds (e.g., 2s):
  - Generate a record
  - Convert it to JSON
  - Send it to Pub/Sub
  - Print it to the terminal

---

### ✅ Example Output

```json
Published: {
  "patient_id": "P008",
  "timestamp": "2025-12-16T10:15:03Z",
  "heart_rate": 78.5,
  "spo2": 98.4,
  "temperature": 37.2,
  "bp_systolic": 120.0,
  "bp_diastolic": 82.0
}
```

Or an example with injected error:

```json
Published: {
  "patient_id": "P005",
  "timestamp": "2025-12-16T10:15:05Z",
  "heart_rate": -1,
  "spo2": 97.8,
  "temperature": 37.6,
  "bp_systolic": 130.0,
  "bp_diastolic": 80.0
}
```

---

## 🎯 Summary

| Component         | Purpose                                      |
|------------------|----------------------------------------------|
| `.env` file       | Stores config (project, topic, interval)     |
| `generate_vitals()` | Simulates real & faulty patient vitals     |
| `Pub/Sub`         | Sends the data to cloud for processing       |
| `STREAM_INTERVAL` | Controls how frequently data is sent         |
| `ERROR_RATE`      | Controls how often fake data is injected     |

This script is useful to **test and validate real-time streaming pipelines** like the one built using Apache Beam and Dataflow.
