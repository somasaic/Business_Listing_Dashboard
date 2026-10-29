# Step 11 — Demo Video and Submission

> **What we do:** record the 3–5 minute demo video, push the code to GitHub,
> and share a Google Drive folder with Honeybee Digital.

---

## 1. Final checklist (from the PDF and the email)

| # | Item | Where | Status |
|---|---|---|---|
| 1 | GitHub repo, separate `frontend/` and `backend/` | https://github.com/somasaic/Business_Listing_Dashboard | ✅ |
| 2 | Demo video, 3–5 min (scraping, API, dashboard) | Google Drive folder | ⬜ record |
| 3 | Database dump `.sql` | `database/listings_dump.sql` (+ copy in Drive) | ✅ |
| 4 | README with setup, tech stack, challenges | `README.md` | ✅ |
| 5 | Name and contact details | README + `contact.txt` in Drive | ⬜ add phone |
| 6 | Folder shared with `career@honeybeedigital.com` | Google Drive → Share | ⬜ |
| 7 | Within 72 hours of the email | — | ⬜ |

---

## 2. Before recording: start everything

```bash
docker compose up -d                                   # MySQL
cd backend  && uvicorn app.main:app --reload           # terminal 1 → :8000
cd frontend && npm run dev                             # terminal 2 → :5173
```

Open three browser tabs: **Swagger** (`localhost:8000/docs`), the **dashboard**
(`localhost:5173`), and **MySQL** (Workbench, or a terminal running `docker exec -it listings-mysql mysql …`).

Recording tools: **OBS Studio** (free) or **Windows Game Bar** (`Win + G`).
Record at 1080p, use a microphone if you have one, close notifications, and zoom
the browser to 110–125% so text is readable.

---

## 3. Demo script (about 4½ minutes)

| Time | Show | Say (summary) |
|---|---|---|
| **0:00–0:20** | README on GitHub | "Hi, I'm Pavan P. This is a Business Listings Dashboard built with React, FastAPI and MySQL. It scrapes 2,700+ real listings, stores them in MySQL and visualises them." |
| **0:20–1:30** | `scraper/config.py`, `osm_scraper.py`, `wikipedia_scraper.py`, then run `python -m scraper.cleaner` | "Google Maps and Justdial forbid scraping in their terms, so I used two open sources: OpenStreetMap through the Overpass API and Wikipedia through BeautifulSoup HTML parsing. The client is polite: User-Agent, delays, retries, caching. The cleaner normalises city names and phone numbers and removes duplicates: 2,740 raw rows become 2,733 clean ones." |
| **1:30–2:45** | Swagger: `POST /api/listings/bulk` (Execute twice), then `city-count`, `category-count?city=Mumbai`, `source-count` | "The bulk API validates every row with Pydantic and inserts new rows in one transaction. Duplicates are detected by a SHA-256 hash with a unique index, so running it twice inserts nothing new. The count endpoints are GROUP BY queries with optional filters." |
| **2:45–3:05** | MySQL: `SELECT city, COUNT(*) … GROUP BY city` | "Same numbers straight from MySQL. The dashboard is really connected to the database." |
| **3:05–4:15** | Dashboard: cards → city chart → click *Mumbai* → category/source update → Table toggle → search the listings table → dark mode / phone width | "Stat cards, a city bar chart, a category bar chart and a source donut. Clicking a bar filters everything else. Every chart has a table view, and it works on mobile and in dark mode." |
| **4:15–4:40** | README "Challenges" section | "Challenges: choosing ToS-friendly sources, inconsistent addresses and phones, duplicates, and an Overpass server rejecting anonymous requests. Thanks for watching." |

---

## 4. GitHub

```bash
git add .
git commit -m "Business Listings Dashboard: scraper, FastAPI, MySQL, React"
git push -u origin main
```

Make sure the repository is **public** (or add the reviewer as a collaborator)
and that `.env` files are **not** committed.

---

## 5. Google Drive folder

Name: **`Pavan P – Python Development Intern Assignment`**

```
├── Demo_Video_Pavan_P.mp4
├── listings_dump.sql
├── listings_clean.csv            (optional – the scraped dataset)
├── screenshots/                  (optional)
└── contact.txt                   → name, email, phone, GitHub link
```

Share → *General access: Restricted* → add `career@honeybeedigital.com` as **Viewer**.

### Suggested email

> **Subject:** Python Development Intern – Assignment Submission – Pavan P
>
> Dear Honeybee Digital Team,
>
> Please find my submission for the Python Development Intern assignment:
>
> • GitHub repository: https://github.com/somasaic/Business_Listing_Dashboard
> • Google Drive folder (demo video, SQL dump, dataset): <link>
>
> The project scrapes 2,733 business listings from OpenStreetMap and Wikipedia,
> stores them in MySQL, serves them through a FastAPI REST API, and visualises
> them in a React + Recharts dashboard. Setup steps and the challenges I faced
> are in the README.
>
> Thank you for the opportunity. I look forward to your feedback.
>
> Regards,
> Pavan P
> <phone> · somasaicheviti@gmail.com
