# Extreme Heatwave Early Warning & Human Thermal Stress Index
## Engineering Blueprint — SIH Problem Statement 26083

Team size: 6 | Timeline: 5 days | Skill level: beginner, AI-assisted

---

## 1. Precise MVP Definition

**"A web dashboard that shows a live, color-coded map of a city's wards, where each ward's color reflects a Heat Index-based thermal stress risk score adjusted for local elderly/outdoor-worker vulnerability, with a 3–5 day forecast, an automated plain-language public health advisory per ward, and a simple ML model that forecasts near-term risk escalation — with a mock SMS/WhatsApp alert trigger to demonstrate the automation pipeline."**

Concretely, a judge should be able to:
1. Open the dashboard and see a map with 8–15 wards of one Indian city, colored Green→Yellow→Orange→Red→Maroon.
2. Click a ward and see: current temp/humidity/wind, computed Heat Index/WBGT, the risk score breakdown (base heat score + vulnerability adjustment), a 3–5 day forecast chart, and an auto-generated advisory.
3. Trigger a "Send Alert" button and see a simulated SMS/WhatsApp message payload.

Everything else is a bonus.

---

## 2. Feature Prioritization

**Must Have (P0 — the demo does not exist without these)**
- Live weather ingestion for a fixed set of ward coordinates (temp, RH, wind, solar radiation where available)
- Heat Index calculation (Rothfusz regression) as the primary explainable metric
- Composite risk score (0–100) + risk category (5 bands)
- Static ward-level vulnerability dataset (elderly %, outdoor worker %) merged into the score
- Leaflet map with choropleth/marker coloring by risk category
- Ward detail panel: current conditions + score breakdown
- 3–5 day forecast (can be sourced directly from weather API, ML enhances it)
- Automated advisory text per risk category
- One working ML component (even a simple regression) that visibly contributes to the forecast
- Deployed/runnable dashboard (localhost is fine for judging)

**Should Have (P1 — do these if P0 is done early)**
- Simplified WBGT as a secondary "outdoor worker" metric
- Historical trend chart (last 7 days) per ward
- Mock alert simulation with realistic SMS/WhatsApp JSON payload shown in UI
- Ward search/filter and a legend with explainable score breakdown UI (bar chart of contributing factors)
- Confidence indicator on ML forecast

**Nice to Have (P2 — only if time remains on Day 4)**
- Real SMS via Twilio trial account to a verified test number
- Multi-city switcher
- Mobile-responsive layout / PWA manifest
- Simple login for a "municipal admin" view
- Downloadable PDF/CSV report of ward risk

**Do NOT Build**
- Any model that outputs actual predicted death counts or hospitalization numbers — you have no validated epidemiological dataset for this in 5 days
- Custom deep learning / neural forecasting models — no time to tune, and a Random Forest or even linear regression will look identical in a 5-min demo
- Your own PostGIS/GeoServer GIS stack — Leaflet + static GeoJSON is enough
- A custom weather forecasting model — always consume an existing forecast API
- Microservices, Docker orchestration, Kubernetes, CI/CD pipelines — one deployable backend + one frontend is enough
- Full authentication/role systems, payment, multi-tenant infra
- Real government SMS gateway integration — simulate it and say so honestly

---

## 3. Recommended Technology Stack (with justification)

| Layer | Choice | Why |
|---|---|---|
| Frontend | **Vanilla HTML/CSS/JS + Bootstrap 5 + Leaflet.js + Chart.js** | Zero build tooling, zero npm/webpack failure modes, every AI coding assistant is excellent at this, fastest to get on screen. Use React *only* if someone on the team already knows it well — otherwise it burns a full day on setup/debugging for beginners. |
| Backend | **Python + FastAPI** | Auto-generated interactive docs (`/docs`) which is huge for a 6-person team integrating in parallel; same language as the ML component so no context-switching; simple `uvicorn` run command. |
| Scheduler | **APScheduler** (in-process) | No external cron/queue infra needed; runs inside the FastAPI process. |
| Database | **SQLite** (via SQLAlchemy) | Zero setup, single file, trivial to reset/seed, more than enough for hackathon data volumes. Do not use Postgres/Mongo unless someone already knows it — it is not worth the setup risk. |
| ML | **scikit-learn** (Linear Regression / Random Forest Regressor) | Trains in seconds on small data, no GPU, extremely well documented, AI assistants generate correct code for it reliably. |
| Weather data | **Open-Meteo API** (primary), OpenWeatherMap (backup) | Open-Meteo needs **no API key**, gives current + hourly + 7-day forecast + historical archive + shortwave solar radiation, all free, no rate-limit headaches during a hackathon. |
| Maps | **Leaflet.js + OpenStreetMap tiles + static GeoJSON** | No API key, no billing account, renders GeoJSON polygons/markers natively, huge tutorial base for AI assistants to draw from. |
| Hosting (optional) | Backend: Render/Railway free tier. Frontend: same server (FastAPI can serve static files) or GitHub Pages. | Avoid deploying at all until Day 4 evening — local demo is acceptable and safer. |

**Rule of thumb for a beginner team:** every extra technology you add is a new way for the demo to break on stage. Stick to this stack even if a fancier one is tempting.

---

## 4. System Architecture

```
                         ┌─────────────────────────┐
                         │   Open-Meteo Weather API │
                         │ (current+forecast+hist.) │
                         └────────────┬─────────────┘
                                      │ HTTP GET (per ward lat/lon)
                                      ▼
                    ┌──────────────────────────────────┐
                    │   APScheduler Job (every 30–60m)   │
                    │   services/weather_fetcher.py      │
                    └────────────┬───────────────────────┘
                                 ▼
                    ┌──────────────────────────────────┐
                    │  services/thermal_index.py          │
                    │  Heat Index / simplified WBGT calc  │
                    └────────────┬───────────────────────┘
                                 ▼
   ┌────────────────────┐    ┌──────────────────────────────┐
   │ data/vulnerability. │───▶│ services/risk_engine.py       │
   │ csv (static, per    │    │ composite risk score+category │
   │ ward)                │    │ + advisory lookup             │
   └────────────────────┘    └────────────┬──────────────────┘
                                            ▼
                                 ┌──────────────────────┐
                                 │   SQLite (db.sqlite3) │
                                 │ wards / weather_      │
                                 │ readings / risk_      │
                                 │ scores / forecasts     │
                                 └───────────┬────────────┘
                                             ▼
                            ┌────────────────────────────┐
                            │   ml/train_model.py (offline)│
                            │   services/ml_model.py       │
                            │   (loads model.pkl, predicts)│
                            └───────────────┬───────────────┘
                                            ▼
                            ┌────────────────────────────┐
                            │   FastAPI REST API (main.py) │
                            │  /api/wards, /api/wards/{id},│
                            │  /api/risk-map, /api/alerts  │
                            └───────────────┬───────────────┘
                                            ▼
                    ┌────────────────────────────────────────┐
                    │  Frontend (Leaflet map + Chart.js +      │
                    │  Bootstrap dashboard) — fetch() to API   │
                    └────────────────────────────────────────┘
```

---

## 5. Complete Data Flow

1. **Seed step (once):** `data/wards.geojson` + `data/vulnerability.csv` are loaded into the `wards` table at startup (id, name, centroid lat/lon, boundary, elderly_pct, outdoor_worker_pct, vulnerability_index).
2. **Ingestion (scheduled every 30–60 min, or on-demand button in demo):** for each ward centroid, call Open-Meteo `/v1/forecast` with `current` + `hourly` + `daily` parameters → raw JSON → insert row into `weather_readings`.
3. **Computation (triggered right after ingestion):** for each new weather_reading, compute Heat Index (and simplified WBGT) → compute base heat score (0–100) from category thresholds → pull ward's `vulnerability_index` → compute composite risk score and category → insert into `risk_scores` → look up advisory text template → attach.
4. **Forecast (ML):** `services/ml_model.py` loads `model.pkl`, takes the last N days of weather_readings for a ward (+ Open-Meteo's own 5-day forecast values as features) → predicts next 3–5 days' Heat Index/risk category → insert into `forecasts` table.
5. **API layer:** exposes current state + history + forecast + advisory as JSON.
6. **Frontend:** on load, calls `/api/risk-map` (GeoJSON FeatureCollection with a `risk_category`/`color` property per ward) → Leaflet renders choropleth. On ward click, calls `/api/wards/{id}` → renders detail panel + Chart.js line chart of forecast.
7. **Alert simulation:** "Send Alert" button on a ward → POST `/api/alerts/simulate` → backend builds a realistic SMS/WhatsApp text payload from the advisory template → returns it (and optionally actually sends via Twilio trial if configured) → logged in `alerts_log`.

---

## 6. Weather APIs & Datasets

- **Primary: Open-Meteo** (`https://api.open-meteo.com`) — no key required, gives current conditions, hourly and 7-day forecast, and a historical archive endpoint (`archive-api.open-meteo.com`) for training data; includes `shortwave_radiation`, `relative_humidity_2m`, `windspeed_10m`, `temperature_2m`. This should be your only dependency for weather.
- **Backup: OpenWeatherMap** (free tier, needs API key, 60 calls/min) — use only if Open-Meteo has an outage during the demo.
- **Optional/advanced: IMD (India Meteorological Department)** open data / Mausam portal — more India-specific but access and formats are inconsistent; **do not depend on this for the MVP**, mention it only as a "future integration" in the pitch.
- **Demographic/vulnerability data:** Census of India 2011 ward/town data (population, age structure) where obtainable; if not obtainable at ward granularity in time, **explicitly build a synthetic-but-realistic CSV** (label it as such in the report) with elderly % and estimated informal/outdoor worker % per ward, sourced from plausible city-level ratios (e.g., city-wide elderly % from Census applied with reasonable ward-level variance). Being upfront that this is illustrative is far safer than presenting invented numbers as real.

---

## 7. GIS / Map Technology

- **Leaflet.js** + **OpenStreetMap** tile layer (no key, no billing).
- Represent wards as a **GeoJSON FeatureCollection** — either:
  - Real ward boundaries if you can find/export them (city GIS portal, OSM boundary extracts, `datameet`/`OpenCity` India GIS community datasets), or
  - **Simplified circles or a synthetic grid** of 8–15 cells over the chosen city if real polygons aren't findable in time (fully acceptable fallback — see §22).
- Color each feature via a `style` function keyed on `risk_category` (Leaflet's standard choropleth pattern — one `L.geoJSON(data, {style: styleFn, onEachFeature: ...})` call).
- No server-side GIS engine (PostGIS/GeoServer) needed — all done client-side from the API's GeoJSON response.

---

## 8. Thermal Stress Metric — Exact Methodology

**Primary metric: Heat Index (HI)** — NWS Rothfusz regression, well-documented, needs only temperature and relative humidity (both reliably available from Open-Meteo), and is explainable to judges with a public NWS reference.

Formula (T in °F, R = relative humidity in %):

```
HI = -42.379 + 2.04901523*T + 10.14333127*R
     - 0.22475541*T*R - 0.00683783*T² - 0.05481717*R²
     + 0.00122874*T²*R + 0.00085282*T*R² - 0.00199788*T²*R²
```

- Convert Open-Meteo °C to °F before applying: `T_F = T_C * 9/5 + 32`.
- Apply the standard NWS adjustments for low-humidity (R<13%, T between 80–112°F, subtract a correction term) and high-humidity (R>85%, T between 80–87°F, add a correction term) — these are documented on the NWS Heat Index page; include them for correctness but they rarely trigger in Indian heatwave conditions.
- Convert result back to °C for display: `HI_C = (HI_F - 32) * 5/9`.
- Below T=80°F (26.7°C), just use the simpler Steadman approximation or actual T, since Rothfusz is only valid above that threshold.

**Secondary metric: simplified outdoor WBGT** (Australian Bureau of Meteorology approximation), useful for the "outdoor worker" narrative since it partially reflects humidity load:

```
e = (RH/100) * 6.105 * exp(17.27*T / (237.7 + T))      [T in °C, vapor pressure in hPa]
WBGT ≈ 0.567*T + 0.393*e + 3.94
```

If you have shortwave solar radiation from Open-Meteo, you can optionally nudge WBGT upward on high-radiation hours (e.g., +1–2°C when `shortwave_radiation` is in the top quartile for that day) and note this is a simplification, not the full ISO 7243 WBGT (which requires a globe thermometer measurement you don't have).

**Recommendation:** lead with Heat Index as your defensible, standard metric; show WBGT as a secondary "feels-like for outdoor workers" number. Do not claim UTCI — it requires mean radiant temperature and a full biometeorological model that is out of scope for 5 days.

---

## 9. Converting Thermal Stress into an Explainable Risk Score

Step 1 — **Base Heat Score (0–100)** from NWS Heat Index categories, linearly interpolated within each band so the score moves smoothly, not in jumps:

| HI (°C) | Category | Score range |
|---|---|---|
| < 27 | Normal | 0–20 |
| 27–32 | Caution | 20–40 |
| 32–39 | Extreme Caution | 40–60 |
| 39–51 | Danger | 60–85 |
| > 51 | Extreme Danger | 85–100 |

Step 2 — **Vulnerability Index (0–1)** per ward:
```
vulnerability_index = 0.5 * norm(elderly_pct) + 0.5 * norm(outdoor_worker_pct)
```
(normalize each 0–1 across your ward set; weights are a transparent, tunable design choice — state this openly in the demo.)

Step 3 — **Composite Risk Score**:
```
composite_score = min(100, base_score * (1 + 0.3 * vulnerability_index))
```
The `0.3` cap means vulnerability can amplify the base heat score by up to 30% — keeps the score dominated by actual weather (defensible) while still visibly reflecting demographic risk.

Step 4 — Map `composite_score` back to a **5-band risk category** (Green/Yellow/Orange/Red/Maroon) using the same thresholds as Step 1, for map coloring and advisory lookup.

**Explainability in the UI:** always show the three numbers side by side — base heat score, vulnerability multiplier, final score — as a small horizontal bar breakdown. This single UI element does more to impress judges than any model complexity.

---

## 10. Incorporating Demographic Vulnerability

- Build `data/vulnerability.csv`: `ward_id, ward_name, population, elderly_pct, outdoor_worker_pct`.
- Source real numbers where you can (city census handbook, ward-level socioeconomic surveys); otherwise construct **reasoned synthetic estimates** and label them "illustrative" in both the UI (small info icon) and the presentation.
- Keep the vulnerability index **static** for the hackathon (it doesn't need to update in real time) — only the weather-driven score updates dynamically. This is an honest and sufficient scope for 5 days.
- Optionally add a UI toggle "show heat-only score vs. vulnerability-adjusted score" — this single toggle visually proves the core value proposition of the whole problem statement in about 3 seconds.

---

## 11 & 12. Realistic ML Approach (4 days) — What It Predicts and What It Needs

**What to build:** a supervised regression/classification model that predicts **next-day and 3-day-ahead Heat Index (or risk category)** per ward, using recent weather trends as features. This is realistic, explainable, and directly relevant — **not** a mortality model.

- **Model:** `RandomForestRegressor` (or `LinearRegression` as a fallback if time is short) from scikit-learn, predicting `HI(t+1)`, `HI(t+2)`, `HI(t+3)` (either as 3 separate small models or one multi-output regressor).
- **Features:** lag features from the last 3 days per ward — `temp(t), temp(t-1), temp(t-2), humidity(t), humidity(t-1), wind(t), month, day_of_year, ward_id (one-hot or target-encoded)`. Optionally also feed in Open-Meteo's own forecast values as a feature — the model then learns a *residual correction*, which is realistic and easy to defend ("we augment the raw forecast with a locally-trained correction model").
- **Training data:** pull 1–2 years of **historical** weather via Open-Meteo's archive API for each ward's centroid (`archive-api.open-meteo.com/v1/archive`), compute HI for each day, and train offline (`ml/train_model.py`) — save the model as `model.pkl` with `joblib`.
- **At inference time:** `services/ml_model.py` loads `model.pkl`, builds the feature row from the last 3 days in `weather_readings`, predicts forward, and writes to the `forecasts` table.
- **If time runs out:** a legitimate, honest fallback is to use Open-Meteo's own 7-day forecast directly as your "3–5 day forecast," and present the ML model purely as a same-day "risk escalation likelihood" classifier (Logistic Regression: will risk category increase by tomorrow, yes/no) trained on the historical data — this is a smaller, still-genuine ML contribution that's very safe to finish in a day.

**Report honestly:** show a simple train/test split metric (e.g., MAE in °C) on a slide — do not claim high accuracy without having actually measured it.

---

## 13. Avoiding Scientifically Unsupported Mortality Claims

- **Never** output a specific predicted number of deaths or hospitalizations. You do not have a validated, calibrated epidemiological model, and presenting invented numbers as predictions would be scientifically indefensible and could mislead judges evaluating for real-world applicability.
- Instead, present a **"Mortality Risk Index"** as an explicitly **relative, ordinal, illustrative category** (Low/Moderate/High/Severe/Extreme) derived transparently from the Heat Index + vulnerability composite score you already computed — not a separately "predicted" health outcome.
- Add a visible disclaimer in the UI and in the pitch: *"This index reflects relative heat-stress risk based on established thermal comfort science (NWS Heat Index) and demographic exposure factors. It is a decision-support prioritization tool, not a clinical or epidemiological mortality forecast."*
- If you want to reference real-world grounding, **cite published research findings in general terms** (e.g., studies have found mortality risk rises measurably above certain Heat Index/WBGT thresholds) without presenting your own tool as having derived or validated those numbers itself.
- This framing is actually a *strength* in front of judges — it shows scientific maturity rather than a weakness.

---

## 14. Backend Architecture

```
backend/app/
  main.py                 # FastAPI app, route registration, startup event (seed DB)
  db.py                   # SQLAlchemy engine/session
  models.py                # ORM models: Ward, WeatherReading, RiskScore, Forecast, Advisory, AlertLog
  schemas.py                # Pydantic request/response models
  services/
    weather_fetcher.py      # calls Open-Meteo, returns normalized dict
    thermal_index.py         # heat_index(), wbgt() pure functions
    risk_engine.py            # compute_composite_score(), categorize()
    vulnerability.py           # loads/serves vulnerability.csv
    advisory.py                # risk_category -> advisory text template
    ml_model.py                 # load model.pkl, predict_forecast(ward_id)
    alerts.py                    # build_alert_payload(), (optional) send via Twilio
  scheduler.py               # APScheduler job wiring weather_fetcher -> thermal_index -> risk_engine
  seed.py                    # one-time loader for wards.geojson + vulnerability.csv
```
Run with `uvicorn app.main:app --reload`. All business logic lives in `services/` as small pure(ish) functions — this makes it trivial for 6 people to work in parallel without merge conflicts, and easy for an AI coding assistant to generate/test each file independently.

---

## 15. Frontend Architecture

```
frontend/
  index.html          # layout: header, map div, sidebar panel, forecast chart, advisory box
  css/style.css        # Bootstrap overrides, risk color variables
  js/
    api.js              # fetch() wrappers for every backend endpoint
    map.js                # Leaflet init, GeoJSON choropleth, click handlers
    charts.js              # Chart.js forecast line chart + history chart
    advisory.js              # renders advisory text + risk breakdown bars
    alerts.js                 # "Send Alert" button -> POST + render mock payload
    app.js                     # wires everything together on page load
```
No bundler, no framework — `<script>` tags in order, loaded after Bootstrap/Leaflet/Chart.js CDN includes. This lets every team member edit their own `.js` file with minimal merge conflicts.

---

## 16. Database Schema

```sql
wards(
  id INTEGER PRIMARY KEY,
  name TEXT,
  lat REAL, lon REAL,
  boundary_geojson TEXT,        -- stored as JSON string
  population INTEGER,
  elderly_pct REAL,
  outdoor_worker_pct REAL,
  vulnerability_index REAL
);

weather_readings(
  id INTEGER PRIMARY KEY,
  ward_id INTEGER REFERENCES wards(id),
  timestamp DATETIME,
  temp_c REAL, humidity_pct REAL, wind_kmh REAL, solar_radiation REAL,
  source TEXT
);

risk_scores(
  id INTEGER PRIMARY KEY,
  ward_id INTEGER REFERENCES wards(id),
  timestamp DATETIME,
  heat_index REAL, wbgt REAL,
  base_score REAL, vulnerability_adjustment REAL, composite_score REAL,
  risk_category TEXT
);

forecasts(
  id INTEGER PRIMARY KEY,
  ward_id INTEGER REFERENCES wards(id),
  forecast_date DATE,
  predicted_heat_index REAL,
  predicted_risk_category TEXT,
  model_version TEXT
);

advisories(
  risk_category TEXT PRIMARY KEY,
  advisory_text TEXT
);

alerts_log(
  id INTEGER PRIMARY KEY,
  ward_id INTEGER REFERENCES wards(id),
  timestamp DATETIME,
  channel TEXT,          -- 'sms' | 'whatsapp'
  message TEXT,
  status TEXT             -- 'simulated' | 'sent'
);
```
SQLite is sufficient — no need for a separate DB server. If your team is more comfortable, this can even start as in-memory Python dicts and be swapped to SQLite once the demo works end-to-end (see §22).

---

## 17. API Endpoints & JSON Formats

**`GET /api/wards`** — list all wards with current risk (for map load)
```json
[
  {"id": 1, "name": "Ward 3 - Old City", "lat": 21.14, "lon": 79.08,
   "risk_category": "Orange", "composite_score": 71.4}
]
```

**`GET /api/risk-map`** — GeoJSON for choropleth
```json
{
  "type": "FeatureCollection",
  "features": [
    {"type": "Feature", "geometry": {...},
     "properties": {"ward_id": 1, "name": "Ward 3", "risk_category": "Orange", "color": "#FF8C00"}}
  ]
}
```

**`GET /api/wards/{id}`** — full ward detail
```json
{
  "ward": {"id": 1, "name": "Ward 3 - Old City", "elderly_pct": 12.4, "outdoor_worker_pct": 28.0},
  "current": {"temp_c": 41.2, "humidity_pct": 46, "wind_kmh": 9, "heat_index": 47.8, "wbgt": 34.1,
              "base_score": 68, "vulnerability_adjustment": 8.5, "composite_score": 76.5, "risk_category": "Red"},
  "forecast": [
    {"date": "2026-08-25", "predicted_heat_index": 46.1, "predicted_risk_category": "Red"},
    {"date": "2026-08-26", "predicted_heat_index": 44.0, "predicted_risk_category": "Orange"}
  ],
  "advisory": {"headline": "Extreme heat danger", "actions": ["Avoid outdoor work 11am-4pm", "Open nearby cooling centers", "..."]}
}
```

**`POST /api/alerts/simulate`** — body `{"ward_id": 1, "channel": "sms"}`, response:
```json
{"ward_id": 1, "channel": "sms", "status": "simulated",
 "message": "ALERT: Ward 3 - Old City is under RED heat risk today (HI 47.8C). Avoid outdoor work 11am-4pm. Cooling centers open at ...",
 "timestamp": "2026-08-24T10:15:00Z"}
```

**`GET /api/advisories/{risk_category}`** — static advisory text lookup.

---

## 18. Recommended Project Folder Structure

```
heatwave-early-warning/
├── README.md
├── backend/
│   ├── requirements.txt
│   ├── app/
│   │   ├── main.py
│   │   ├── db.py
│   │   ├── models.py
│   │   ├── schemas.py
│   │   ├── scheduler.py
│   │   ├── seed.py
│   │   └── services/
│   │       ├── weather_fetcher.py
│   │       ├── thermal_index.py
│   │       ├── risk_engine.py
│   │       ├── vulnerability.py
│   │       ├── advisory.py
│   │       ├── ml_model.py
│   │       └── alerts.py
│   └── data/
│       ├── wards.geojson
│       └── vulnerability.csv
├── ml/
│   ├── train_model.py
│   ├── historical_weather.csv   (generated)
│   └── model.pkl                (generated)
├── frontend/
│   ├── index.html
│   ├── css/style.css
│   └── js/{api.js, map.js, charts.js, advisory.js, alerts.js, app.js}
└── docs/
    ├── architecture.md
    ├── api_spec.md
    └── demo_script.md
```

---

## 19. Team Responsibilities (6 members)

| # | Role | Owns | Key files |
|---|---|---|---|
| P1 | Team Lead / Backend Core | FastAPI app, DB models, seed script, API integration, merges everyone's work | `main.py`, `db.py`, `models.py`, `schemas.py`, `seed.py` |
| P2 | Data Engineer | Weather ingestion + thermal index math | `weather_fetcher.py`, `thermal_index.py` |
| P3 | Risk & Vulnerability Engineer | Risk scoring, vulnerability CSV, advisory text | `risk_engine.py`, `vulnerability.py`, `advisory.py`, `data/vulnerability.csv` |
| P4 | ML Engineer | Historical data pull, model training, forecast serving | `ml/train_model.py`, `services/ml_model.py`, `model.pkl` |
| P5 | Frontend — Map | Leaflet map, GeoJSON choropleth, ward click interactions | `map.js`, `data/wards.geojson`, `style.css` (map part) |
| P6 | Frontend — Dashboard & Demo | Sidebar/detail panel, charts, advisory UI, alert button, overall polish, slides, demo rehearsal | `charts.js`, `advisory.js`, `alerts.js`, `app.js`, `index.html`, `docs/demo_script.md` |

P1 also owns integration risk — they should do a full end-to-end smoke test at the end of every day.

---

## 20 & 21. 4-Day Development Schedule with Daily Checkpoints
*(Day 0 = setup/planning same afternoon; Days 1–4 = build; Day 5 = buffer + polish + rehearsal, described after)*

### Day 0 (a few hours) — Setup
- Repo created, folder structure scaffolded, everyone's dev environment running (`python -m venv`, `pip install fastapi uvicorn sqlalchemy scikit-learn pandas requests apscheduler`).
- Pick the demo city; P5 finds/builds `wards.geojson` (8–15 wards); P3 drafts `vulnerability.csv` with placeholder numbers.
- **Checkpoint:** Everyone can run `uvicorn app.main:app --reload` and see `/docs` load; `wards.geojson` renders as plain markers on a blank Leaflet page.

### Day 1 — Vertical Slice (single ward, end-to-end)
- P1: DB models + seed script (loads wards + vulnerability into SQLite).
- P2: `weather_fetcher.py` pulls live data for ONE ward from Open-Meteo; `thermal_index.py` computes Heat Index.
- P3: `risk_engine.py` computes composite score for that one ward (hardcoded vulnerability ok for now).
- P1: `GET /api/wards/{id}` returns real computed data for that one ward.
- P5/P6: Frontend fetches that one endpoint and displays the numbers in a plain sidebar (map optional today).
- **Checkpoint (end of Day 1):** One ward, real weather → real Heat Index → real risk score → visible in browser. This is the most important milestone of the whole project — everything after is expansion, not new plumbing.

### Day 2 — Scale to All Wards + Map
- P2: Loop ingestion over all wards; scheduler running every 30–60 min (or manual "refresh" endpoint for demo safety).
- P3: Full vulnerability CSV wired in for all wards; advisory text templates for all 5 risk categories.
- P1: `GET /api/wards`, `GET /api/risk-map` finished and returning all wards.
- P5: Full Leaflet choropleth live, colored by risk_category, click-to-select working.
- P4: Historical data pull started (`archive-api.open-meteo.com`) for all ward centroids; first model training attempt.
- **Checkpoint (end of Day 2):** Full map, all wards colored correctly and clickable, showing real current data. This alone is already a presentable prototype.

### Day 3 — Forecast, ML, Alerts
- P4: `model.pkl` trained and `services/ml_model.py` serving `/forecasts`; `GET /api/wards/{id}` now includes forecast array.
- P6: Forecast chart (Chart.js) rendering in sidebar; advisory panel showing full text + action list.
- P3/P6: `/api/alerts/simulate` implemented and wired to a "Send Alert" button showing the JSON payload nicely formatted.
- P1: full integration pass — every endpoint tested via `/docs`.
- **Checkpoint (end of Day 3):** Every Must-Have feature exists and works end-to-end at least once, even if visually rough.

### Day 4 — Polish, Vulnerability Toggle, Robustness, Rehearsal
- All: UI polish (Bootstrap styling, legend, risk-breakdown bars, loading states).
- P3/P6: "heat-only vs vulnerability-adjusted" toggle (this is your key differentiator — do not skip it).
- P2/P1: error handling for API failures (cached last-known-good data if Open-Meteo is briefly down), seed script re-runnable for a clean demo reset.
- P6: finalize slides + `demo_script.md`; full team dry run of the 3–5 min demo, twice.
- **Checkpoint (end of Day 4):** Demo has been rehearsed twice on the actual machine that will be used, with no live internet dependency that could fail (have a cached fallback).

### Day 5 (buffer, if available) — Rehearsal & Contingency Only
- No new features. Fix bugs found in rehearsal, prepare offline fallback screenshots/video in case of no wifi at venue, print/save a 1-pager architecture diagram for judges.

---

## 22. Simplest Possible Version That Still Satisfies the Problem Statement

If time is critically short, this is the true floor — still legitimately addresses every bullet in the problem statement:

- **1 city, 8 wards represented as points** (not polygons) — Leaflet colored circle markers instead of choropleth polygons (saves needing real GeoJSON boundaries).
- **Heat Index only** (skip WBGT).
- **In-memory Python dict/JSON file instead of SQLite** — fetch weather on page load / button click, compute on the fly, no persistence needed for a live demo.
- **Forecast = Open-Meteo's own 7-day forecast directly**, with a single simple Linear Regression "risk escalation likelihood" classifier as your ML component (still genuinely ML, trained on the historical archive, just smaller scope).
- **Static vulnerability CSV** with clearly-labeled illustrative numbers.
- **Advisory = static text lookup table** keyed by risk category (no generation needed).
- **Alert = a JSON payload rendered on screen**, explicitly labeled "simulated — would integrate with SMS gateway (e.g. Twilio, government bulk-SMS API) in production."

This version is achievable by a beginner team in 2–3 focused days, leaving real buffer time.

---

## 23. Common Technical Risks & Fallbacks

| Risk | Fallback |
|---|---|
| Open-Meteo down/rate-limited during demo | Cache last successful fetch in DB/JSON; demo from cache if needed; mention this explicitly as a resilience feature, not a hidden hack |
| No real ward GeoJSON boundaries found in time | Use circular markers at ward centroids, or a synthetic 3x3/4x4 grid overlay labeled "Zone A1, A2..." — still satisfies "Zone/Ward level" requirement |
| ML model won't train in time / errors | Fall back to Open-Meteo's forecast + a trivial persistence/trend model; be honest about it, frame ML as "risk escalation classifier" which is smaller and safer |
| Team unfamiliar with Leaflet | Use the official Leaflet quickstart + GeoJSON tutorial almost verbatim — AI assistants generate this reliably; budget half a day, not more |
| Merge conflicts across 6 people | Strict file ownership per §19; P1 does all merges into `main`; everyone else works on feature branches |
| Venue wifi failure | Keep a fully working **offline** version: cached weather snapshot baked into the seed data, so the whole app runs with zero internet |
| Scope creep (adding polygons/ML/auth "for polish") | Follow the Must-Have list strictly until Day 3 checkpoint is hit; anything else is Day 4 only, and optional |

---

## 24. Demo Flow for Judges (3–5 minutes)

1. **(0:00–0:30) Problem framing** — "Standard heat warnings use only temperature. We compute what the heat actually does to the human body, per ward, factoring in who's most vulnerable."
2. **(0:30–1:00) Map overview** — show the live color-coded ward map; point out 2–3 wards in different risk colors.
3. **(1:00–1:45) Ward drill-down** — click the highest-risk ward; show current temp/humidity/wind → Heat Index/WBGT → risk score breakdown (base + vulnerability bars). Say the exact formula name (NWS Heat Index) out loud — it signals rigor.
4. **(1:45–2:30) Vulnerability toggle** — flip "heat-only" vs "vulnerability-adjusted" view live; show the same weather producing a materially different risk color in a high-elderly ward vs a low-vulnerability ward. This is your strongest visual moment.
5. **(2:30–3:15) Forecast + ML** — show the 3–5 day forecast chart; mention the ML model briefly and honestly (what it predicts, roughly how it was trained, that it's a risk-escalation forecast, not a mortality prediction).
6. **(3:15–3:45) Advisory + alert automation** — show the auto-generated advisory text, click "Send Alert," show the simulated SMS/WhatsApp payload; state clearly this is simulated and describe the real integration path (Twilio/govt SMS gateway/NDMA).
7. **(3:45–4:15) Close** — one slide: architecture diagram, tech stack, and 2–3 sentences on real-world deployment path (real ward boundaries, verified demographic data, live alert gateway, IMD integration).

---

## 25. Claims To Make vs. Claims To Avoid

**Safe to claim:**
- "We compute an explainable heat-risk score using the NWS Heat Index, a scientifically established metric, rather than raw temperature alone."
- "Our risk score is transparently adjustable by demographic vulnerability factors — you can see exactly how much elderly population and outdoor worker density change the outcome."
- "We use machine learning to forecast near-term heat-index/risk escalation trained on historical weather data for these locations."
- "The mortality risk index is a relative, illustrative prioritization category informed by published heat-mortality research, intended to help authorities prioritize response — not a clinical prediction."
- "The alert system is a working simulation demonstrating exactly how it would integrate with a real SMS/WhatsApp gateway."

**Do NOT claim:**
- Any specific number of predicted deaths, hospitalizations, or casualty counts.
- That the vulnerability/demographic data is official government data, if it is synthetic/illustrative — say so plainly.
- That your ML model has been validated against real historical mortality outcomes (unless you've genuinely done this with a real, cited dataset, which is unlikely in 5 days).
- That alerts are actually being delivered to real citizens/authorities, unless you've actually wired up and tested a real SMS provider.
- That your WBGT is the full ISO 7243 standard — say clearly it's a simplified estimation, since you lack a globe thermometer/mean radiant temperature input.

---

## IMPLEMENTATION TASK SEQUENCE

Ordered to get a **working end-to-end vertical slice as early as possible** (target: end of Day 1), then widen scope. Each task is scoped to be handed to an AI coding assistant directly.

---

**TASK 1**
- Person: P1 (Lead)
- Learn: FastAPI basics, project scaffolding, `uvicorn`
- Files: `backend/app/main.py`, `backend/requirements.txt`
- Inputs: none
- Outputs: A running FastAPI app with a `GET /health` endpoint returning `{"status": "ok"}`, visible at `/docs`
- Dependencies: none
- Done when: `uvicorn app.main:app --reload` runs, `/health` returns 200 in browser

**TASK 2**
- Person: P5
- Learn: GeoJSON basics, how to draw ward points/polygons for a chosen city
- Files: `backend/data/wards.geojson`
- Inputs: chosen city + 8–15 ward names/approximate centroids (can hand-pick from Google Maps coordinates)
- Outputs: valid GeoJSON FeatureCollection, one Feature per ward, `properties: {id, name}`
- Dependencies: none
- Done when: file validates on geojson.io and shows correct points/shapes over the chosen city

**TASK 3**
- Person: P3
- Learn: basic CSV design for vulnerability indicators
- Files: `backend/data/vulnerability.csv`
- Inputs: ward list from Task 2, any obtainable/estimated elderly % and outdoor worker % per ward
- Outputs: CSV with columns `ward_id, ward_name, population, elderly_pct, outdoor_worker_pct`
- Dependencies: Task 2 (needs ward IDs/names to align)
- Done when: every ward in wards.geojson has a matching row

**TASK 4**
- Person: P1
- Learn: SQLAlchemy models + SQLite
- Files: `backend/app/db.py`, `backend/app/models.py`
- Inputs: schema from §16
- Outputs: SQLite tables created on app startup
- Dependencies: Task 1
- Done when: `db.sqlite3` file is generated with all tables (verify with any SQLite browser or `sqlite3` CLI `.tables`)

**TASK 5**
- Person: P1
- Learn: loading GeoJSON/CSV into SQL rows
- Files: `backend/app/seed.py`
- Inputs: `wards.geojson` (Task 2), `vulnerability.csv` (Task 3)
- Outputs: `wards` table populated with all wards + vulnerability fields + computed `vulnerability_index`
- Dependencies: Tasks 2, 3, 4
- Done when: querying the `wards` table returns all wards with non-null vulnerability_index

**TASK 6**
- Person: P2
- Learn: Open-Meteo API request format (`current`, `daily` params), `requests` library
- Files: `backend/app/services/weather_fetcher.py`
- Inputs: a single ward's lat/lon (hardcode for first pass)
- Outputs: a Python function `fetch_weather(lat, lon) -> dict` returning normalized `{temp_c, humidity_pct, wind_kmh, solar_radiation}` plus a 5-day forecast list
- Dependencies: none (can be built in parallel with Tasks 1–5)
- Done when: calling the function in a script prints real current weather for that lat/lon

**TASK 7**
- Person: P2
- Learn: the Heat Index (Rothfusz) formula and simplified WBGT formula from §8
- Files: `backend/app/services/thermal_index.py`
- Inputs: temp_c, humidity_pct (and optionally solar_radiation, wind_kmh)
- Outputs: functions `heat_index(temp_c, rh) -> float` and `wbgt(temp_c, rh) -> float`
- Dependencies: none
- Done when: unit-testable against a couple of known reference values (e.g., 35°C/70% RH should read noticeably higher HI than 35°C/20% RH)

**TASK 8 — CRITICAL VERTICAL SLICE MILESTONE**
- Person: P1 (integrates P2 + P3's work)
- Learn: wiring service functions into a FastAPI route
- Files: `backend/app/services/risk_engine.py`, `backend/app/main.py` (new route)
- Inputs: output of Task 6 (weather) + Task 7 (thermal index) + one ward's `vulnerability_index` from DB
- Outputs: `GET /api/wards/{id}` returning full JSON per §17 for ONE real ward, live-computed
- Dependencies: Tasks 5, 6, 7
- Done when: hitting `/api/wards/1` in the browser/`/docs` returns real current weather + real Heat Index + real composite risk score for a real place — **this is the Day 1 checkpoint**

**TASK 9**
- Person: P5
- Learn: Leaflet.js quickstart, adding a tile layer + a single marker
- Files: `frontend/index.html`, `frontend/js/map.js`
- Inputs: none (static test data first, then Task 2's GeoJSON)
- Outputs: a page showing an OSM map centered on the chosen city with markers for all wards (uncolored is fine at this stage)
- Dependencies: Task 2
- Done when: map loads in browser with all ward markers visible and clickable (click just logs ward id to console for now)

**TASK 10**
- Person: P6
- Learn: `fetch()` API basics, rendering JSON into HTML
- Files: `frontend/js/api.js`, `frontend/js/app.js`
- Inputs: Task 8's live endpoint
- Outputs: clicking a ward marker calls `/api/wards/{id}` and displays raw current weather + risk score in a sidebar `<div>`
- Dependencies: Tasks 8, 9
- Done when: clicking any (mocked, if only one ward is wired) marker shows real numbers on screen — **full vertical slice proven end-to-end in the browser**

*(Everything below widens the scope now that the pipe works.)*

**TASK 11**
- Person: P2
- Files: `backend/app/services/weather_fetcher.py` (extend), `backend/app/scheduler.py`
- Inputs: all wards from DB
- Outputs: loop that fetches+stores weather for every ward; APScheduler job running every 30–60 min, plus a manual `POST /api/refresh` for demo control
- Dependencies: Task 8
- Done when: `weather_readings` table has rows for every ward after one run

**TASK 12**
- Person: P3
- Files: `backend/app/services/advisory.py`, `backend/app/models.py` (advisories table seed)
- Inputs: 5 risk categories
- Outputs: advisory text + action list for each category, exposed via risk_engine output
- Dependencies: Task 8
- Done when: `/api/wards/{id}` response includes a populated `advisory` object

**TASK 13**
- Person: P1
- Files: `backend/app/main.py` (new routes)
- Inputs: existing risk_engine output for all wards
- Outputs: `GET /api/wards` and `GET /api/risk-map` per §17
- Dependencies: Task 11
- Done when: both endpoints return correct JSON for all wards

**TASK 14**
- Person: P5
- Files: `frontend/js/map.js` (extend)
- Inputs: Task 13's `/api/risk-map`
- Outputs: full choropleth coloring by risk_category, legend on map
- Dependencies: Task 13
- Done when: all wards render in correct risk colors and update on refresh

**TASK 15**
- Person: P4
- Learn: Open-Meteo historical archive API, pandas basics
- Files: `ml/train_model.py`
- Inputs: all ward centroids
- Outputs: `ml/historical_weather.csv` with 1–2 years of daily weather per ward
- Dependencies: Task 2 (needs ward coordinates)
- Done when: CSV has clean rows for every ward with no major gaps

**TASK 16**
- Person: P4
- Learn: scikit-learn `RandomForestRegressor`/`LinearRegression`, feature engineering with lag features, `joblib`
- Files: `ml/train_model.py` (extend), produces `ml/model.pkl`
- Inputs: `historical_weather.csv`
- Outputs: trained model + a printed MAE metric on a held-out test split
- Dependencies: Task 15
- Done when: `model.pkl` exists and loading + predicting on a sample row works without error

**TASK 17**
- Person: P4
- Files: `backend/app/services/ml_model.py`
- Inputs: `model.pkl`, recent `weather_readings` for a ward
- Outputs: function `predict_forecast(ward_id) -> list[{date, predicted_heat_index, predicted_risk_category}]`, writes to `forecasts` table
- Dependencies: Task 16, Task 11
- Done when: calling this for a ward returns 3–5 sensible forward dates

**TASK 18**
- Person: P1
- Files: `backend/app/main.py` (extend `/api/wards/{id}`)
- Inputs: Task 17's forecast function
- Outputs: `/api/wards/{id}` now includes populated `forecast` array
- Dependencies: Task 17
- Done when: endpoint response matches the §17 example shape

**TASK 19**
- Person: P6
- Learn: Chart.js line chart basics
- Files: `frontend/js/charts.js`
- Inputs: Task 18's forecast array
- Outputs: forecast line chart rendered in sidebar on ward click
- Dependencies: Task 18
- Done when: chart shows real predicted values per ward

**TASK 20**
- Person: P6
- Files: `frontend/js/advisory.js`, `frontend/css/style.css`
- Inputs: Task 12's advisory object + Task 8's score breakdown (base/vulnerability/composite)
- Outputs: advisory text panel + a small 3-bar breakdown chart of base score vs vulnerability adjustment vs final score
- Dependencies: Task 12
- Done when: breakdown bars visually match the numbers returned by the API

**TASK 21**
- Person: P3
- Files: `backend/app/services/alerts.py`, `backend/app/main.py` (new route)
- Inputs: ward's advisory + current risk
- Outputs: `POST /api/alerts/simulate` per §17, logs to `alerts_log`
- Dependencies: Task 12
- Done when: endpoint returns a realistic message string given a ward_id

**TASK 22**
- Person: P6
- Files: `frontend/js/alerts.js`
- Inputs: Task 21's endpoint
- Outputs: "Send Alert" button per ward showing the returned payload nicely formatted (styled like an SMS bubble)
- Dependencies: Task 21
- Done when: clicking the button on any ward shows a realistic simulated message on screen

**TASK 23**
- Person: P3 + P6 (pair)
- Files: `backend/app/services/risk_engine.py` (add a `heat_only_score` alongside `composite_score`), `frontend/js/app.js` (toggle control)
- Inputs: existing risk_engine
- Outputs: UI toggle switching map/detail view between heat-only and vulnerability-adjusted scores
- Dependencies: Tasks 14, 20
- Done when: toggling visibly changes ward colors/scores for at least one high-vulnerability ward

**TASK 24**
- Person: P1
- Files: all `services/*.py` (add try/except + cached fallback), `seed.py` (re-runnable reset)
- Inputs: none
- Outputs: graceful handling if Open-Meteo is unreachable (serve last cached `weather_readings` row instead of erroring)
- Dependencies: Task 11
- Done when: temporarily disabling internet still lets the demo load using cached data

**TASK 25**
- Person: All (P1 coordinates)
- Files: `docs/demo_script.md`, presentation slides
- Inputs: finished app
- Outputs: rehearsed 3–5 min demo script matching §24, a backup screen-recording of the working demo
- Dependencies: Tasks 1–24
- Done when: full run-through completes within time, twice, on the actual demo machine
