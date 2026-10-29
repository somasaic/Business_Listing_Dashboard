# Step 7 — Loading the Data into MySQL (via the Bulk API)

> **What we build:** `scraper/uploader.py`. It reads `listings_clean.json` and
> POSTs it to `/api/listings/bulk` in batches of 250.
>
> **Why through the API and not straight into MySQL?** The brief asks for an
> *Insert Listings API* that bulk-inserts scraped data. Using it for the real
> load proves it works on 2,700+ rows. It also keeps validation and duplicate
> handling in **one** place, since the scraper never needs database credentials.

---

## 1. The code

```python
def batches(rows, size):
    for start in range(0, len(rows), size):
        yield rows[start:start + size]          # generator: one slice at a time

def upload(api_base="http://localhost:8000", batch_size=250):
    rows = json.loads(config.CLEAN_JSON.read_text(encoding="utf-8"))
    for number, batch in enumerate(batches(rows, batch_size), start=1):
        response = requests.post(f"{api_base}/api/listings/bulk",
                                 json={"listings": batch}, timeout=60)
        if response.status_code != 201:
            raise RuntimeError(f"batch {number} failed: {response.status_code} {response.text}")
```

| Concept | Why |
|---|---|
| **Batching** (250 rows per request) | small enough for fast requests and clear progress, large enough to be efficient |
| **Generator** (`yield`) | produces slices lazily, without building a list of lists |
| `requests.post(json=...)` | serialises to JSON and sets `Content-Type: application/json` |
| `timeout=60` | never hang forever if the API is down |
| Fail fast on any non-201 | a bad batch stops the run with the API's error message |

---

## 2. Run it

```bash
# terminal 1 – API must be running
cd backend && uvicorn app.main:app

# terminal 2 – from the project root
python -m scraper.uploader
# or scrape + clean + upload in one go:
python -m scraper.run_pipeline --upload
```

Actual output:
```
INFO batch 1: {'received': 250, 'inserted': 250, 'duplicates_skipped': 0}
...
INFO batch 11: {'received': 233, 'inserted': 233, 'duplicates_skipped': 0}
INFO upload finished: {'received': 2733, 'inserted': 2733, 'duplicates_skipped': 0}
```

Running it **again** proves idempotency:
```
INFO upload finished: {'received': 2733, 'inserted': 0, 'duplicates_skipped': 2733}
```

---

## 3. Verify in MySQL

```bash
docker exec -it listings-mysql mysql -ulistings_user -plistings_pass business_listings
```
```sql
SELECT city, COUNT(*) c FROM listing_master GROUP BY city ORDER BY c DESC;
```
```
Mumbai     434
Bengaluru  415
Delhi      384
Chennai    383
Pune       362
Hyderabad  346
Kolkata    319
Ahmedabad   90
```
These match `GET /api/dashboard/city-count` exactly, which confirms the API
reports what is in the database.

---

## 4. Alternative: test the API by hand in Swagger
1. Open http://localhost:8000/docs
2. Expand **POST /api/listings/bulk** → *Try it out*
3. The example body is pre-filled from the schema's `json_schema_extra`. Click **Execute**.
4. You get `201` and `{"received":1,"inserted":1,"duplicates_skipped":0}`. Execute again and you get `inserted: 0`.

➡️ Next: [Step 8 — React dashboard](08-frontend-react.md)
