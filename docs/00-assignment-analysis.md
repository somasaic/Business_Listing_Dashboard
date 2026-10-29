# Step 0 — Understanding the Assignment

> **Goal of this step:** read the brief carefully, list every requirement, and
> turn it into a plan *before* writing a single line of code.

---

## 1. What we received

| Input | What it contained |
|---|---|
| **Email** from Honeybee Digital (HR Team) | Role: *Python Development Intern*. Deadline: **72 hours (2–3 days)**. Submit a **single Google Drive / WeTransfer folder** shared with `career@honeybeedigital.com`, including a **video explanation** and the candidate's **name & contact details**. |
| **PDF** "Internship Assignment: Python Development Intern" | The technical brief: build a **Business Listings Dashboard** using **React + FastAPI + MySQL**. |

When the two disagree, the **stricter rule wins**:

* The PDF says 3–5 days and the email says 72 hours, so we plan for **72 hours**.
* The PDF says "GitHub (mandatory)" and the email says "Drive / WeTransfer folder", so we do **both**. The code goes to GitHub, and the Drive folder holds the video, the SQL dump and a text file with the GitHub link and contact details.

---

## 2. Requirements checklist

### Mandatory tech stack
| Layer | Required | What we use |
|---|---|---|
| Frontend | React.js | React 19 + Vite |
| Backend API | FastAPI (Python) | FastAPI + SQLAlchemy 2 + Pydantic 2 |
| Database | MySQL | MySQL 8.4 (Docker or local install) |

### Part 1 — Data collection (web scraping)
- [x] **500+** business listings
- [x] Fields: Business Name, Category, City, Address, Phone (if available), Source
- [x] From Google Maps / Justdial / Sulekha / *any other business directory*

### Part 2 — Database design
- [x] MySQL database + table **`listing_master`**
- [x] Columns: `id (PK), business_name, category, city, address, phone, source, created_at`

### Part 3 — Backend (FastAPI)
- [x] **Insert Listings API**: bulk insert scraped data into MySQL
- [x] **Dashboard APIs**: city-wise count, category-wise count, source-wise count

### Part 4 — Frontend (React)
- [x] City-wise, category-wise and source-wise business counts
- [x] Clean, minimal design
- [x] Bar / Pie charts using Chart.js / **Recharts** / ApexCharts

### Expected output
- [x] Displays aggregated data visually
- [x] Fetches data from FastAPI
- [x] Connected to MySQL

### Submission
- [x] GitHub repo with **separate `frontend/` and `backend/` folders**
- [x] Demo video (3–5 min): scraping approach, API working, dashboard functionality
- [x] Database dump (`.sql` file)
- [x] `README.md` with setup instructions, tech stack, challenges faced

---

## 3. Reading between the lines

**Evaluation weights**

| Criteria | Weight | What we did about it |
|---|---|---|
| Functionality | 30% | Every required API and chart works end-to-end, and there are extras (filters, table view, listing browser). |
| Code Quality | 20% | Layered backend (routers → crud → models), typed Pydantic schemas, 33 automated tests, no hard-coded secrets. |
| UI/UX | 15% | Minimal card layout, accessible colours, light/dark mode, tooltips, click-to-filter, chart/table toggle, mobile layout. |
| Data Accuracy | 15% | A dedicated cleaning step (normalised cities and phones, dedupe) plus a unique hash in MySQL so duplicates can't be stored. |
| API Design | 10% | RESTful resource naming, correct status codes (201/422/503), validation, pagination, Swagger docs. |
| Documentation | 10% | Root README plus this `docs/` folder, one file per step. |

**The "Important Notes" in the PDF matter:**

1. *"Avoid scraping in a way that violates website terms of service."*
   Google Maps and Justdial forbid automated scraping in their terms and use
   anti-bot protection. Scraping them hard would show poor judgement, so we chose
   sources that **allow** programmatic access (see [Step 4](04-web-scraping.md)).
2. *"You may use mock data if scraping is blocked."* We didn't need to. All
   listings are **real**.
3. *"Focus on logic, structure, and clarity over perfection."* This is why the
   code is split into small, single-purpose modules, and why we wrote these docs.

---

## 4. Architecture we decided on

```
            ┌─────────────────────────┐
            │  OpenStreetMap          │  Overpass API (ODbL)
            │  Wikipedia              │  MediaWiki API + HTML (CC BY-SA)
            └───────────┬─────────────┘
                        │ HTTP (polite: User-Agent, delay, retries)
                        ▼
  scraper/   osm_scraper.py + wikipedia_scraper.py  ──►  raw JSON
             cleaner.py   (normalise, validate, dedupe) ──► listings_clean.json / .csv
             uploader.py  (batches of 250)
                        │ POST /api/listings/bulk
                        ▼
  backend/   FastAPI  ──  Pydantic validation ── SQLAlchemy ──►  MySQL  listing_master
                        ▲
                        │ GET /api/dashboard/*-count, /summary, /api/listings
  frontend/  React + Recharts dashboard  (http://localhost:5173)
```

**Why the scraper uploads through the API instead of writing to MySQL directly:**
the brief asks for an *Insert Listings API*, and routing every write through it
means validation and duplicate checks live in **one place**.

---

## 5. Step plan (each step has its own doc)

| # | Step | Doc |
|---|---|---|
| 0 | Analyse the assignment | this file |
| 1 | Environment setup | [01-environment-setup.md](01-environment-setup.md) |
| 2 | Project structure | [02-project-structure.md](02-project-structure.md) |
| 3 | Database design (MySQL) | [03-database-design.md](03-database-design.md) |
| 4 | Web scraping | [04-web-scraping.md](04-web-scraping.md) |
| 5 | Data cleaning | [05-data-cleaning.md](05-data-cleaning.md) |
| 6 | Backend API (FastAPI) | [06-backend-fastapi.md](06-backend-fastapi.md) |
| 7 | Loading data via the bulk API | [07-loading-data.md](07-loading-data.md) |
| 8 | Frontend dashboard (React) | [08-frontend-react.md](08-frontend-react.md) |
| 9 | Testing | [09-testing.md](09-testing.md) |
| 10 | Database dump | [10-database-dump.md](10-database-dump.md) |
| 11 | Demo video and submission | [11-submission-and-demo.md](11-submission-and-demo.md) |
