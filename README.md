# Raipur Safe Map

A women's safety map for Raipur, Chhattisgarh. Visualizes crime hotspots from official records, news reports, and anonymous user submissions. Shows day vs night patterns separately so people can make informed decisions about where to go and when.

---

## What's in the box

```
raipur-safe-map/
├── demo.html               # ★ Open this in any browser — works immediately
├── backend/                # FastAPI + PostGIS API server
│   ├── main.py             # App entry
│   ├── database.py         # SQLAlchemy / PostGIS connection
│   ├── models.py           # Incident, UserReport, PoliceStation ORM models
│   ├── schemas.py          # Pydantic I/O schemas
│   ├── routers/
│   │   ├── incidents.py    # GET /api/incidents (with spatial + time filter)
│   │   ├── reports.py      # POST /api/reports (with corroboration logic)
│   │   └── risk.py         # GET /api/risk-grid, /api/hotspots
│   ├── services/
│   │   └── kde.py          # KDE + DBSCAN risk math
│   ├── scripts/
│   │   ├── seed_db.py      # Loads seed_incidents.json into Postgres
│   │   └── scraper.py      # News scraper template for ongoing ingestion
│   ├── data/
│   │   └── seed_incidents.json  # ~60 seed incidents across 12 Raipur areas
│   └── requirements.txt
├── frontend/               # React + Vite + Leaflet
│   ├── index.html
│   ├── package.json
│   ├── vite.config.js
│   └── src/
│       ├── main.jsx
│       ├── App.jsx
│       ├── api.js
│       ├── styles.css
│       └── components/
│           ├── Map.jsx
│           ├── Sidebar.jsx
│           └── ReportModal.jsx
├── db/
│   └── init.sql            # Enables PostGIS extension
└── docker-compose.yml      # One-command Postgres+PostGIS setup
```

---

## See it work right now (zero setup)

Just open `demo.html` in any browser. It's a self-contained file with the seed data baked in — Leaflet map of Raipur, real police stations, ~50 seed incidents from real Raipur news, day/night toggle, click-to-report. No backend needed.

This is your **playground for the concept**. Once you understand the model, move to the full stack below.

---

## Run the full stack (with database)

You'll need:

- Docker + Docker Compose (for the database)
- Python 3.11+
- Node.js 18+

### 1. Start the database

```bash
docker compose up -d db
```

This launches PostgreSQL with PostGIS extension on `localhost:5432`. Database `raipur_safemap`, user `postgres`, password `postgres`.

### 2. Set up the backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate          # on Windows: .venv\Scripts\activate
pip install -r requirements.txt

# Seed the database with the JSON fixture (~60 real Raipur incidents)
python -m scripts.seed_db

# Start the API
uvicorn main:app --reload
```

Backend is now live at `http://localhost:8000`. Auto-generated API docs at `http://localhost:8000/docs`.

### 3. Set up the frontend

In a new terminal:

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:5173`.

---

## Architecture in one diagram

```
┌────────────────┐   ┌────────────────┐   ┌──────────────────┐
│  News scraper  │──▶│                │   │                  │
│  (nightly cron)│   │   PostgreSQL   │◀──│   FastAPI        │
└────────────────┘   │   + PostGIS    │   │   /api/*         │
                     │                │   │                  │
┌────────────────┐   │  incidents     │   │  - KDE risk grid │
│  Seed data JSON│──▶│  user_reports  │   │  - DBSCAN cluster│
│  (manual)      │   │  police_stations│  │  - Corroboration │
└────────────────┘   └────────────────┘   └────────┬─────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │   React frontend │
                                          │   Leaflet map +  │
                                          │   heatmap +      │
                                          │   day/night UI   │
                                          └──────────────────┘
```

---

## The risk math (read `backend/services/kde.py`)

Every incident gets a weight:

```
weight = severity × time_decay × source_trust
```

- **severity**: 2 (theft) up to 10 (sexual assault). Hard-coded server-side so users can't inflate it.
- **time_decay**: `exp(-λ × days_old)` with λ tuned for a 180-day half-life.
- **source_trust**: 1.0 for news/NCRB, 0.5 for unverified user reports.

These weighted points get spread across the map using **Kernel Density Estimation** with a Gaussian kernel (bandwidth ~200m). The result is the smooth heatmap you see — not just disconnected dots.

Hotspots are clustered using a simple **DBSCAN** (eps = 400m, min_samples = 3). For production scale you'd replace this with `sklearn.cluster.DBSCAN` using the haversine metric.

---

## User reports & abuse prevention

Anonymous reports go through three gates before they affect the public map:

1. **Rate limiting** — max 5 reports per anonymous_id per hour
2. **Corroboration** — a report stays unverified (and excluded from the heatmap) until at least 2 reports of the same type land within 200m and 30 days
3. **Source-trust weighting** — even verified user reports count for half what news-sourced incidents do

This is what stops a single bad actor from defacing your map.

---

## Data sources

- **NCRB** (annual): https://data.gov.in (city-aggregate stats, used for calibration)
- **News**: Patrika, Dainik Bhaskar, Times of India, Hindustan Times — Raipur editions. Scraper template in `backend/scripts/scraper.py`.
- **Crowdsourced**: in-app anonymous reports

---

## Phased roadmap

- [x] **Phase 1 — MVP** (demo.html): static map + day/night + reporting
- [x] **Phase 2 — Real backend**: PostGIS, scraper, ingestion pipeline
- [x] **Phase 3 — Risk surface**: KDE heatmap with time decay
- [x] **Phase 4 — User reports**: anonymous reporting with corroboration
- [ ] **Phase 5 — Safe routing**: shortest *and* lowest-risk path between two points (use OSRM/GraphHopper, modify edge weights with risk score)
- [ ] **Phase 6 — Mobile**: port frontend to React Native
- [ ] **Phase 7 — Alerts**: notify when entering a high-risk area
- [ ] **Phase 8 — Trusted contacts**: live location sharing with N people on demand

---

## Ethical commitments

This app makes deliberate choices to avoid harm:

- **Wording matters**: areas show "fewer reported incidents" — never labeled "safe." Underreporting is real.
- **No street-level pinpointing**: the heatmap blurs incidents into 200m+ blobs. We don't single out homes or businesses.
- **No identifying data**: reports are anonymous. No phone numbers, no names, no precise location history.
- **Bias acknowledgment**: the underlying data is biased toward reported crimes. Neighborhoods where women don't feel safe going to police may appear deceptively "safe." The footer in the UI reminds users of this.

---

## Where to extend this

Once you have the basics running, here's what'd add real value, in order of difficulty:

1. Wire up a real geocoder (Nominatim's free API → swap to Google's when accuracy gets critical) so the scraper can parse "near Telibandha Talab" into actual coordinates.
2. Add the streetlight overlay. Talk to the Raipur Municipal Corporation about lighting data, or crowd-source it (volunteers walk routes after dark and mark dark stretches).
3. Build the safe-route feature. This is the **single most useful feature** the app could have: "I'm at Pandri, I need to get to Civil Lines, give me the safest route, not the shortest." Use OSRM or GraphHopper with a custom edge weighting that penalizes risk.
4. Notification when entering a high-risk zone (geofencing). Mobile-only.
5. Trusted-contact live location sharing — "I'm walking home, watch me until I arrive."

---

## License

MIT. Build on it, fork it, ship it. If you launch something based on this in Raipur or elsewhere, send me a note — I want to see it.
