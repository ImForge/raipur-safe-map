# Raipur Safe Map — Complete Setup & Deployment Guide

This guide takes you from zero to a live, deployed app. Read it top to bottom the
first time so you understand the full picture, then follow each section step by step.

---

## Before anything else — understand what you have

Your project has three separate pieces that each need their own setup:

| Piece | What it is | How you run it |
|-------|-----------|---------------|
| `dashboard.html` | The premium UI — fully self-contained | Just double-click it. No setup at all. |
| `backend/` | The Python API server (FastAPI) | Needs Python + a database |
| `frontend/` | The React app (connects to the backend) | Needs Node.js |

**The most important thing to know:** `dashboard.html` already works right now with
no installation. If you want to show the project to someone today, just open that file.
The backend + frontend setup is for when you want real live data, a news scraper, and
user reports flowing into the map.

---

## Part 1 — Install the prerequisites

You only do this once. These are the tools the project depends on.

### 1.1 Node.js (needed for the React frontend)

Node.js is a JavaScript runtime. npm (Node Package Manager) comes with it and is what
you use to install frontend libraries.

- Go to **https://nodejs.org** and download the LTS version (the one labelled "Recommended for most users").
- Run the installer, keep all defaults.
- When it finishes, open a new terminal and verify:
  ```
  node --version
  npm --version
  ```
  Both commands should print a version number, e.g. `v22.0.0` and `10.0.0`.

### 1.2 Python 3.11+ (needed for the FastAPI backend)

Python is the language the backend is written in.

- Go to **https://www.python.org/downloads** and download Python 3.11 or 3.12.
- **Windows only:** on the first screen of the installer, check the box that says
  **"Add Python to PATH"** before clicking Install. This is the most common mistake.
- Verify:
  ```
  python --version
  ```
  Should print `Python 3.11.x` or `3.12.x`.

### 1.3 Docker Desktop (needed for the database)

Docker lets you run a PostgreSQL+PostGIS database in a container — meaning you don't
have to install a full database server on your machine. It's a single app.

- Go to **https://www.docker.com/products/docker-desktop** and download Docker Desktop
  for your OS.
- Install it and start it. You'll see a whale icon in your taskbar/menu bar when it's
  running.
- Verify in a terminal:
  ```
  docker --version
  ```

### 1.4 Git (needed for GitHub)

Git is the version-control tool that GitHub is built on.

- **Windows:** download from **https://git-scm.com/download/win**. During install,
  when it asks about the default editor, choose whatever you're comfortable with.
  Everything else can stay as the default.
- **Mac:** Git is already installed if you have Xcode tools. Run `git --version` — if
  it's not there, it will prompt you to install it.
- Verify:
  ```
  git --version
  ```

### 1.5 VS Code (recommended code editor)

If you don't have it: **https://code.visualstudio.com**. Download and install.

---

## Part 2 — Get the project running locally

Open VS Code. Open the project folder (`raipur-safe-map`) with
File → Open Folder.

Then open the built-in terminal: Terminal → New Terminal. Everything below is typed
into this terminal.

### 2.1 Start the database

The database runs inside Docker. Make sure Docker Desktop is open and its whale icon
shows it's running. Then:

```bash
docker compose up -d db
```

What this does: downloads a PostgreSQL image with PostGIS (the spatial extension),
creates a database called `raipur_safemap`, and starts it in the background.
The `-d` flag means "detached" — it runs silently, you don't see any logs.

**Expected output:** Docker will print something like:
```
✔ Container raipur_safemap_db  Started
```

To check it's running:
```bash
docker ps
```
You should see a row with `postgis/postgis` in the image column.

> The database keeps running until you stop Docker or run `docker compose down`.
> Next time you work on the project, just open Docker Desktop and run
> `docker compose up -d db` again.

### 2.2 Set up the Python backend

Python projects use a "virtual environment" — a sandboxed folder that holds the
project's specific library versions without mixing them with other Python projects.

```bash
cd backend
python -m venv .venv
```

Now activate it:

```bash
# On Windows (Command Prompt or PowerShell):
.venv\Scripts\activate

# On Mac / Linux:
source .venv/bin/activate
```

Your terminal prompt will change to show `(.venv)` at the start, confirming it's
active. Now install the backend's libraries:

```bash
pip install -r requirements.txt
```

This downloads about 15 packages including FastAPI, SQLAlchemy, PostGIS support, and
the scraper tools. It takes 1-3 minutes.

Now create the database tables and load the seed data:

```bash
python -m scripts.seed_db
```

**Expected output:**
```
✓ Seeded 47 incidents and 11 police stations.
```

If you see an error saying it can't connect to the database, the Docker container
isn't running yet — go back to step 2.1.

Now start the API server:

```bash
uvicorn main:app --reload
```

**Expected output:**
```
INFO:     Uvicorn running on http://127.0.0.1:8000
INFO:     Application startup complete.
```

Open **http://localhost:8000/docs** in your browser. You'll see an interactive API
explorer listing all the endpoints. This confirms the backend is working.

> Keep this terminal open while you work on the project. The `--reload` flag means
> it automatically restarts when you edit a Python file.

### 2.3 Set up the React frontend

Open a **second terminal** in VS Code (click the `+` button in the terminal panel).
Make sure you're back in the project root folder (not `backend/`):

```bash
cd frontend
npm install
```

This reads `package.json` and downloads all the frontend libraries (React, Leaflet,
Vite, etc.) into a `node_modules` folder. Takes 1-2 minutes the first time.

Start the development server:

```bash
npm run dev
```

**Expected output:**
```
  VITE v5.4.6  ready in 300ms
  ➜  Local:   http://localhost:5173/
```

Open **http://localhost:5173** in your browser. You should see the React app, which
talks to your FastAPI backend through the Vite proxy on `/api`.

### 2.4 Quick reference: running it after the first setup

Next time you want to work on the project:

```bash
# Terminal 1 — start the database
docker compose up -d db

# Terminal 2 — start the backend
cd backend
.venv\Scripts\activate   (Windows)  or  source .venv/bin/activate (Mac/Linux)
uvicorn main:app --reload

# Terminal 3 — start the frontend
cd frontend
npm run dev
```

Then open http://localhost:5173.

---

## Part 3 — Push to GitHub

GitHub stores your code online. It's your backup, your portfolio, and your deployment
source (Vercel and Render both pull from GitHub).

### 3.1 Create the GitHub repository

1. Go to **https://github.com** and sign in (create an account if you don't have one).
2. Click the **+** in the top right → **New repository**.
3. Name it: `raipur-safe-map`
4. Leave it **Public** (so Vercel and Render can see it for free).
5. **Do NOT** check "Add a README" or "Add .gitignore" — you already have those.
6. Click **Create repository**.

GitHub will show you a page with commands. You'll use those in a moment.

### 3.2 Create a .gitignore file

Before pushing, you need to tell Git which files to ignore. Create a file called
`.gitignore` in the root of your project with this content:

```
# Python
backend/.venv/
backend/__pycache__/
backend/**/__pycache__/
*.pyc
*.pyo

# Node
frontend/node_modules/
frontend/dist/

# Environment files (contains secrets — never commit these)
.env
backend/.env
frontend/.env

# Database / Docker volumes
postgres-data/

# OS files
.DS_Store
Thumbs.db
```

### 3.3 Initialize and push

In the VS Code terminal, make sure you're in the **project root** (the
`raipur-safe-map` folder, not inside `backend` or `frontend`):

```bash
git init
git add .
git commit -m "Initial commit: Raipur Safe Map v0.1"
```

Now connect to GitHub. Copy the remote URL from the GitHub page you just created —
it looks like `https://github.com/YOUR_USERNAME/raipur-safe-map.git`:

```bash
git remote add origin https://github.com/YOUR_USERNAME/raipur-safe-map.git
git branch -M main
git push -u origin main
```

It will ask for your GitHub username and password. For the password, GitHub no longer
accepts your account password — you need a **Personal Access Token**:
- Go to GitHub → Settings (your profile, top right) → Developer settings →
  Personal access tokens → Tokens (classic) → Generate new token.
- Give it a name like "my laptop", set expiry to 90 days, check the `repo` checkbox,
  click Generate.
- Copy the token and paste it as the password.

After the push, refresh your GitHub repository page — your code is there.

> Going forward: every time you make changes, push them with:
> ```
> git add .
> git commit -m "describe what you changed"
> git push
> ```

---

## Part 4 — Deploy to Vercel and Render

Here's the honest picture of how deployment works:

```
VERCEL (free)                RENDER (free)              SUPABASE (free)
─────────────────            ─────────────────          ─────────────────
React frontend     ──api──▶  FastAPI backend  ──db──▶   PostgreSQL
                             (Python)                   + PostGIS
```

**Vercel** is excellent for the frontend (React). It auto-deploys every time you push
to GitHub.

**Render** is the simplest free hosting for a Python backend. You don't need to
change any of your existing code.

**Supabase** gives you a free PostgreSQL database with PostGIS already installed.

### 4.1 Set up Supabase (production database)

1. Go to **https://supabase.com** and sign up (free, use GitHub login).
2. Click **New project**. Name it `raipur-safemap`. Choose the region closest to
   India (Singapore or Mumbai). Set a strong database password — save it somewhere.
3. Wait 2 minutes for the project to provision.
4. Go to **Settings → Database**. You'll see a connection string labelled
   **"Connection string" → "URI"**. It looks like:
   ```
   postgresql://postgres:[YOUR-PASSWORD]@db.xxxxxxxxxxxx.supabase.co:5432/postgres
   ```
   Copy this — you'll need it in steps 4.2 and 4.3.
5. Go to **SQL Editor** and run this to enable PostGIS:
   ```sql
   CREATE EXTENSION IF NOT EXISTS postgis;
   ```
   Click Run. You should see "Success".

### 4.2 Deploy the backend on Render

1. Go to **https://render.com** and sign up (use GitHub login — it gives Render access
   to your repos).
2. Click **New** → **Web Service**.
3. Connect it to your `raipur-safe-map` repository.
4. Fill in the settings:
   - **Name:** `raipur-safe-map-api`
   - **Root Directory:** `backend`
   - **Runtime:** `Python 3`
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `uvicorn main:app --host 0.0.0.0 --port $PORT`
5. Under **Environment Variables**, add one variable:
   - Key: `DATABASE_URL`
   - Value: the Supabase connection string from step 4.1
6. Click **Create Web Service**.

Render will build and deploy. It takes about 3-5 minutes the first time.

Once it's live, Render gives you a URL like `https://raipur-safe-map-api.onrender.com`.
Copy it.

7. **Seed the production database.** Open Render's Shell tab for your service and run:
   ```
   python -m scripts.seed_db
   ```

> **Free tier note:** Render's free tier spins the server down after 15 minutes of
> inactivity. The first request after a sleep wakes it up — it takes about 30 seconds.
> This is fine for a portfolio project. Paid plans keep it always on.

### 4.3 Deploy the frontend on Vercel

First, you need to tell the React app where the production API is. Create a file
called `.env.production` inside the `frontend/` folder:

```
VITE_API_BASE_URL=https://raipur-safe-map-api.onrender.com
```

Then update `frontend/src/api.js` — change the BASE line from:

```javascript
const BASE = '/api';
```

to:

```javascript
const BASE = import.meta.env.VITE_API_BASE_URL
  ? import.meta.env.VITE_API_BASE_URL + '/api'
  : '/api';
```

This means: in development, use the Vite proxy to `/api`; in production, use the
Render URL. Commit and push this change:

```bash
git add .
git commit -m "Add production API URL for Vercel deployment"
git push
```

Now deploy:

1. Go to **https://vercel.com** and sign up (use GitHub login).
2. Click **Add New** → **Project**.
3. Import your `raipur-safe-map` repository.
4. On the configuration screen:
   - **Framework Preset:** Vercel should auto-detect Vite.
   - **Root Directory:** click Edit and type `frontend`.
   - **Build Command:** `npm run build`
   - **Output Directory:** `dist`
5. Under **Environment Variables**, add:
   - Key: `VITE_API_BASE_URL`
   - Value: `https://raipur-safe-map-api.onrender.com`
6. Click **Deploy**.

Vercel builds the React app and gives you a URL like
`https://raipur-safe-map.vercel.app`.

> From now on, every time you `git push`, Vercel **automatically re-deploys** the
> frontend. Zero manual steps.

### 4.4 Fix CORS for production

Your FastAPI backend currently only allows requests from `localhost:5173`. You need to
add the Vercel URL. In `backend/main.py`, update the CORS section:

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "http://localhost:5173",
        "http://localhost:3000",
        "https://raipur-safe-map.vercel.app",  # add your Vercel URL
    ],
    ...
)
```

Commit and push. Render will auto-deploy the backend update.

### 4.5 Host dashboard.html on Vercel (optional)

If you just want the beautiful standalone dashboard live without any backend, Vercel
can serve it as a static file in about 30 seconds:

1. In your GitHub repo, `dashboard.html` is in the project root.
2. In Vercel, create a **new project** importing the same repo.
3. Set **Root Directory** to `.` (the project root).
4. Set **Framework Preset** to "Other".
5. Leave Build Command blank.
6. Set **Output Directory** to `.`
7. Deploy.

Vercel will serve `dashboard.html` at the root URL. It'll work immediately because it
has no backend dependency — all the seed data is built in.

---

## Part 5 — Quick reference

| What you want to do | Command / Where |
|---------------------|----------------|
| Start everything locally | See Part 2.4 |
| Stop the database | `docker compose down` |
| Save your code to GitHub | `git add . && git commit -m "message" && git push` |
| See backend errors in production | Render dashboard → Logs tab |
| See frontend errors in production | Vercel dashboard → Functions tab |
| Add new Python library | `pip install X && pip freeze > requirements.txt` then push |
| Add new npm library | `npm install X` in `frontend/`, then push |
| Re-seed the production database | Render Shell → `python -m scripts.seed_db` |

---

## Common problems and fixes

**`pip install` fails with permission error on Windows**
Make sure your virtual environment is activated — your prompt should show `(.venv)`.

**`uvicorn: command not found`**
Your virtual environment isn't activated. Run the `activate` command from Part 2.2.

**Backend says "could not connect to database"**
Docker isn't running. Open Docker Desktop, wait for the whale icon to turn solid, then
`docker compose up -d db`.

**Vercel build fails with "could not resolve import"**
The `node_modules` folder needs to be gitignored (it is, if you used the .gitignore
above). Vercel runs `npm install` itself.

**Frontend shows blank/error in production but works locally**
Check that `VITE_API_BASE_URL` is set in Vercel environment variables (Part 4.3 step 5).
Also check that your Render backend is awake — the first request after sleep takes ~30s.

**`git push` asks for password but rejects it**
GitHub needs a Personal Access Token, not your account password. See Part 3.3.
