# Step 2 — Project Structure

> **What we build:** the folder layout of the repository.
>
> **Why:** the brief explicitly asks for **separate folders for frontend and
> backend**. A clear structure also scores on *Code Quality*, because a reviewer
> can find anything in seconds.

---

## 1. The tree

```
Business_Listing_Dashboard/
├── README.md                    ← main entry point: setup, tech stack, challenges
├── docker-compose.yml           ← one-command MySQL
├── .gitignore
│
├── database/
│   ├── schema.sql               ← CREATE DATABASE + CREATE TABLE listing_master
│   └── listings_dump.sql        ← mysqldump of the final data (submission item)
│
├── scraper/                     ← Part 1: data collection (Python package)
│   ├── config.py                ← cities, categories, endpoints, cleaning rules
│   ├── http_client.py           ← polite HTTP client (User-Agent, delay, retries)
│   ├── osm_scraper.py           ← source 1: OpenStreetMap / Overpass API
│   ├── wikipedia_scraper.py     ← source 2: Wikipedia API + BeautifulSoup
│   ├── cleaner.py               ← normalise, validate, de-duplicate
│   ├── uploader.py              ← POST cleaned data to the FastAPI bulk endpoint
│   ├── run_pipeline.py          ← one command for the whole pipeline
│   ├── requirements.txt
│   ├── data/                    ← scraped + cleaned JSON / CSV
│   └── tests/test_cleaner.py
│
├── backend/                     ← Part 3: FastAPI
│   ├── app/
│   │   ├── main.py              ← app object, CORS, error handler, health check
│   │   ├── config.py            ← settings from .env
│   │   ├── database.py          ← SQLAlchemy engine, session, get_db dependency
│   │   ├── models.py            ← ORM model for listing_master
│   │   ├── schemas.py           ← Pydantic request/response models
│   │   ├── crud.py              ← all SQL queries (service layer)
│   │   └── routers/
│   │       ├── listings.py      ← POST /api/listings/bulk, GET /api/listings
│   │       └── dashboard.py     ← GET /api/dashboard/{summary,city-count,…}
│   ├── tests/                   ← pytest API tests
│   ├── requirements.txt
│   └── .env.example
│
├── frontend/                    ← Part 4: React dashboard
│   ├── src/
│   │   ├── api/client.js        ← fetch wrapper for the API
│   │   ├── hooks/useDashboardData.js
│   │   ├── components/          ← StatCards, ChartCard, CountBarChart, SourcePieChart, …
│   │   ├── App.jsx              ← page layout + filter state
│   │   └── index.css            ← design tokens, light/dark theme
│   ├── package.json
│   └── .env.example
│
└── docs/                        ← this step-by-step documentation
```

---

## 2. Design principles behind the layout

| Principle | How it shows up |
|---|---|
| **Separation of concerns** | Scraping, API and UI are independent. Each can be run, tested and replaced on its own. |
| **Single responsibility** | Each Python module does *one* thing: `osm_scraper` only fetches OSM, `cleaner` only cleans. |
| **Configuration over code** | All "what to scrape" lists live in `scraper/config.py`. Adding a city is a one-line change. |
| **Layered backend** | `routers` (HTTP) → `crud` (business logic / SQL) → `models` (tables). Routers never contain SQL. |
| **Component-based UI** | Each React component renders one piece of the screen and receives data via props. |
| **Secrets out of git** | `.env` files are ignored; `.env.example` documents them. |

> **Concept — Python package:** a folder with `__init__.py`. Because
> `scraper/` is a package, its modules use **relative imports**
> (`from . import config`) and run with `python -m scraper.osm_scraper`, so the
> imports work no matter which folder you start from.

✅ **Result of this step:** an empty but well-organised skeleton that the next
steps fill in.
