# Step 1 — Environment Setup

> **What we build:** a working development machine. That means Python, Node.js,
> MySQL and Git, plus an isolated Python virtual environment.
>
> **Why:** every later step depends on these tools. Pinning versions makes the
> project reproducible on the reviewer's machine.

---

## 1. Tools and versions used

| Tool | Version | Used for |
|---|---|---|
| Python | 3.13 | scraper + FastAPI backend |
| Node.js / npm | 24 / 11 | React frontend (Vite dev server and build) |
| MySQL | 8.4 | database (via Docker, or a local install) |
| Docker Desktop | 29 | *optional* way to run MySQL with one command |
| Git | 2.52 | version control, push to GitHub |
| VS Code | latest | editor |

Check what is installed:

```bash
python --version
node --version
npm --version
git --version
docker --version      # optional
mysql --version       # only if MySQL is installed locally
```

---

## 2. MySQL — two options

### Option A: Docker (what we used)
`docker-compose.yml` in the project root runs MySQL 8.4 and **automatically
runs `database/schema.sql`** the first time the container starts.

```bash
docker compose up -d          # start (first run downloads the image)
docker ps                     # STATUS should say "(healthy)"
docker compose down           # stop (data is kept in a Docker volume)
```

Credentials (also in `backend/.env.example`):

| Setting | Value |
|---|---|
| host / port | `localhost` / `3306` |
| database | `business_listings` |
| user / password | `listings_user` / `listings_pass` |
| root password | `root123` |

> **Concept — Docker volume:** the `mysql_data` volume stores MySQL's files
> outside the container, so `docker compose down` and `up` keep the data.
> `docker compose down -v` deletes it.

> **Concept — `docker-entrypoint-initdb.d`:** the official MySQL image runs any
> `.sql` file in that folder on the first start, when the data directory is empty.
> We mount `schema.sql` there.

### Option B: Local MySQL install (Windows)
1. Download **MySQL Installer** from dev.mysql.com and install *MySQL Server 8.x* + *MySQL Workbench*.
2. Set a root password during setup.
3. Create the schema and a user:
   ```sql
   SOURCE database/schema.sql;
   CREATE USER 'listings_user'@'localhost' IDENTIFIED BY 'listings_pass';
   GRANT ALL PRIVILEGES ON business_listings.* TO 'listings_user'@'localhost';
   ```
4. Add `C:\Program Files\MySQL\MySQL Server 8.x\bin` to `PATH` so `mysql` and `mysqldump` work in a terminal.

---

## 3. Python virtual environment

```bash
# from the project root
python -m venv .venv

# activate
.venv\Scripts\activate          # Windows (PowerShell / cmd)
source .venv/bin/activate       # macOS / Linux / Git Bash: source .venv/Scripts/activate

pip install -r backend/requirements.txt -r scraper/requirements.txt
```

> **Concept — virtual environment (`venv`):** a private folder of Python
> packages for this project only. It avoids version clashes with other projects
> and makes `requirements.txt` the single source of truth for dependencies.

> **Concept — pinned versions (`fastapi==0.162.0`):** the reviewer installs
> *exactly* what we tested with, so "works on my machine" problems are avoided.

### Python packages and why each one is needed

| Package | Layer | Why |
|---|---|---|
| `requests` | scraper | HTTP client for the Overpass and MediaWiki APIs |
| `beautifulsoup4` + `lxml` | scraper | parse Wikipedia HTML (lxml is the fast parser BeautifulSoup uses) |
| `fastapi` | backend | the web framework (routing, validation, OpenAPI docs) |
| `uvicorn[standard]` | backend | ASGI server that runs the FastAPI app |
| `SQLAlchemy` | backend | ORM / SQL toolkit, used to talk to MySQL from Python |
| `PyMySQL` | backend | pure-Python MySQL driver used by SQLAlchemy |
| `cryptography` | backend | needed by PyMySQL for MySQL 8's `caching_sha2_password` login |
| `python-dotenv` | backend | loads `backend/.env` into environment variables |
| `pytest`, `httpx` | tests | test runner and the HTTP client behind FastAPI's `TestClient` |

---

## 4. Node / React

```bash
cd frontend
npm install           # installs react, react-dom, recharts, vite …
```

The project was created with:

```bash
npm create vite@latest frontend -- --template react
npm install recharts
```

> **Concept — Vite:** a modern build tool. It serves the app instantly during
> development, with hot reload, and bundles it for production with `npm run build`.
> It replaced the older Create React App.

---

## 5. Environment variables (`.env`)

Secrets and machine-specific settings are **never hard-coded**:

| File | Committed? | Purpose |
|---|---|---|
| `backend/.env.example` | ✅ yes | template showing which variables exist |
| `backend/.env` | ❌ no (`.gitignore`) | your real values |
| `frontend/.env.example` | ✅ yes | `VITE_API_URL=http://localhost:8000` |

```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

> **Concept — 12-factor configuration:** the same code runs anywhere, and only
> the environment changes. Vite exposes only variables prefixed `VITE_` to the
> browser, which stops server secrets from leaking into the frontend bundle.

---

## 6. Git

```bash
git init
git remote add origin https://github.com/somasaic/Business_Listing_Dashboard.git
```

`.gitignore` excludes `.venv/`, `node_modules/`, `.env` and the raw scraper cache.

✅ **Result of this step:** all tools installed, MySQL running and healthy,
Python and Node dependencies installed.
