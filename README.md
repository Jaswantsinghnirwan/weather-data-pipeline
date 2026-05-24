# 🌡️ Weather Data Pipeline

A lightweight, event-driven data pipeline that polls real-time weather data, detects anomalies, fires webhook alerts, and persists everything to a local SQLite database — all containerised with Docker.

---

## System Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Container                         │
│                                                             │
│  APScheduler (every N seconds)                              │
│        │                                                    │
│        ▼                                                    │
│  [1] Fetch  ──► Open-Meteo API (current weather)           │
│        │                                                    │
│        ▼                                                    │
│  [2] Store  ──► SQLite: readings table                      │
│        │                                                    │
│        ▼                                                    │
│  [3] Detect ──► Compare against thresholds / prev reading   │
│        │                                                    │
│        ▼  (if condition met)                               │
│  [4] Alert  ──► POST JSON ──► Webhook.site                  │
│             └─► SQLite: alerts table                        │
└─────────────────────────────────────────────────────────────┘
```

**Data source:** [Open-Meteo](https://open-meteo.com/) — free, no API key required.

**Anomaly / condition logic (three independent checks per tick):**

| Alert type | Condition |
|---|---|
| `HEAT_ALERT` | Temperature ≥ `HEAT_THRESHOLD` (default 35 °C) |
| `FREEZE_ALERT` | Temperature ≤ `FREEZE_THRESHOLD` (default 0 °C) |
| `SPIKE_ALERT` | Absolute change from previous reading ≥ `SPIKE_DELTA` (default ±5 °C) |

All thresholds are configurable via environment variables — no code changes needed.

---

## Quick Start

### 1. Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/) installed.

### 2. Configure

```bash
cp .env.example .env
```

Open `.env` and set your webhook URL:

```
# Get a free URL at https://webhook.site — just open the site and copy your unique URL
WEBHOOK_URL=https://webhook.site/YOUR-UNIQUE-ID-HERE
```

Optionally adjust location, thresholds, or poll interval.

### 3. Run (single command)

```bash
docker compose up --build
```

That's it. The pipeline starts immediately, runs a first fetch, then repeats on the configured schedule.

To run in the background:

```bash
docker compose up --build -d
```

---

## Verifying Scheduled Ingestion is Running

### Option A — Live logs

```bash
docker compose logs -f
```

You will see timestamped output like:

```
2024-05-21 10:00:00 [INFO] ▶ Pipeline tick started
2024-05-21 10:00:01 [INFO] Fetched → temp=22.4°C  wind=14.0 km/h  code=1
2024-05-21 10:00:01 [INFO] ✓ No anomalies detected (temp=22.4°C)
2024-05-21 10:00:01 [INFO] ◀ Pipeline tick complete
2024-05-21 10:10:01 [INFO] ▶ Pipeline tick started   ← next scheduled run
```

### Option B — Query the database directly

```bash
# Open a shell inside the running container
docker exec -it weather-pipeline sh

# Query the readings table
sqlite3 /data/pipeline.db "SELECT * FROM readings ORDER BY id DESC LIMIT 10;"
```

Each successful fetch adds a new row with a timestamp and temperature reading.

---

## Verifying Webhook Alerts are Being Triggered

### Option A — Webhook.site dashboard

1. Open [https://webhook.site](https://webhook.site) in your browser.
2. Go to your unique URL — every incoming POST appears in real time.

Each alert payload looks like:

```json
{
  "source": "weather-pipeline",
  "alert_type": "HEAT_ALERT",
  "temperature": 36.2,
  "message": "Temperature 36.2°C exceeds heat threshold 35.0°C",
  "triggered_at": "2024-05-21T10:00:01.123456",
  "location": { "latitude": 48.8566, "longitude": 2.3522 }
}
```

### Option B — Lower the threshold to force an alert (for testing)

```bash
# Edit .env — set heat threshold below the current temperature
HEAT_THRESHOLD=10.0

# Restart
docker compose down && docker compose up --build
```

An alert will fire on the very first tick and appear in your Webhook.site dashboard.

### Option C — Query the alerts table

```bash
docker exec -it weather-pipeline sh
sqlite3 /data/pipeline.db "SELECT * FROM alerts ORDER BY id DESC LIMIT 10;"
```

`webhook_sent = 1` confirms the POST was delivered successfully.

---

## Configuration Reference

All settings live in `.env` (or can be passed as environment variables):

| Variable | Default | Description |
|---|---|---|
| `WEBHOOK_URL` | *(empty)* | **Required.** Your Webhook.site URL |
| `LATITUDE` | `48.8566` | Location latitude |
| `LONGITUDE` | `2.3522` | Location longitude |
| `HEAT_THRESHOLD` | `35.0` | °C above which a HEAT_ALERT fires |
| `FREEZE_THRESHOLD` | `0.0` | °C at or below which a FREEZE_ALERT fires |
| `SPIKE_DELTA` | `5.0` | °C change between readings that triggers a SPIKE_ALERT |
| `POLL_INTERVAL` | `600` | Seconds between API polls |

---

## Stopping the Pipeline

```bash
docker compose down
```

The SQLite database is persisted in a named Docker volume (`pipeline_data`) and survives restarts.

To also remove the stored data:

```bash
docker compose down -v
```

---

## Project Structure

```
weather-pipeline/
├── app/
│   └── pipeline.py        # All pipeline logic (fetch → process → alert → store)
├── .env.example           # Configuration template
├── docker-compose.yml     # Service definition and environment wiring
├── Dockerfile             # Python 3.11-slim image + dependencies
├── requirements.txt       # requests, APScheduler
└── README.md
```

---

## Design Decisions

- **Open-Meteo** was chosen because it requires no API key, has a stable free tier, and returns current conditions in a single lightweight JSON response.
- **APScheduler** provides reliable in-process scheduling without needing a separate cron container or external broker.
- **SQLite** keeps the stack self-contained — no database service to configure or manage.
- **Three independent alert conditions** (heat, freeze, spike) demonstrate different detection patterns: absolute threshold checks and relative delta checks.
- The pipeline **runs once on startup** before the scheduler begins, so the first reading and any immediate alerts appear without waiting for the first interval.
