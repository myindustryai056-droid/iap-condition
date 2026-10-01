# 🏭 Industrial Anomaly Detection Platform (IAP)

Real-time anomaly detection for industrial equipment using a multi-method ML ensemble — all self-hosted via Docker Compose.

---

## Architecture

```
┌──────────────┐   MQTT    ┌─────────────┐   asyncpg  ┌──────────────────┐
│  Edge Agent  │──────────▶│  Mosquitto  │           │  TimescaleDB     │
│ (field device│           │  (broker)   │     ┌────▶│  (time-series)   │
└──────────────┘           └─────────────┘     │     └──────────────────┘
                                               │
┌──────────────┐  REST/WS  ┌──────────────┐   │     ┌──────────────────┐
│  React SPA   │◀─────────▶│  FastAPI     │───┤     │  Redis           │
│  (frontend)  │           │  (backend)   │   │     │  (pub-sub cache) │
└──────────────┘           └──────────────┘   │     └──────────────────┘
       ▲                          │            │
       │         Nginx            │ ML Worker  │     ┌──────────────────┐
       └──────────(proxy)─────────┘────────────┘     │  Grafana         │
                                                     │  (monitoring)    │
                                                     └──────────────────┘
```

## Stack

| Layer        | Technology                     |
|--------------|--------------------------------|
| Backend      | FastAPI + asyncpg + SQLAlchemy |
| Database     | PostgreSQL 15 + TimescaleDB    |
| Cache/Pub-sub| Redis 7                        |
| MQTT broker  | Eclipse Mosquitto 2            |
| ML Engine    | Scikit-learn + NumPy/SciPy     |
| Frontend     | React 18 + Vite + Tailwind CSS |
| Reverse proxy| Nginx 1.25                     |
| Monitoring   | Grafana                        |

---

## Quick Start

### 1. Prerequisites

- Docker ≥ 24 and Docker Compose v2
- `make` (optional but recommended)

### 2. Clone & configure

```bash
git clone <repo-url> iap
cd iap

# Edit secrets before first boot
cp .env .env.local
nano .env.local        # change all *_PASSWORD and JWT_SECRET values
```

### 3. Start

```bash
make up
# or: docker compose up -d
```

| Service      | URL                             |
|--------------|---------------------------------|
| Frontend     | http://localhost:3000           |
| Backend API  | http://localhost:8000/docs      |
| Grafana      | http://localhost:3001           |
| pgAdmin      | http://localhost:5050 (dev only)|

### 4. Development mode (with pgAdmin)

```bash
make dev
```

---

## SSL (HTTPS)

Generate a self-signed certificate for local development:

```bash
make ssl-dev
```

For production, follow the instructions in `docker/nginx/ssl/README.md`.

---

## ML Anomaly Detection

The `Anomaly_engine.py` uses a **5-detector ensemble**:

| Detector         | Weight | Best for                            |
|------------------|--------|-------------------------------------|
| Z-Score          | 25 %   | Slow drift & steady-state outliers  |
| IQR              | 20 %   | Robust to non-Gaussian distributions|
| EWMA             | 20 %   | Control-chart style detection       |
| Rate-of-Change   | 15 %   | Sudden spikes / step changes        |
| Isolation Forest | 20 %   | Multivariate / complex patterns     |

---

## Useful Commands

```bash
make logs              # tail all logs
make logs s=backend    # tail backend only
make shell-backend     # bash inside backend container
make shell-db          # psql session
make migrate           # run DB migrations
make test              # run pytest
make lint              # ruff linter
make clean             # destroy everything (volumes too)
```

---

## Project Structure

```
.
├── backend/
│   ├── main.py                    # FastAPI entry point
│   ├── Anomaly_engine.py          # ML detection engine
│   ├── requirements.txt
│   └── app/
│       ├── config.py              # Pydantic settings
│       ├── database.py            # SQLAlchemy + asyncpg
│       ├── models.py              # Pydantic schemas
│       ├── routers/               # API routes
│       ├── services/              # Business logic
│       ├── middleware/            # Security, audit
│       ├── websocket/             # Real-time WS
│       └── ml/                    # Training worker
├── frontend/
│   └── src/
│       ├── components/            # React components
│       ├── contexts/              # Auth context
│       ├── hooks/                 # WebSocket hook
│       ├── pages/                 # Page components
│       └── services/              # API client
├── database/
│   └── schema.sql                 # TimescaleDB schema
├── docker/
│   ├── Dockerfile.backend
│   ├── Dockerfile.frontend
│   ├── nginx/                     # Reverse proxy config + SSL
│   ├── mosquitto/                 # MQTT broker config
│   └── grafana/                   # Dashboards & datasources
├── docker-compose.yml
├── Makefile
└── .env
```

---

## Environment Variables

All secrets live in `.env`. Key variables:

| Variable            | Default                         | Description              |
|---------------------|---------------------------------|--------------------------|
| `POSTGRES_PASSWORD` | `iap_secure_password_change_me` | PostgreSQL password      |
| `REDIS_PASSWORD`    | `redis_password`                | Redis password           |
| `JWT_SECRET`        | `change_this_…`                 | JWT signing key          |
| `MQTT_USER`         | `iap_broker`                    | MQTT broker username     |
| `MQTT_PASS`         | `mqtt_password`                 | MQTT broker password     |
| `SMTP_PASSWORD`     | *(empty)*                       | SendGrid API key         |
| `SLACK_WEBHOOK_URL` | *(empty)*                       | Slack notification hook  |
| `ANOMALY_ENGINE_MODE` | `simple`                       | `simple` (5-detector ensemble, default) or `advanced` (FFT vibration + autoencoder + adaptive threshold + fault classifier, opt-in) |
| `N8N_USER` / `N8N_PASSWORD` | `admin` / `changeme_n8n_password` | n8n web UI login (http://localhost:5678) |
| `N8N_ANOMALY_WEBHOOK_URL` | `http://n8n:5678/webhook/anomaly-alert` | Where anomaly alerts are POSTed for n8n to orchestrate (maintenance ticket, WhatsApp, escalation, ...). Failing silently (non-blocking) if n8n has no matching workflow yet. |

> ⚠️  **Change all default passwords before deploying to production.**

---

## Machine Connection Manager (SCAN MACHINES)

`POST /api/v1/connection/scan` — scans the local network for industrial devices across
every supported protocol (Modbus TCP, OPC-UA, EtherNet/IP, MQTT brokers, optional PROFINET
identification) and returns each candidate with a suggested connection method and a
GREEN/ORANGE/RED compatibility hint (PROFINET is ORANGE by default — real process data
needs a PROFINET→Modbus/OPC-UA gateway; see `backend/app/edge/connection_manager.py`).

`POST /api/v1/connection/test` — attempts to actually connect to one chosen device and
returns a plain-language success/failure message (never a raw exception).

This is the single source of truth for protocol drivers, shared by both the backend
(REST endpoints above) and `agent.py` (the actual collection loop) — no duplicated
driver code between the two.

---

## Remote Machine Compatibility Check (pre-sale)

`POST /api/v1/compatibility/check` (multipart form) — a customer/sales user submits machine name,
manufacturer, model, known protocol and optional photos (PLC / ports / HMI). Answer:
**GREEN** (direct), **ORANGE** (adapter/gateway) or **RED** (technical integration needed first).

- The verdict comes from structured fields + the knowledge base in
  `backend/app/services/compatibility_kb.py` — a *starting* set that is meant to grow.
- **Photos are stored for manual technical review only — there is no automatic image analysis.**
- Every request is saved in `compatibility_checks`; `GET /api/v1/compatibility/checks?only_unreviewed=true`
  lists unknown machines so they can be reviewed and added to the KB.
- Existing databases: `schema.sql` only runs on a fresh volume — run the `compatibility_checks`
  `CREATE TABLE` block from `database/schema.sql` manually on an already-initialised database.

## Remote Diagnostic Package

- `POST /api/v1/diagnostics/analyze/{device_id}` — real analysis from actual data (connection freshness,
  data quality, open anomalies). Every score deduction is listed in `factors`.
- `POST /api/v1/diagnostics/system-package` → builds a zip (`diagnostic.json` + `summary.txt`) with system
  status, machines, protocol status, communication errors, recent logs, configuration summary and anomaly
  information; `GET /api/v1/diagnostics/package/download/{filename}` downloads it.
- **Privacy by design:** generated on demand by an authenticated user, **never sent automatically**, and it
  contains no raw telemetry values and no raw anomaly snapshots. The customer decides whether to send it.

---

## Simulation / Demo mode

Page **Simulation** (sidebar) — press *Démarrer la simulation*: 3 virtual machines (hydraulic pump,
compressor, conveyor motor) produce healthy readings. Then trigger faults (bearing wear, overheating,
pressure drop, sensor drift, motor overcurrent, communication loss) and watch the live chart, the
detections (severity, method, score, **detection delay**), alert channels and the timeline.

**What is simulated vs real** — only the machines and their readings are simulated (flagged `[SIM]`,
`devices.config.simulated = true`, `telemetry.metadata.simulated = true`). Readings go into the real
`telemetry` table; the real detection loop, ML engine, alert service, notification channels and n8n
webhook react to them. Two demo-only accelerations, both shown in the UI: detection runs every 5 s
instead of 30 s, and fault ramps are faster than real wear.

- API: `/api/v1/simulation/{status,start,stop,trigger,clear,reset}` (admin/manager only).
- *Corriger la panne* stops the fault and closes its open anomalies (like a technician would).
- *Réinitialiser* deletes all simulated devices, telemetry, anomalies and notification logs.
- Set `SIMULATION_ENABLED=false` on customer units so demo data can never mix with production monitoring.

### Pipeline fixes found while building the simulator
- Alerts never left the system: the background loop consumed a different `AlertService()` instance than
  the one the detector fed, and the alert service queried a non-existent `alert_configs` table.
- `notification_logs` recorded `sent` for every requested channel; it now records the real outcome
  (`sent` / `failed` / `skipped` = channel not configured).
- New rule-based **communication_loss** alert (last 10 readings ≥ 80% invalid) — ML detectors ignore
  NULL readings, so a dead machine raised nothing before.

---

## Reports: Gmail (free, no paid provider) + local llama3.1

### 1. One-time setup — Google OAuth Client (free, ~5 min)
1. https://console.cloud.google.com/ → create a project (or reuse one).
2. **APIs & Services → Library** → enable **Gmail API** (free, no billing account needed).
3. **APIs & Services → OAuth consent screen** → External → fill app name/email → add scope
   `.../auth/gmail.send` → add your own Gmail as a test user (while in "Testing" mode).
4. **APIs & Services → Credentials → Create Credentials → OAuth client ID** → type **Web application** →
   Authorized redirect URI: `http://<box-ip-or-domain>:8000/api/v1/integrations/gmail/callback`.
5. Copy the Client ID/Secret into `.env`:
   ```
   GMAIL_OAUTH_CLIENT_ID=...
   GMAIL_OAUTH_CLIENT_SECRET=...
   GMAIL_OAUTH_REDIRECT_URI=http://<box-ip-or-domain>:8000/api/v1/integrations/gmail/callback
   FRONTEND_URL=http://<box-ip-or-domain>:3000
   ```
6. Restart the backend, open **Reports**, click **Connecter Gmail**, approve on Google's screen.
   The connected Gmail account is the *sender*; recipients are simply your app users with role
   `manager` / `technician` — create those accounts first (`POST /api/v1/auth/register` or the Users page),
   otherwise reports generate but nobody receives them (`email_status: failed:no_recipients`).

While the OAuth consent screen is in **Testing** mode (the default, free, no Google review needed),
only the test users you added can authorize it — fine for a single company mailbox. Publishing the
app for public use requires Google's verification review; not needed here.

### 2. One-time setup — pull the LLM (free, ~5 GB download, then fully offline)
```
docker compose up -d ollama
docker compose exec ollama ollama pull llama3.1:8b-instruct-q4_0
```
This quantized variant needs **~5-6 GB RAM** — workable on an 8 GB Raspberry Pi 5 but tight
alongside Postgres/Redis/n8n/backend; on a 4 GB Pi it will not fit. If reports keep coming back
as "modèle indisponible — version brute" check `docker compose logs ollama` and confirm the pull
finished. A smaller/faster model can be set via `REPORT_LLM_MODEL` in `.env` (any tag `ollama pull`
accepts) — the fallback (plain templated report, clearly labeled) fires automatically whenever
Ollama is unreachable or the model isn't pulled, so reports are never silently blocked on the LLM.

### 3. What happens automatically
- **Daily report**: every day at `DAILY_REPORT_HOUR` (default 8, box's local time), llama3.1 writes a
  summary from real data (connected machines, anomalies, data quality) and e-mails it to every
  `manager`/`technician` user via the connected Gmail.
- **Immediate anomaly report**: any `high`/`critical` anomaly triggers its own short llama3.1-written
  alert e-mail to `manager` users only, in addition to the existing alert channels (email/Slack/n8n) —
  fire-and-forget, never delays the normal alert pipeline.
- Every report is stored in `reports` (PDF via reportlab) — **Reports** page lists and downloads them.

### Known gaps
- Gmail free tier send quota (~500/day per account) is far above what this box needs, but is a hard
  ceiling if you also send other mail from the same account.
- No token-revocation detection: if you revoke access from Google's side, the box only notices on the
  next send attempt (`last_error` on the Gmail card) — no proactive check yet.
