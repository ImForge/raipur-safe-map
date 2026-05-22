# Raipur Safe Map — AI Coding Assistant Project Prompt

> **How to use this file:** Paste this into your AI coding assistant as project context.
> - **VS Code + Copilot:** save as `.github/copilot-instructions.md` in the repo root.
> - **Cursor:** save as `.cursorrules` in the repo root.
> - **Claude / ChatGPT:** paste at the start of a conversation before asking for code.
>
> It tells the AI what the project is, how it is built, the conventions to follow,
> what already exists, and how to extend each part correctly.

---

## 1. Project identity

You are helping build **Raipur Safe Map**, a women's safety crime-mapping web application
for the city of **Raipur, Chhattisgarh, India**.

The app collects crime/incident data from three sources — official statistics, local news,
and anonymous public reports — and renders it as an interactive risk **heatmap** with a
**day vs night toggle**, because the safety of a place depends on the time of day.

**Mission constraints that must never be violated:**
- This is a real safety tool. Correctness and honesty matter more than cleverness.
- Never imply an area is "safe" — only "fewer reported incidents". Under-reporting is real.
- Never expose or store personally identifying user data. Reports are anonymous.
- Never weaken the abuse-prevention logic (rate limiting, corroboration, trust weighting).

---

## 2. Tech stack (do not substitute without being asked)

| Layer | Technology | Notes |
|-------|-----------|-------|
| Database | PostgreSQL + **PostGIS** extension | All location data is PostGIS `geometry(Point, 4326)`. |
| ORM | SQLAlchemy 2.x + GeoAlchemy2 | GeoAlchemy2 is what makes SQLAlchemy understand PostGIS. |
| Backend | **FastAPI** (Python 3.11+) | Async. Pydantic for all request/response schemas. |
| Server | Uvicorn | `uvicorn main:app --reload` in dev. |
| Scraping | Requests + BeautifulSoup + lxml | For the news ingestion pipeline. |
| Frontend | React 18 + Vite + Leaflet + leaflet.heat | **Under redesign — do not modify unless explicitly asked.** |
| Dev infra | Docker Compose | Runs the Postgres/PostGIS container. |

Python dependencies are pinned in `backend/requirements.txt`. Never bump versions unprompted.

---

## 3. Architecture (three tiers)

```
INGESTION                STORAGE                 SERVING
news scraper  ──insert──▶ PostgreSQL ◀──read──── FastAPI /api/*
seed JSON     ──insert──▶ + PostGIS              ──JSON──▶ React + Leaflet
user reports  ──POST───▶ (3 tables)
```

Data flows in two directions:
- **Inbound:** scraper / seed / user reports → cleaned → geocoded → stored.
- **Outbound:** stored incidents → risk model scores them → JSON → map.

---

## 4. Directory structure

```
raipur-safe-map/
├── demo.html               # standalone demo, no backend — do not break it
├── docker-compose.yml      # Postgres+PostGIS container
├── db/init.sql             # enables PostGIS extension
├── backend/
│   ├── main.py             # FastAPI app + CORS + router registration
│   ├── database.py         # SQLAlchemy engine/session, get_db dependency
│   ├── models.py           # ORM models: Incident, UserReport, PoliceStation
│   ├── schemas.py          # Pydantic I/O schemas
│   ├── requirements.txt
│   ├── routers/
│   │   ├── incidents.py    # GET  /api/incidents
│   │   ├── reports.py      # POST /api/reports
│   │   └── risk.py         # GET  /api/risk-grid, GET /api/hotspots
│   ├── services/
│   │   └── kde.py          # THE RISK MODEL: weighting, KDE, DBSCAN
│   ├── scripts/
│   │   ├── seed_db.py      # loads data/seed_incidents.json into DB
│   │   └── scraper.py      # news scraper template
│   └── data/
│       └── seed_incidents.json
└── frontend/               # React — out of scope unless asked
```

---

## 5. Domain model — memorize these facts

### 5.1 Crime types and severity (1–10 scale)

Severity is **set server-side from the crime type — never trust a client-supplied severity**.

```
suspicious        1      assault            6
harassment        2      domestic_violence  7
theft             2      suspicious_death   8
chain_snatching   3      sexual_assault    10
stalking          4
```

The canonical map lives in `backend/routers/reports.py` as `TYPE_TO_SEVERITY` and in
`backend/scripts/scraper.py` as `DEFAULT_SEVERITY`. **Keep these two in sync.**

### 5.2 The risk model (in `services/kde.py`)

The model is a **statistical model, not machine learning**. There is no neural net, no
training loop. It is a Kernel Density Estimation with these parameters:

```
weight = severity × time_decay × source_trust

time_decay  = exp(-λ × days_old),  λ = 0.00385   → ~180-day half-life
KDE kernel  = Gaussian, bandwidth h = 0.002°     → ~200 m
grid        = 50 × 50 cells over the Raipur bbox
DBSCAN      = eps 400 m, min_samples 3           → hotspot clustering
source_trust= 1.0 for news/ncrb/manual, 0.5 for unverified user reports
```

If asked to "train the model", that means **calibrate these parameters** — there is no
gradient descent. A genuine ML version would be a separate, future supervised model
(gradient-boosted trees over (grid-cell, time-slot) examples). Do not silently swap the
KDE model for an ML model; propose it explicitly first.

### 5.3 Time of day

`night` = incident hour `< 6` OR `>= 19`. `day` = everything else.
Every risk computation is done **separately** for day and night.

### 5.4 Geography

- Raipur city center: `lat 21.2514, lng 81.6296`. Bounding box roughly
  `lat 21.20–21.30, lng 81.59–81.74`.
- All coordinates are **WGS 84 / SRID 4326** (standard GPS lat-lng).
- Known localities (used by the scraper gazetteer): Telibandha, Civil Lines, Kotwali,
  Devendra Nagar, Gol Bazar, Tikrapara, Pandri, Mowa, Khamhardih, Gudhiyari,
  Mandir Hasaud, Ganj.

---

## 6. Database schema

Three tables, kept separate on purpose (different trust levels).

**`incidents`** — trusted on arrival (news/official). Feeds the public map immediately.
`id, type, severity, area, description, source, source_url, occurred_at, location(Point 4326), created_at`

**`user_reports`** — needs corroboration before it counts.
`id, type, severity, time_of_day, occurred_at, location(Point 4326), is_verified,
corroboration_count, anonymous_id, created_at`

**`police_stations`** — static reference data.
`id, name, phone, location(Point 4326)`

**PostGIS rules:**
- Insert points with `func.ST_SetSRID(func.ST_MakePoint(lng, lat), 4326)` — **lng first**.
- Read coords back with `func.ST_X(location)` (lng) and `func.ST_Y(location)` (lat).
- Proximity queries use `ST_DWithin(a::geography, b::geography, metres)`.
- Bounding-box filter uses `location && ST_MakeEnvelope(minLng, minLat, maxLng, maxLat, 4326)`.
- Every spatial table needs a GIST index on `location`.

---

## 7. API contract (do not break these shapes)

| Endpoint | Returns |
|----------|---------|
| `GET /api/incidents?time_of_day=&min_lat=&min_lng=&max_lat=&max_lng=` | list of incidents w/ `lat`,`lng` |
| `POST /api/reports` | the stored report incl. `is_verified` |
| `GET /api/risk-grid?time_of_day=&resolution=&bbox` | `{time_of_day, cells:[{lat,lng,weight}]}` |
| `GET /api/hotspots?time_of_day=&limit=` | list of `{area,lat,lng,score,incident_count}` |
| `GET /health` | `{status:"ok"}` |

`POST /api/reports` body: `{type, lat, lng, time_of_day, description?, anonymous_id?}`.
The client must NOT send `severity` — the server sets it.

---

## 8. Abuse prevention — never weaken this

User reports pass three independent gates (all in `routers/reports.py`):

1. **Rate limit** — max 5 reports/hour per `anonymous_id`. In-memory now; should become
   Redis for multi-server deployments.
2. **Corroboration** — a new report is `is_verified=False` and **excluded from the public
   map** until ≥2 same-type reports exist within 200 m and 30 days.
3. **Trust weighting** — even verified user reports carry `source_trust=0.5`, half of
   news-confirmed incidents.

If you add a new way to submit data, it must pass equivalent gates.

---

## 9. Code conventions

**Python / backend:**
- Type-hint every function signature. Pydantic schemas for all API I/O — no raw dicts.
- Routers stay thin; heavy logic goes in `services/`. The risk math lives ONLY in `kde.py`.
- Database access only through the `get_db` dependency. Never open ad-hoc connections.
- Read config from environment variables (`DATABASE_URL` etc.). Never hardcode secrets.
- Docstrings explain *why*, not *what*. Keep the existing teaching-comment style — this
  is a learning project; comments should help a CS student understand the reasoning.
- snake_case for functions/variables, PascalCase for classes, UPPER_CASE for constants.

**General:**
- Small, focused commits/changes. One concern at a time.
- When you change a parameter of the risk model, state the new value's reasoning.
- Prefer standard library and already-listed dependencies. Adding a dependency requires
  an explicit note explaining why.
- Never edit `demo.html`'s data model without keeping it consistent with the backend.

---

## 10. Current status

**Built and working:**
- ✅ `demo.html` — standalone interactive demo with seed data
- ✅ Backend: FastAPI app, all 4 API endpoints, 3 ORM models, Pydantic schemas
- ✅ Risk model: weighting + time decay + Gaussian KDE + DBSCAN clustering
- ✅ User reports with rate limiting + corroboration + trust weighting
- ✅ Seed data (~60 incidents, 12 police stations) + seed loader script
- ✅ News scraper **template** (selectors not yet configured for real sites)
- ✅ Docker Compose for the database

**Not done / next up:**
- ⬜ Configure scraper CSS selectors for real Raipur news sites (start with one site)
- ⬜ Real geocoder integration (Nominatim, then optionally Google) to replace the
  fixed gazetteer
- ⬜ Phase 5: **safe routing** — lowest-risk path, not shortest. Highest-value feature.
- ⬜ Street-lighting data layer
- ⬜ Proximity alerts; trusted-contact location sharing; mobile app

---

## 11. How to extend each part (guidance for common tasks)

**Add a new crime type:** update `TYPE_TO_SEVERITY` (reports.py) AND `DEFAULT_SEVERITY` +
`CRIME_TYPE_KEYWORDS` (scraper.py). Pick a severity consistent with the existing scale.

**Tune the risk model:** edit only the constants at the top of `services/kde.py`
(`TIME_DECAY_LAMBDA`, `KDE_BANDWIDTH_DEG`, DBSCAN `eps`/`min_samples`). Explain the
reasoning for any new value. Do not scatter these numbers across files.

**Configure the scraper for a real site:** open the site, inspect its HTML, fill in the
`link_selector`, `article_title_selector`, `article_body_selector` for that source in the
`SOURCES` list. Respect robots.txt; keep the 1-second throttle.

**Build safe routing (Phase 5):** integrate OSRM or GraphHopper. The routing engine sums
road-segment lengths to find the shortest path; instead, weight each segment's cost by the
risk score along it so the engine returns the safest reasonable route. The risk surface
from `kde.py` is the input. Expose it as `POST /api/safe-route`. A suggested route is a
suggestion, never a safety guarantee — reflect that in any response/UI text.

**Add an ML model (only if explicitly asked):** keep it separate from KDE. Frame it as
supervised prediction over (grid-cell, time-slot) examples, label = incident occurred/count,
features = nearby recent counts, distance to police station, lighting, population density,
POIs, day-of-week, holiday. Gradient-boosted trees (XGBoost/LightGBM) as the first model;
validate on a later, unseen time period. Do not replace KDE without discussion.

---

## 12. When unsure

- If a request conflicts with the mission constraints in §1, flag it instead of complying.
- If a change would weaken abuse prevention (§8) or privacy, flag it.
- If asked for a feature that implies an ML "trained model", clarify whether they mean
  calibrating the KDE parameters or building a genuine supervised model — they are
  different things (see §5.2).
- Prefer asking one sharp clarifying question over guessing on architecture decisions.
