# Prerequisites and Setup Guide (from clone to recording)

This guide takes you from a fresh Windows computer to a running dashboard,
ready to record the demo video.

| Part | What you do | Time |
|---|---|---|
| A | Install the prerequisites | 30–60 min (one time) |
| B | Get the code into **your own** GitHub repo | 10 min |
| C | Update personal details to **your** details | 15 min |
| D | Set up and run the project | 20 min |
| E | Verify everything works | 10 min |
| F | Understand the project before recording | 2–3 hours (do not skip) |
| G | Record the demo video | 1 hour |
| H | Submit | 15 min |

---

## Part A — Install the prerequisites

### A1. Hardware
* 8 GB RAM or more (Docker uses about 2 GB)
* About 5 GB free disk space
* Windows 10/11 64-bit (macOS and Linux also work; commands in the notes)

### A2. Software to install

| # | Software | Version | Download | Notes |
|---|---|---|---|---|
| 1 | **Git** | latest | https://git-scm.com/download/win | Keep the defaults. Includes *Git Bash*. |
| 2 | **Python** | 3.11 – 3.13 | https://www.python.org/downloads/ | ⚠️ Tick **"Add python.exe to PATH"** on the first screen. |
| 3 | **Node.js** | 20 LTS or newer | https://nodejs.org/ | Includes `npm`. |
| 4 | **Docker Desktop** | latest | https://www.docker.com/products/docker-desktop/ | Accept **WSL 2** when asked. Restart the PC afterwards. |
| 5 | **VS Code** | latest | https://code.visualstudio.com/ | Extensions: *Python*, *ES7+ React snippets*. |
| 6 | **OBS Studio** | latest | https://obsproject.com/ | For recording the video (Windows Game Bar `Win+G` also works). |
| 7 | MySQL Workbench *(optional)* | 8.x | https://dev.mysql.com/downloads/workbench/ | GUI for viewing tables. Not required. |

> You do **not** need to install MySQL Server. It runs inside Docker.

### A3. Check the installs
Open a **new** PowerShell window and run:

```powershell
git --version          # git version 2.x
python --version       # Python 3.11+ 
node --version         # v20+ 
npm --version          # 10+
docker --version       # Docker version 2x
```

Start **Docker Desktop** from the Start menu and wait until it shows
**"Engine running"**. Then:

```powershell
docker run --rm hello-world     # prints "Hello from Docker!"
```

### A4. One-time PowerShell setting
This lets PowerShell run the Python virtual-environment activation script:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```
Answer `Y`.

### A5. Tell Git who you are
These must be **your own** name and email. Every commit you make will carry them.

```powershell
git config --global user.name  "Your Full Name"
git config --global user.email "your.email@example.com"
```

---

## Part B — Get the code into your own GitHub repo

1. Log in to **your** GitHub account → **New repository**
   * Name: `Business_Listing_Dashboard`
   * Visibility: **Public**
   * **Do not** add a README, .gitignore or license, so the repo stays empty
2. Clone the original project and point it at your repo:

```powershell
cd D:\Projects                     # any folder you like
git clone https://github.com/somasaic/Business_Listing_Dashboard.git
cd Business_Listing_Dashboard

git remote set-url origin https://github.com/<your-github-username>/Business_Listing_Dashboard.git
git push -u origin main
```

> The existing commit history shows who wrote the original code. Keep it as it
> is. Your own work appears as **your** commits on top of it (Part F4).

---

## Part C — Update personal details

These are **all** the places that contain contact details. Replace them with yours:

| # | File | Line (approx.) | Current | Change to |
|---|---|---|---|---|
| 1 | `README.md` | 8 | `**Author:** Pavan P · somasaicheviti@gmail.com` | your name · your email · your phone |
| 2 | `README.md` | 75 | `git clone https://github.com/somasaic/…` | `https://github.com/<your-github-username>/…` |
| 3 | `scraper/config.py` | 26 | `contact: somasaicheviti@gmail.com` in `USER_AGENT` | **your** email. You are the one sending requests to OpenStreetMap and Wikipedia when you re-scrape. |
| 4 | `docs/01-environment-setup.md` | 158 | `git remote add origin https://github.com/somasaic/…` | your repo URL |
| 5 | `docs/11-submission-and-demo.md` | 12, 87 | `https://github.com/somasaic/…` | your repo URL |
| 6 | `docs/11-submission-and-demo.md` | 43, 67, 70, 81, 98 | `Pavan P` | your name, if it is different |
| 7 | `docs/11-submission-and-demo.md` | 99 | `<phone> · somasaicheviti@gmail.com` | your phone · your email |
| 8 | `docs/running_and_managing.md` | 3, 15, 22, 31 | `F:\Python_Assignment_Pavan` | the folder you cloned into |
| 9 | `docs/running_and_managing.md` | 123 | `https://github.com/somasaic/…` | your repo URL |

Use VS Code to find any you missed: press `Ctrl+Shift+F` and search for
`somasaic`. There should be **0 results** in project files, apart from this
guide and the git history.

### C2. Add an honest Acknowledgements section (required)
Add this section to the end of `README.md` and adjust it to what is true for you:

```markdown
## Acknowledgements
The initial version of this project was built with the help of a friend and the
AI assistant Claude (Anthropic). I set it up on my own machine, studied every part,
and made these changes myself: <list your own changes from Part F4>.
I can explain the scraping approach, the API design and the dashboard in detail.
```

Companies increasingly accept AI-assisted work when it is **disclosed** and the
candidate understands it. Hiding it is what causes problems in interviews.

### C3. Commit
```powershell
git add .
git commit -m "Update contact details and add acknowledgements"
git push
```

---

## Part D — Set up and run the project

Run everything from the cloned project folder in PowerShell.

### D1. Start MySQL (Docker)
```powershell
docker compose up -d
docker ps
```
Wait until the `listings-mysql` row shows **`(healthy)`** (about 20–60 s the
first time, while the MySQL image downloads). The database
`business_listings` and the table `listing_master` are created automatically.

### D2. Python environment and backend dependencies
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1          # prompt now starts with (.venv)
python -m pip install --upgrade pip
pip install -r backend/requirements.txt -r scraper/requirements.txt
```

### D3. Environment files
```powershell
copy backend\.env.example backend\.env
copy frontend\.env.example frontend\.env
```
The defaults already match `docker-compose.yml`, so you don't need to edit them.

### D4. Start the API (terminal 1)
```powershell
.\.venv\Scripts\Activate.ps1
cd backend
uvicorn app.main:app --reload
```
Wait for `Application startup complete.`, then open **http://localhost:8000/docs**.

### D5. Load the data (terminal 2, project root)
```powershell
.\.venv\Scripts\Activate.ps1
python -m scraper.uploader
```
Expected last line:
`upload finished: {'received': 2733, 'inserted': 2733, 'duplicates_skipped': 0}`

> Alternatives: re-scrape everything yourself (about 10 min) with
> `python -m scraper.run_pipeline --upload`, or import the SQL dump with
> `Get-Content database\listings_dump.sql | docker exec -i listings-mysql mysql -uroot -proot123`.

### D6. Start the dashboard (terminal 3)
```powershell
cd frontend
npm install          # first time only, about 1 min
npm run dev
```
Open **http://localhost:5173**.

---

## Part E — Verify

| Check | How | Expected |
|---|---|---|
| MySQL healthy | `docker ps` | `(healthy)` |
| API connected | http://localhost:8000/api/health | `{"status":"ok","database":"connected"}` |
| Data loaded | http://localhost:8000/api/dashboard/summary | `"total_listings":2733` |
| Dashboard | http://localhost:5173 | cards, 2 bar charts, donut, listings table |
| Filtering | click the *Mumbai* bar | other charts update and the bar is highlighted |
| Backend tests | `cd backend; python -m pytest -q` | `15 passed` |
| Scraper tests | `python -m pytest scraper/tests -q` (project root) | `18 passed` |
| Frontend checks | `cd frontend; npm run lint; npm run build` | no warnings, build succeeds |

Problems? See the troubleshooting table in [running_and_managing.md](running_and_managing.md).

---

## Part F — Understand it before you record (do not skip)

The interviewer will ask "why" questions. Make sure you can answer them yourself.

### F1. Read the docs in order
`docs/00` → `docs/11`. For each step, open the matching code file next to the doc.

### F2. Run each part by hand
```powershell
python -m scraper.osm_scraper          # watch the per-city counts
python -m scraper.cleaner              # read the cleaning stats
```
In Swagger, run `POST /api/listings/bulk` twice and explain why the second run inserts 0.

### F3. Be able to answer these without notes
1. Why OpenStreetMap and Wikipedia instead of Google Maps or Justdial?
2. What does the Overpass query `nwr["amenity"="hospital"]["name"]["addr:street"](bbox)` mean?
3. How does BeautifulSoup extract the address from a Wikipedia infobox?
4. How are phone numbers normalised? Give 3 examples.
5. Why is there a `listing_hash` column, and why not a UNIQUE key on (name, city, address)?
6. What does `Depends(get_db)` do?
7. Why does the bulk API return 422 for one bad row, and why is nothing saved?
8. Which SQL query is behind `/api/dashboard/city-count`? Why is `city` indexed?
9. What is CORS and why is it needed here?
10. Why does the city chart stay unfiltered when you select a city?

### F4. Make your own changes (strongly recommended)
Small, real changes you can talk about. Commit each one:
* Add a city: add a bounding box to `CITIES` in `scraper/config.py` and re-run the pipeline
* Add a category (for example `("amenity", "fuel")` → "Petrol Pump")
* Add a new API endpoint, for example `/api/dashboard/city-category-count`
* Add a chart, for example "listings with vs without phone" per city
* Improve the README with your own screenshots

```powershell
git add .
git commit -m "Add <your change>"
git push
```

---

## Part G — Record the demo video (3–5 min)

### G1. Prepare the "live" setup
1. Docker Desktop running → `docker compose up -d` → `(healthy)`
2. Terminal 1: API running. Terminal 2: dashboard running.
3. Open browser tabs in this order:
   1. your GitHub repo (README)
   2. VS Code with `scraper/config.py`, `osm_scraper.py`, `wikipedia_scraper.py`, `cleaner.py`
   3. http://localhost:8000/docs (Swagger)
   4. a terminal with MySQL open:
      `docker exec -it listings-mysql mysql -ulistings_user -plistings_pass business_listings`
   5. http://localhost:5173 (dashboard)
4. Browser zoom 110–125%, notifications off (Windows Focus Assist), other apps closed.

### G2. OBS settings
* Sources → **Display Capture** (or *Window Capture*) + **Audio Input Capture** (mic)
* Settings → Output → Recording format **mp4**. Video → **1920×1080**, 30 fps
* Do a 10-second test recording and check the audio

### G3. Script
Use the timed script in [11-submission-and-demo.md](11-submission-and-demo.md) §3:
intro → scraping approach → API in Swagger → same numbers in MySQL → dashboard
walkthrough → challenges. Say it **in your own words**, and mention the AI/friend
help in one sentence, as in your README.

---

## Part H — Submit
Follow [11-submission-and-demo.md](11-submission-and-demo.md) §4–5:
1. Push the final code to **your** public repo.
2. Create a Google Drive folder with the video, `database/listings_dump.sql`, and `contact.txt` (your name, email, phone, repo link).
3. Share it with `career@honeybeedigital.com` as a Viewer.
4. Send the email **before the deadline**.

---

## Shutting down after recording
```powershell
# Ctrl+C in the API and dashboard terminals, then:
docker compose down          # keeps the data for next time
```
