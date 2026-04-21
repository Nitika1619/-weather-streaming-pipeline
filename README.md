# -weather-streaming-pipeline
Real-time weather data pipeline using GCP Pub/Sub, Apache Beam, Dataflow and BigQuery
# 🌦️ Real-Time Weather Streaming Pipeline

![Python](https://img.shields.io/badge/Python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Apache Beam](https://img.shields.io/badge/Apache%20Beam-2.x-FF6F00?style=for-the-badge&logo=apache&logoColor=white)
![Google Cloud](https://img.shields.io/badge/Google%20Cloud-Pub%2FSub-4285F4?style=for-the-badge&logo=googlecloud&logoColor=white)
![Dataflow](https://img.shields.io/badge/GCP-Dataflow-34A853?style=for-the-badge&logo=googlecloud&logoColor=white)
![BigQuery](https://img.shields.io/badge/BigQuery-Analytics-4285F4?style=for-the-badge&logo=googlebigquery&logoColor=white)
![Looker Studio](https://img.shields.io/badge/Looker-Studio-EA4335?style=for-the-badge&logo=looker&logoColor=white)

An end-to-end **real-time data streaming pipeline** built on Google Cloud Platform that ingests live weather data from 3 Indian cities at 1–2 records/sec, processes it through a 4-step Apache Beam transformation pipeline, loads clean data into BigQuery, and visualises insights on a live Looker Studio dashboard.

---

## 📌 Live Demo

| Resource | Link |
|----------|------|
| 📊 Looker Studio Dashboard | [View Live Dashboard](#) |
| 🗄️ BigQuery Table Schema | See below |

---

## 🏗️ Architecture

```
┌─────────────────┐     1-2 msg/sec     ┌──────────────────┐
│  Python Producer │ ──────────────────► │  Cloud Pub/Sub   │
│  (Open-Meteo API)│                     │  (weather-raw)   │
└─────────────────┘                     └────────┬─────────┘
                                                  │
                                                  ▼
                                        ┌──────────────────┐
                                        │    Dataflow      │
                                        │  (Apache Beam)   │
                                        │                  │
                                        │ 1. Parse JSON    │
                                        │ 2. Filter nulls  │
                                        │ 3. Validate +    │
                                        │    anomaly flag  │
                                        │ 4. Enrich record │
                                        └────────┬─────────┘
                                                  │
                                                  ▼
                                        ┌──────────────────┐
                                        │    BigQuery      │
                                        │ weather_dataset  │
                                        │ .weather_stream  │
                                        └────────┬─────────┘
                                                  │
                                                  ▼
                                        ┌──────────────────┐
                                        │  Looker Studio   │
                                        │ Live Dashboard   │
                                        └──────────────────┘
```

---

## 🛠️ Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Data Source | Open-Meteo API | Free real-time weather API, no key required |
| Ingestion | Python + google-cloud-pubsub | Fetch and publish messages every 1 second |
| Message Queue | Google Cloud Pub/Sub | Decouple producer from processor, buffer messages |
| Stream Processing | Apache Beam + GCP Dataflow | 4-step transformation pipeline |
| Data Warehouse | Google BigQuery | Store and query clean streaming data |
| Visualisation | Looker Studio | Live auto-refreshing dashboard |

---

## ⚙️ How It Works

### Step 1 — Python Producer (`producer.py`)
A Python script runs in an infinite loop, calling the **Open-Meteo API** every second for 3 Indian cities: **Indore, Mumbai, and Delhi**. Each API response is wrapped into a JSON message and published to a **Cloud Pub/Sub topic** called `weather-raw`.

Each message looks like this:
```json
{
  "city": "Indore",
  "temperature": 38.0,
  "humidity": 15,
  "wind_kmh": 14.2,
  "weather_code": 0,
  "fetched_at": "2025-04-21T08:15:36+00:00"
}
```

### Step 2 — Cloud Pub/Sub
Pub/Sub acts as a **message queue** between the producer and the pipeline. This decouples the two — if the pipeline restarts, messages wait safely in the queue and nothing is lost. The Dataflow pipeline reads from the subscription `weather-raw-sub`.

### Step 3 — Apache Beam Pipeline (`pipeline.py`)
The pipeline reads messages from Pub/Sub and passes each one through **4 transformation steps**:

| Step | Transform | What it does |
|------|-----------|-------------|
| 1 | `ParseJson` | Decodes raw bytes → Python dict. Drops unparseable messages. |
| 2 | `FilterNulls` | Drops any record missing `temperature`, `humidity`, or `city`. |
| 3 | `CleanAndValidate` | Rounds all floats to 2 decimal places. Flags readings outside physical range (temp < -50°C or > 60°C, humidity outside 0–100%) as `is_anomaly = TRUE`. |
| 4 | `Enrich` | Adds UTC timestamp and `source = "open-meteo-api"` to every record. |

### Step 4 — BigQuery
Clean records are streamed directly into a BigQuery table. SQL queries run instantly over millions of rows for analytics like hourly trends, city comparisons, and anomaly frequency.

### Step 5 — Looker Studio Dashboard
Connected directly to BigQuery, the dashboard auto-refreshes and displays:
- 📈 Temperature trend over time (line chart, one line per city)
- 💧 Average humidity per city (bar chart)
- 🌡️ Latest temperature scorecards per city
- ⚠️ Anomaly detection table

---

## 🗄️ BigQuery Table Schema

**Dataset:** `weather_dataset`  
**Table:** `weather_stream`

| Column | Type | Description |
|--------|------|-------------|
| `city` | STRING | City name (Indore / Mumbai / Delhi) |
| `timestamp` | TIMESTAMP | UTC time when record was processed |
| `temperature_c` | FLOAT | Temperature in Celsius, rounded to 2dp |
| `humidity_pct` | FLOAT | Relative humidity percentage, rounded to 2dp |
| `wind_kmh` | FLOAT | Wind speed in km/h, rounded to 2dp |
| `weather_code` | INTEGER | WMO weather condition code |
| `is_anomaly` | BOOLEAN | TRUE if reading is outside physical range |
| `source` | STRING | Data source identifier |

### Sample SQL Queries

```sql
-- Average temperature per city in last 1 hour
SELECT
    city,
    ROUND(AVG(temperature_c), 2) AS avg_temp,
    ROUND(AVG(humidity_pct), 2)  AS avg_humidity,
    COUNT(*)                      AS total_readings
FROM `weather-pipeline-493711.weather_dataset.weather_stream`
WHERE timestamp >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
GROUP BY city
ORDER BY avg_temp DESC;

-- Hourly temperature trend for Indore
SELECT
    TIMESTAMP_TRUNC(timestamp, HOUR) AS hour,
    ROUND(AVG(temperature_c), 2)     AS avg_temp
FROM `weather-pipeline-493711.weather_dataset.weather_stream`
WHERE city = 'Indore'
GROUP BY hour
ORDER BY hour;

-- All anomalies detected
SELECT city, timestamp, temperature_c, humidity_pct
FROM `weather-pipeline-493711.weather_dataset.weather_stream`
WHERE is_anomaly = TRUE
ORDER BY timestamp DESC
LIMIT 20;
```

---

## 🚀 How to Run Locally

### Prerequisites
- Python 3.11
- Google Cloud account with billing enabled
- GCP APIs enabled: Pub/Sub, Dataflow, BigQuery

### 1. Clone the repo
```bash
git clone https://github.com/Nitika1619/weather-streaming-pipeline.git
cd weather-streaming-pipeline
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

### 3. Authenticate with GCP
```bash
gcloud auth application-default login
gcloud config set project weather-pipeline-493711
```

### 4. Create GCP resources (one-time)
```bash
# Create Pub/Sub topic and subscription
gcloud pubsub topics create weather-raw
gcloud pubsub subscriptions create weather-raw-sub --topic=weather-raw --ack-deadline=60

# Create BigQuery dataset
bq mk --dataset --location=US weather-pipeline-493711:weather_dataset

# Create GCS bucket for Dataflow temp files
gsutil mb -l us-central1 gs://weather-pipeline-493711-temp
```

### 5. Run the pipeline (two terminals)

**Terminal 1 — Start the producer:**
```bash
python producer.py
```

**Terminal 2 — Start the Beam pipeline:**
```bash
python pipeline.py
```

### 6. Verify data in BigQuery
Go to **console.cloud.google.com → BigQuery → weather_dataset → weather_stream → Preview**

You should see rows appearing within 1–2 minutes.

---

## 📁 Project Structure

```
weather-streaming-pipeline/
│
├── producer.py          # Fetches weather API + publishes to Pub/Sub
├── pipeline.py          # Apache Beam pipeline with 4 transforms
├── requirements.txt     # Python dependencies
├── .gitignore           # Excludes secrets and cache files
└── README.md            # This file
```

---

## 💡 Key Design Decisions

**Why Pub/Sub instead of calling Dataflow directly?**
Pub/Sub decouples the producer and processor. If the pipeline restarts, messages wait safely in the queue — nothing is lost. This is a fundamental pattern in production data engineering.

**Why keep anomalies instead of dropping them?**
Flagging bad readings as `is_anomaly = TRUE` rather than deleting them preserves data for debugging and pattern analysis. You can always filter them out in queries.

**Why DirectRunner instead of DataflowRunner?**
DirectRunner runs the Beam pipeline locally — zero cost, easier to debug, and sufficient for a project of this scale. The same pipeline code deploys to DataflowRunner with a single flag change when scaling to production.

---

## 📈 What I Learned

- Designing decoupled streaming architectures using Pub/Sub
- Writing Apache Beam DoFn transforms for real-time data cleaning
- Streaming data into BigQuery with schema enforcement
- Writing analytical SQL for time-series data
- Building live dashboards in Looker Studio connected to BigQuery

---

## 👩‍💻 Author

**Nitika Chowdhary**  
M.Tech Internet of Things — Devi Ahilya Vishwavidyalaya, Indore  
📧 nitikachowdhary9@gmail.com  
🔗 [LinkedIn](https://www.linkedin.com/in/nitika-chowdhary-63181a22b) | [GitHub](https://github.com/Nitika1619)

---

*Built as a portfolio project to demonstrate real-time data engineering on GCP.*
