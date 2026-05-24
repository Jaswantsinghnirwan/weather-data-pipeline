<div align="center">

# 🌡️ Weather Data Pipeline

### An automated, event-driven weather monitoring system
### that detects anomalies and fires real-time alerts

![Python](https://img.shields.io/badge/Python-3.11-blue?style=for-the-badge&logo=python&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containerised-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-Local%20Storage-003B57?style=for-the-badge&logo=sqlite&logoColor=white)
![Open-Meteo](https://img.shields.io/badge/Open--Meteo-Free%20API-FF6B35?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)

**Fetches → Stores → Detects → Alerts. Fully automated. Zero manual steps.**

[Features](#-features) • [Quick Start](#-quick-start) • [How It Works](#-how-it-works) • [Configuration](#-configuration) • [Verify It's Working](#-verify-its-working) • [Project Structure](#-project-structure)

</div>

---

## 📌 What Is This?

A lightweight **event-driven data pipeline** built with entirely free and open-source tools.

It automatically:
- 📡 **Collects** real-time weather data from the [Open-Meteo API](https://open-meteo.com/) on a recurring schedule
- 💾 **Stores** every reading in a local SQLite database
- 🔍 **Detects** anomalies — heat alerts, freeze alerts, and sudden temperature spikes
- 🚨 **Fires webhook alerts** instantly when a condition is met
- 🐳 **Runs anywhere** with a single Docker command — no setup, no accounts, no API keys

---

## ✨ Features

| Feature | Detail |
|---|---|
| ⏱️ Scheduled Polling | Fetches weather data every N seconds (configurable) |
| 🌡️ Heat Alert | Fires when temperature exceeds a defined threshold |
| 🥶 Freeze Alert | Fires when temperature drops to or below freezing |
| ⚡ Spike Alert | Fires when temperature changes suddenly between two readings |
| 💾 Persistent Storage | Every reading and alert saved to SQLite with timestamps |
| 🔔 Webhook Delivery | Instant POST to any webhook URL (Webhook.site, Slack, etc.) |
| 🐳 One-Command Launch | Fully containerised with Docker Compose |
| ⚙️ Zero Code Changes | All config via environment variables in `.env` |

---

## 🚀 Quick Start

### Prerequisites
- [Docker Desktop](https://docs.docker.com/get-docker/) installed and running

That's the only requirement.

---

### Step 1 — Get a Free Webhook URL

1. Open **[https://webhook.site](https://webhook.site)** in your browser
2. You'll instantly get a unique URL like:
   ```
   https://webhook.site/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
   ```
3. Keep this tab open — alerts will appear here in real time

---

### Step 2 — Configure

```bash
cp .env.example .env
```

Open `.env` and set your webhook URL and location:

```env
WEBHOOK_URL=https://webhook.site/YOUR-UNIQUE-ID-HERE
LATITUDE=26.2389
LONGITUDE=73.0243
HEAT_THRESHOLD=10.0
POLL_INTERVAL=30
```

> 💡 Set `HEAT_THRESHOLD=10.0` and `POLL_INTERVAL=30` to see alerts fire immediately during testing.

---

### Step 3 — Run

```bash
docker compose up --build
```

**That's it.** The pipeline starts, fetches live weather data, and begins monitoring.

To run in the background:
```bash
docker compose up --build -d
```

---

## ⚙️ How It Works

Every tick (every N seconds), the pipeline runs 4 steps:

```
┌──────────────────────────────────────────────────────────────────┐
│                        Docker Container                           │
│                                                                  │
│   APScheduler ──► triggers every N seconds                       │
│         │                                                        │
│         ▼                                                        │
│   [1] FETCH   ───────────────► Open-Meteo API                   │
│         │                      (temperature, wind, weather code) │
│         ▼                                                        │
│   [2] STORE   ───────────────► SQLite: readings table            │
│         │                                                        │
│         ▼                                                        │
│   [3] DETECT  ───────────────► check_conditions()               │
│         │                      • temp >= HEAT_THRESHOLD?         │
│         │                      • temp <= FREEZE_THRESHOLD?       │
│         │                      • |change| >= SPIKE_DELTA?        │
│         │                                                        │
│         ▼  (if any condition triggered)                         │
│   [4] ALERT   ───────────────► POST JSON to Webhook URL          │
│                └─────────────► SQLite: alerts table              │
└──────────────────────────────────────────────────────────────────┘
```

### Alert Types

| Alert | Condition | Type |
|---|---|---|
| `HEAT_ALERT` | Temperature ≥ `HEAT_THRESHOLD` (default 35°C) | Absolute threshold |
| `FREEZE_ALERT` | Temperature ≤ `FREEZE_THRESHOLD` (default 0°C) | Absolute threshold |
| `SPIKE_ALERT` | \|current − previous\| ≥ `SPIKE_DELTA` (default 5°C) | Relative delta |

All three checks run **independently** on every tick. Multiple alerts can fire simultaneously.

### Sample Alert Payload

When an alert fires, this JSON is POSTed to your webhook URL:

```json
{
  "source": "weather-pipeline",
  "alert_type": "HEAT_ALERT",
  "temperature": 38.5,
  "message": "Temperature 38.5°C exceeds heat threshold 35.0°C",
  "triggered_at": "2026-05-25T10:00:01.123456",
  "location": {
    "latitude": 26.2389,
    "longitude": 73.0243
  }
}
```

---

## 🔧 Configuration

All settings live in `.env`. No code changes ever needed.

| Variable | Default | Description |
|---|---|---|
| `WEBHOOK_URL` | *(empty)* | **Required** — Your Webhook.site or any webhook URL |
| `LATITUDE` | `48.8566` | Location latitude (default: Paris) |
| `LONGITUDE` | `2.3522` | Location longitude (default: Paris) |
| `HEAT_THRESHOLD` | `35.0` | °C — fires `HEAT_ALERT` when temp ≥ this |
| `FREEZE_THRESHOLD` | `0.0` | °C — fires `FREEZE_ALERT` when temp ≤ this |
| `SPIKE_DELTA` | `5.0` | °C — fires `SPIKE_ALERT` when change ≥ this |
| `POLL_INTERVAL` | `600` | Seconds between API polls (use `30` for demos) |
| `DB_PATH` | `/data/pipeline.db` | Path to SQLite database inside container |

---

## ✅ Verify It's Working

### Verify scheduled ingestion is running

**Option 1 — Watch the live logs:**
```bash
docker compose logs -f
```

You'll see a new tick every N seconds:
```
2026-05-25 10:00:00 [INFO] === Weather Pipeline starting ===
2026-05-25 10:00:00 [INFO] Location: 26.2389, 73.0243 | Poll interval: 30s
2026-05-25 10:00:01 [INFO] ▶ Pipeline tick started
2026-05-25 10:00:01 [INFO] Fetched → temp=38.5°C  wind=14.0 km/h  code=0
2026-05-25 10:00:01 [WARNING] 🚨 HEAT_ALERT — Temperature 38.5°C exceeds threshold 10.0°C
2026-05-25 10:00:01 [INFO] Webhook delivered → HTTP 200
2026-05-25 10:00:01 [INFO] ◀ Pipeline tick complete
2026-05-25 10:00:31 [INFO] ▶ Pipeline tick started  ← next scheduled run
```

**Option 2 — Query the database:**
```bash
# Enter the container
docker exec -it weather-pipeline sh

# View latest readings (with clean column formatting)
sqlite3 /data/pipeline.db -column -header \
  "SELECT * FROM readings ORDER BY id DESC LIMIT 5;"
```

Each row = one successful fetch. Rows growing over time = scheduler is working.

---

### Verify webhook alerts are being triggered

**Option 1 — Webhook.site dashboard:**

Open your Webhook.site URL in a browser. Every alert appears as a new POST request in real time with the full JSON payload.

**Option 2 — Query the alerts table:**
```bash
docker exec -it weather-pipeline sh
sqlite3 /data/pipeline.db -column -header "SELECT * FROM alerts;"
```

`webhook_sent = 1` confirms the POST was delivered successfully.

**Option 3 — Force an alert immediately (for testing):**

Set `HEAT_THRESHOLD=1.0` in `.env`, then restart:
```bash
docker compose down && docker compose up --build
```
An alert will fire on the very first tick.

---

## 📁 Project Structure

```
weather-pipeline/
│
├── app/
│   └── pipeline.py          # Core pipeline — fetch, store, detect, alert
│
├── .env.example             # Configuration template (copy to .env)
├── .gitignore               # Ensures .env is never committed
├── docker-compose.yml       # Service, volumes, and environment wiring
├── Dockerfile               # Python 3.11-slim + SQLite + dependencies
├── requirements.txt         # requests, APScheduler
└── README.md
```

---

## 🗄️ Database Schema

**`readings` table** — every data fetch

| Column | Type | Description |
|---|---|---|
| `id` | INTEGER | Auto-increment primary key |
| `fetched_at` | TEXT | UTC timestamp |
| `temperature` | REAL | °C |
| `windspeed` | REAL | km/h |
| `weathercode` | INTEGER | WMO weather code |

**`alerts` table** — every triggered alert

| Column | Type | Description |
|---|---|---|
| `id` | INTEGER | Auto-increment primary key |
| `triggered_at` | TEXT | UTC timestamp |
| `alert_type` | TEXT | HEAT_ALERT / FREEZE_ALERT / SPIKE_ALERT |
| `temperature` | REAL | Temperature that triggered the alert |
| `message` | TEXT | Human-readable alert description |
| `webhook_sent` | INTEGER | 1 = delivered, 0 = failed |

---

## 🛑 Stopping the Pipeline

```bash
# Stop (data is preserved)
docker compose down

# Stop and delete all stored data
docker compose down -v
```

The SQLite database lives in a named Docker volume (`pipeline_data`) and survives container restarts.

---

## 🧠 Design Decisions

| Decision | Reason |
|---|---|
| **Open-Meteo API** | Free, no API key, stable. Zero friction for anyone running the project. |
| **APScheduler** | Runs inside the Python process — no separate cron container or OS config needed. |
| **SQLite** | File-based, zero setup, self-contained. Perfect for a single-service pipeline. |
| **Three alert types** | Demonstrates two detection strategies: absolute thresholds (heat/freeze) and relative delta (spike). |
| **Runs on startup** | Pipeline fetches immediately on launch — no waiting for the first interval. |
| **webhook_sent flag** | Alerting and storage are decoupled — a failed webhook never causes data loss. |
| **All config in .env** | Zero code changes needed to change location, thresholds, or timing. |

---

## 🔮 Future Improvements

- [ ] FastAPI dashboard to visualise readings in real time
- [ ] Retry logic with exponential backoff for failed webhooks
- [ ] Email as a second alert channel
- [ ] Moving average anomaly detection (flags readings > 2 std deviations from recent mean)
- [ ] Swap SQLite → PostgreSQL for multi-service scalability
- [ ] Unit tests for `check_conditions()` logic

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

<div align="center">

Built with ❤️ using Python • Docker • Open-Meteo • SQLite • APScheduler

</div>
