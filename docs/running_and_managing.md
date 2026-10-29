# Running and managing the Business Listings Dashboard

Run all of these in **PowerShell** from `F:\Python_Assignment_Pavan` unless a step says otherwise.

The API, dashboard and MySQL are running right now in this session. The API and dashboard servers stop when this session ends, so use the steps below to start everything yourself.

## 1. Start everything (do this each time)

You need **3 terminals**. In VS Code, `Ctrl+Shift+5` splits the terminal.

**Step 1: Start Docker Desktop** from the Start menu and wait until it shows "Engine running".

**Step 2: Start MySQL** (terminal 1):
```powershell
cd F:\Python_Assignment_Pavan
docker compose up -d
docker ps          # STATUS must show "(healthy)". Wait about 10 seconds if it says "starting".
```

**Step 3: Start the API** (terminal 2):
```powershell
cd F:\Python_Assignment_Pavan
.\.venv\Scripts\Activate.ps1
cd backend
uvicorn app.main:app --reload
```
Open http://localhost:8000/docs to see Swagger.

**Step 4: Start the dashboard** (terminal 3):
```powershell
cd F:\Python_Assignment_Pavan\frontend
npm run dev
```
Open http://localhost:5173 to see the dashboard.

Your data is kept in a Docker volume, so you **don't** need to reload it after a restart.

## 2. Stop everything

| What | How |
|---|---|
| API and dashboard | Press `Ctrl+C` in their terminals |
| MySQL (data is kept) | `docker compose down` |
| MySQL and **delete all data** | `docker compose down -v` |

## 3. Check that it's working

```powershell
docker ps                                            # Is MySQL up and healthy?
curl http://localhost:8000/api/health                # Is the API connected to the DB?
curl http://localhost:8000/api/dashboard/summary     # Is the data there? (should say 2733)
```

## 4. Look at the data in MySQL

```powershell
docker exec -it listings-mysql mysql -ulistings_user -plistings_pass business_listings
```
```sql
SELECT COUNT(*) FROM listing_master;
SELECT city, COUNT(*) c FROM listing_master GROUP BY city ORDER BY c DESC;
SELECT * FROM listing_master LIMIT 5;
exit
```
If you prefer a GUI, connect MySQL Workbench to `localhost:3306` with user `listings_user` and password `listings_pass`.

## 5. Manage the data

Activate the virtualenv first: `.\.venv\Scripts\Activate.ps1`. The API must be running for the upload commands.

| Task | Command |
|---|---|
| Empty the table | `docker exec listings-mysql mysql -uroot -proot123 -e "TRUNCATE business_listings.listing_master"` |
| Reload the scraped data through the API | `python -m scraper.uploader` |
| Re-run cleaning, then upload | `python -m scraper.run_pipeline --skip-scrape --upload` |
| **Re-scrape from scratch** (about 10 min) | Delete `scraper\data\raw`, then run `python -m scraper.run_pipeline --upload` |
| Load from the SQL dump instead | `Get-Content database\listings_dump.sql \| docker exec -i listings-mysql mysql -uroot -proot123` |
| Regenerate the dump after the data changes | `docker exec listings-mysql mysqldump -uroot -proot123 --databases business_listings --single-transaction --no-tablespaces --set-gtid-purged=OFF --default-character-set=utf8mb4 > database\listings_dump.sql` |
| Full reset (new empty DB, then reload) | `docker compose down -v`, then `docker compose up -d`, wait until healthy, then `python -m scraper.uploader` |

To add a city or category, edit `scraper\config.py` (the `CITIES`, `OSM_CATEGORIES` and `WIKI_CATEGORIES` lists), then re-scrape.

> ⚠️ **PowerShell note on the dump:** `>` in Windows PowerShell 5.1 writes UTF-16, which MySQL can't import. Run the regenerate command in **Git Bash** (that's how it was made), or pipe it through `| Out-File -Encoding utf8 database\listings_dump.sql` instead.

## 6. Run the tests

```powershell
cd backend; python -m pytest -q; cd ..          # 15 API tests (no MySQL needed)
python -m pytest scraper/tests -q               # 18 scraper tests
cd frontend; npm run lint; npm run build; cd .. # checks the frontend
```

## 7. Save changes to GitHub

```powershell
git status                      # see what changed
git add .
git commit -m "Describe your change"
git push
```
`.env`, `.venv` and `node_modules` are ignored automatically, so they're never pushed.

## 8. Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `Activate.ps1 cannot be loaded … running scripts is disabled` | PowerShell script policy | Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once |
| `docker: failed to connect to the docker API` | Docker Desktop isn't running | Start Docker Desktop and wait for "Engine running" |
| Dashboard shows a red banner: "Could not load dashboard data" | The API isn't running | Start the API (step 3 in section 1) |
| API returns `503 Database unavailable` | MySQL is down or still starting | Run `docker compose up -d` and wait for `(healthy)` |
| Dashboard loads but shows 0 listings | The table is empty, for example after `down -v` | Run `python -m scraper.uploader` |
| `Address already in use` / port 8000 or 5173 busy | An old server is still running | Run `netstat -ano \| findstr :8000`, then `taskkill /PID <pid> /F` |
| Port 3306 busy | A local MySQL is already installed and running | Stop it in `services.msc`, or change the compose port to `"3307:3306"` and set `MYSQL_PORT=3307` in `backend\.env` |
| CORS error in the browser console | The frontend is on a different URL or port | Add that URL to `CORS_ORIGINS` in `backend\.env`, then restart the API |
| `ModuleNotFoundError: fastapi` | The virtualenv isn't activated | Run `.\.venv\Scripts\Activate.ps1` |
| `ModuleNotFoundError: scraper` | Running from the wrong folder | Run `python -m scraper...` from the **project root** |
| Uploader fails with `Connection refused` | The API isn't running | Start the API first |
| Scraper is slow or times out | The free public map data server is busy | Re-run it; it retries, falls back to other servers, and reuses its saved responses |

## 9. On a new computer (or for the reviewer)

```powershell
git clone https://github.com/somasaic/Business_Listing_Dashboard.git
cd Business_Listing_Dashboard
docker compose up -d
python -m venv .venv; .\.venv\Scripts\Activate.ps1
pip install -r backend/requirements.txt -r scraper/requirements.txt
copy backend\.env.example backend\.env
copy frontend\.env.example frontend\.env
cd frontend; npm install; cd ..
```
Then start the API (step 3), run `python -m scraper.uploader` once to load the data, and start the dashboard (step 4).

I can also save this as `docs/12-running-and-operations.md` and push it with the rest of the docs.