# 🌐 NetPulse — Your Application Guide

> **NetPulse** is a **Network Reliability Monitoring Platform** — a real-world full-stack application that continuously watches your internet connection and tells you exactly when, why, and how often your network fails.

---

## 🤔 What Problem Does It Solve?

Ever had your internet go down and wondered:
- *Was it my router? My ISP? Just a DNS hiccup?*
- *How long was I offline?*
- *How reliable is my connection actually?*

**NetPulse answers all of that — automatically, continuously, with historical data.**

---

## 🏗️ The Three-Part Architecture

Your app is split into **3 independent pieces** that work together:

```
┌─────────────────────────────────────────────────────────────────┐
│  YOUR COMPUTER                                                  │
│                                                                 │
│  ┌──────────────────┐    POST data     ┌──────────────────┐    │
│  │  Python Agent    │ ──────────────► │  FastAPI Backend  │    │
│  │  (Monitoring)    │                  │  (API + Database) │    │
│  │                  │ ◄────────────── │                   │    │
│  │  Runs every 60s  │   settings/ack   │  SQLite Database  │    │
│  └──────────────────┘                  └──────────────────┘    │
│                                                ▲               │
│                                                │ GET data       │
│                                       ┌──────────────────┐     │
│                                       │  React Dashboard  │     │
│                                       │  (What you see)   │     │
│                                       │  localhost:5173   │     │
│                                       └──────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### 🔵 Part 1 — Python Monitoring Agent (`agent/`)
Runs in the background every 60 seconds. Performs 4 health checks:

| Check | What It Does |
|-------|-------------|
| **Gateway** | Pings your router (192.168.1.1). If ICMP is blocked, tries TCP port 80/443 |
| **DNS** | Resolves `google.com`, `cloudflare.com`, `github.com` — measuring how long each takes |
| **Internet** | Makes HTTPS requests to 3 external servers. 2-of-3 must succeed to call internet "healthy" |
| **Latency/Ping** | Sends 10 pings, calculates min/avg/max latency and packet loss % |

> [!TIP] **Local Buffer:** If the backend is unreachable, the agent stores data in `agent_buffer.db` locally and uploads it when the backend comes back online. This means **you never lose monitoring data** even when your internet is down.

---

### 🟢 Part 2 — FastAPI Backend (`backend/`)
The brain of the operation. Running on **http://127.0.0.1:8000**

**What it does:**
- Receives monitoring data from the agent every 60 seconds
- Stores it in `netpulse.db` (SQLite database)
- **Detects incidents** automatically (e.g., 3 consecutive gateway failures = incident created)
- **Calculates uptime** using the real formula: `((Monitored Time - Downtime) / Monitored Time) × 100`
- Exposes REST APIs for the dashboard to consume

**Interactive API Docs:** [http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs) ← Try your API here!

---

### 🟣 Part 3 — React Dashboard (`frontend/`)
Running on **http://127.0.0.1:5173** — this is what you open in your browser.

---

## 📺 The 6 Dashboard Pages

### 1. 🏠 Overview
**Your mission control.** See everything at a glance:
- 4 KPI cards: overall status, avg latency, packet loss, uptime %
- Live latency area chart (last 20 measurements)
- Subsystem health (gateway / DNS / internet — green/red/amber)
- Recent incidents table

### 2. 🔌 Connectivity
**Deep-dive into each connectivity layer:**
- **Gateway card** — Is your router reachable? ICMP vs TCP fallback badge
- **DNS card** — How many domains resolved? Average resolution time
- **Internet card** — How many external endpoints responded? Quorum status

### 3. 📈 Performance
**Historical latency and packet loss charts:**
- Line chart showing min/avg/max latency over time
- Bar chart showing packet loss percentage per measurement
- Measurement log table with timestamps

### 4. 🚨 Incidents
**Every outage, logged and categorized:**
- Filter by severity: `critical`, `high`, `medium`, `low`
- Filter by type: `gateway`, `dns`, `internet`, `high_latency`, `packet_loss`
- See start time, recovery time, and duration of each incident

### 5. 📊 Reliability
**Your network's SLA report card:**
- Uptime % for last 24h, 7 days, 30 days
- Total outages count, total downtime, average latency
- Uptime trend area chart
- Switch between time periods

### 6. ⚙️ Settings
**Configure the monitoring agent:**
- Ping interval (default: 60s)
- Failure thresholds (how many failures before an incident is raised)
- DNS targets list (which domains to resolve)
- Internet targets list (which URLs to probe)

---

## 🚀 How To Use It Right Now

### Step 1 — Open the Dashboard
Open your browser → **[http://localhost:5173](http://localhost:5173)**

### Step 2 — Feed it data (run the agent)
Open a new terminal in `c:\Users\Dhrubojyoti\Desktop\netplus\` and run:

```powershell
# Run once to see immediate results:
python agent/monitor.py --once

# OR run continuously (every 60 seconds):
python agent/monitor.py
```

### Step 3 — Watch the dashboard update
The dashboard auto-refreshes every **15 seconds**. After a few agent cycles you'll see:
- Latency charts filling up
- Uptime % calculating
- Gateway/DNS/Internet status turning green

### Step 4 — Explore the API
Visit **[http://localhost:8000/docs](http://localhost:8000/docs)** to see and test all API endpoints interactively.

---

## 🗂️ File Map — Where Is Everything?

```
netplus/
├── agent/                    ← Python monitoring agent
│   ├── config.py             ← Configuration (env vars, defaults)
│   ├── monitor.py            ← Main entry point (run this!)
│   ├── monitoring/
│   │   ├── gateway.py        ← Router detection + ping
│   │   ├── dns_probe.py      ← DNS resolution checks
│   │   ├── internet_probe.py ← HTTPS connectivity checks
│   │   └── ping_probe.py     ← Latency + packet loss measurement
│   ├── storage/
│   │   └── local_buffer.py   ← Offline data buffering (SQLite)
│   └── sync/
│       └── uploader.py       ← Syncs buffered data to backend
│
├── backend/
│   └── app/
│       ├── main.py           ← FastAPI app entry point
│       ├── models/           ← Database table definitions (SQLAlchemy)
│       ├── schemas/          ← Request/response shapes (Pydantic)
│       ├── services/         ← Business logic (incidents, reliability)
│       ├── api/v1/endpoints/ ← REST API route handlers
│       └── database/         ← DB connection + session management
│
├── frontend/
│   └── src/
│       ├── App.jsx           ← Router, 6 page routes
│       ├── layouts/          ← Dark sidebar layout
│       ├── pages/            ← Overview, Connectivity, Performance, etc.
│       ├── components/       ← Reusable: StatusBadge, StatCard, etc.
│       └── services/api.js   ← All Axios calls to the backend
│
├── tests/                    ← 50 automated tests (all passing)
├── docker-compose.yml        ← Run everything with Docker
├── netpulse.db               ← Your live database (auto-created)
└── agent_buffer.db           ← Agent offline buffer (auto-created)
```

---

## 🔄 Data Flow — End to End

```
Every 60 seconds:

agent/monitor.py
  │
  ├─► gateway.py        → "Router at 192.168.1.1, ICMP OK, 20ms"
  ├─► dns_probe.py      → "3/3 domains resolved, avg 5ms"
  ├─► internet_probe.py → "3/3 endpoints reachable, avg 766ms"
  └─► ping_probe.py     → "avg 21ms, 0% packet loss"
        │
        ▼
  local_buffer.py  → Save to agent_buffer.db (offline safety)
        │
        ▼
  uploader.py      → POST /api/v1/metrics  →  FastAPI backend
                                                    │
                                        ┌───────────┴───────────┐
                                        │                       │
                                  measurement_service    incident_service
                                  (save to DB)           (detect outages)
                                        │
                                        ▼
                                   netpulse.db
                                        │
                                        ▼
                              React Dashboard polls:
                              GET /api/v1/metrics/latest
                              GET /api/v1/metrics/history
                              GET /api/v1/incidents
                              GET /api/v1/reliability/summary
```

---

## 🌡️ Status Colors Explained

| Color | Meaning |
|-------|---------|
| 🟢 **Green** | Healthy — everything working |
| 🟡 **Amber** | Warning / Degraded — partial failure |
| 🔴 **Red** | Offline / Critical — confirmed failure |
| 🔵 **Blue** | Informational |

---

## 🧪 Running the Tests

```powershell
# Backend + Agent tests (50 tests)
cd c:\Users\Dhrubojyoti\Desktop\netplus
pytest tests -v

# Frontend tests (5 tests)
cd frontend
npm test
```

---

## ⚡ Current Status

| Service | URL | Status |
|---------|-----|--------|
| FastAPI Backend | http://localhost:8000 | ✅ Running |
| React Dashboard | http://localhost:5173 | ✅ Running |
| API Docs | http://localhost:8000/docs | ✅ Available |

> [!IMPORTANT]
> To get data into the dashboard, you must **run the agent** in a terminal:
> ```powershell
> cd c:\Users\Dhrubojyoti\Desktop\netplus
> python agent/monitor.py --once
> ```
