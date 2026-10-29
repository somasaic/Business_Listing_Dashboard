# Step 6 — Backend API (FastAPI)

> **What we build:** a REST API that **bulk-inserts** listings into MySQL and
> serves **city-, category- and source-wise counts** (plus a summary and a
> paginated listing browser) to the React dashboard.
>
> **Why FastAPI:** it's mandatory in the brief, and it fits the job well:
> automatic request validation through Pydantic, automatic interactive docs
> (Swagger), type hints everywhere, and high performance.

Folder: `backend/app/`

---

## 1. Layered architecture

```
HTTP request
   │
   ▼
routers/listings.py, routers/dashboard.py   ← HTTP layer: paths, query params, status codes
   │   (FastAPI validates input with schemas.py)
   ▼
crud.py                                     ← service layer: all SQL / business rules
   │
   ▼
models.py  +  database.py                   ← data layer: ORM table, engine, sessions
   │
   ▼
MySQL  listing_master
```

| File | Responsibility |
|---|---|
| `config.py` | read settings from `.env` (DB host/user/password, CORS origins) |
| `database.py` | create the SQLAlchemy **engine** (connection pool), the **session factory**, and the `get_db` dependency |
| `models.py` | the `Listing` ORM class mapped to `listing_master` |
| `schemas.py` | Pydantic models: **what JSON comes in and goes out** |
| `crud.py` | functions that run queries: `bulk_insert_listings`, `count_by`, `get_summary`, `list_listings` |
| `routers/*.py` | endpoint definitions, kept thin |
| `main.py` | creates the app, adds CORS and the error handler, includes routers, health check |

**Why layers?** Each layer can change without touching the others. For
example, we could swap MySQL for PostgreSQL by changing only `config.py`, and
`crud.py` can be tested without HTTP.

---

## 2. Endpoints

| Method | Path | Purpose | Success |
|---|---|---|---|
| `POST` | `/api/listings/bulk` | **Required:** bulk insert scraped listings | `201 Created` |
| `GET` | `/api/dashboard/city-count` | **Required:** city-wise count | `200` |
| `GET` | `/api/dashboard/category-count` | **Required:** category-wise count | `200` |
| `GET` | `/api/dashboard/source-count` | **Required:** source-wise count | `200` |
| `GET` | `/api/dashboard/summary` | extra: totals for the stat cards | `200` |
| `GET` | `/api/listings` | extra: paginated / filtered / searchable rows | `200` |
| `GET` | `/api/health` | extra: API + database health check | `200` |

Interactive docs: **http://localhost:8000/docs** (Swagger UI) and `/redoc`.

### REST design decisions
* **Nouns in paths, verbs in HTTP methods:** `POST /api/listings/bulk`, not `/insertListings`.
* `/api` prefix: separates API routes from anything else the server might serve.
* **Correct status codes:** `201` for creation, `422` for invalid input (automatic), `503` when MySQL is down.
* **Consistent response shape** for all three count endpoints, so the frontend uses one chart component for all of them.
* **Optional filters and `limit`** on the count endpoints, so the dashboard can cross-filter (for example, category counts in Mumbai only).

---

## 3. Bulk insert API in detail

### Request
```http
POST /api/listings/bulk
Content-Type: application/json

{
  "listings": [
    {"business_name": "Apollo Pharmacy", "category": "Pharmacy", "city": "Bengaluru",
     "address": "12, 100 Feet Road, Indiranagar", "phone": "+918022223333", "source": "OpenStreetMap"}
  ]
}
```

### Response
```json
{ "received": 250, "inserted": 248, "duplicates_skipped": 2 }
```

### Validation (Pydantic, `schemas.py`)
```python
Name = Annotated[str, StringConstraints(strip_whitespace=True, min_length=1, max_length=255)]

class ListingBase(BaseModel):
    business_name: Name
    category: Label            # 1..100 chars
    city: Label
    address: str | None = Field(default=None, max_length=500)
    phone: str | None = Field(default=None, max_length=30)
    source: SourceName         # 1..50 chars

    @field_validator("address", "phone", mode="before")
    @classmethod
    def blank_to_none(cls, value):      # "" or "   "  ->  None
        ...

class BulkInsertRequest(BaseModel):
    listings: list[ListingCreate] = Field(min_length=1, max_length=5000)
```

If **any** row is invalid (empty name, 600-char address, missing field),
FastAPI returns **422** with the exact location of the error, for example
`["body", "listings", 1, "business_name"]`, and **nothing is written**.
This is *all-or-nothing* behaviour. The 5,000-row cap protects the server from
huge payloads.

### The insert algorithm (`crud.bulk_insert_listings`)
```python
# 1. hash each row; drop duplicates inside the same request
row["listing_hash"] = compute_listing_hash(name, city, address)   # SHA-256
unique_rows.setdefault(row["listing_hash"], row)

# 2. ask MySQL which hashes already exist (IN-queries in chunks of 1000)
select(Listing.listing_hash).where(Listing.listing_hash.in_(chunk))

# 3. insert only the new rows with ONE executemany, in ONE transaction
db.execute(insert(Listing), new_rows)
db.commit()
```

| Concept | Why it matters |
|---|---|
| **Bulk insert / `executemany`** | one round-trip for hundreds of rows instead of one per row, which is far faster |
| **Transaction** (`commit` once) | either the whole batch is saved or none of it |
| **Idempotency** | sending the same data twice inserts nothing the second time. We verified this: the second upload returned `inserted: 0, duplicates_skipped: 2733`. |
| **Chunked `IN (...)`** | keeps each SQL statement a reasonable size |
| **UNIQUE index as a final guard** | even with concurrent uploads, MySQL itself refuses a duplicate hash |

---

## 4. Dashboard count APIs in detail

```python
GROUPABLE = {"city": Listing.city, "category": Listing.category, "source": Listing.source}

def count_by(db, dimension, *, limit=None, city=None, category=None, source=None):
    column = GROUPABLE[dimension]
    count = func.count(Listing.id).label("count")
    stmt = select(column, count)
    stmt = _apply_filters(stmt, city, category, source)
    stmt = stmt.group_by(column).order_by(count.desc(), column.asc())
    rows = db.execute(stmt).all()
```

SQL generated (city example):
```sql
SELECT listing_master.city, count(listing_master.id) AS count
FROM listing_master
GROUP BY listing_master.city
ORDER BY count DESC, listing_master.city ASC;
```

Response:
```json
{
  "dimension": "city",
  "total": 2733,
  "items": [
    {"label": "Mumbai", "count": 434},
    {"label": "Bengaluru", "count": 415}
  ]
}
```

* **One generic function** serves all three endpoints (the DRY principle).
* The column comes from a **whitelist dictionary**, never from user text, so
  there is no possibility of SQL injection through the dimension name.
* Filter values go into the query as **bound parameters** (`WHERE city = %s`),
  which is also injection-safe. The same applies to `search` in `LIKE`.
* Secondary sort by label gives a **deterministic order** when counts tie.
* `total` is computed before `limit` is applied, so "top 5" charts can still show correct percentages.

Summary query: one round-trip computes all five numbers:
```sql
SELECT COUNT(id), COUNT(DISTINCT city), COUNT(DISTINCT category),
       COUNT(DISTINCT source), COUNT(phone)      -- COUNT(column) skips NULLs
FROM listing_master;
```

---

## 5. Key FastAPI concepts used

### Dependency Injection: `Depends(get_db)`
```python
def get_db():
    db = SessionLocal()
    try:
        yield db          # FastAPI hands this session to the endpoint
    finally:
        db.close()        # always runs, even if the endpoint raised an error

DB = Annotated[Session, Depends(get_db)]

@router.get("/city-count")
def city_count(db: DB, limit: Limit = None, ...):
```
Each request gets its **own** database session, which is always closed. In
tests the same mechanism lets us point the app at a different database.

### Path operations and query parameters
```python
@router.get("", response_model=schemas.PaginatedListings)
def list_listings(db: DB,
                  page: Annotated[int, Query(ge=1)] = 1,
                  page_size: Annotated[int, Query(ge=1, le=100)] = 20, ...):
```
Type hints **are** the validation: `?page=0` or `?page_size=1000` automatically returns 422.

### `response_model`
Filters and documents the output. Internal fields such as `listing_hash` never leak to clients.

### `APIRouter` and tags
Routes are grouped by feature (`Listings`, `Dashboard`, `Health`), and Swagger shows them in those groups.

### Lifespan event
```python
@asynccontextmanager
async def lifespan(app):
    Base.metadata.create_all(bind=engine)   # create table if missing
    yield
    engine.dispose()                        # close pooled connections on shutdown
```

### CORS middleware
The React app runs on `localhost:5173` and the API on `localhost:8000`. Browsers
treat these as different **origins** and block the requests unless the API
allows it:
```python
app.add_middleware(CORSMiddleware, allow_origins=settings.cors_origins,
                   allow_methods=["GET", "POST"], allow_headers=["*"])
```
Only the known frontend origins are allowed (not `*`), read from `.env`.

### Global exception handler
```python
@app.exception_handler(OperationalError)
async def database_unavailable(request, exc):
    return JSONResponse(status_code=503, content={"detail": "Database unavailable"})
```
If MySQL is down, the client gets a clear **503** instead of a 500 stack trace.

---

## 6. SQLAlchemy concepts used

| Concept | Where |
|---|---|
| **Engine with connection pool** (`pool_size=5`, `max_overflow=10`) | `database.py`. Re-uses DB connections across requests. |
| `pool_pre_ping=True`, `pool_recycle=3600` | avoids "MySQL server has gone away" after idle time |
| `sessionmaker` / `Session` | unit of work per request |
| SQLAlchemy 2.0 `select()` style | all queries in `crud.py` |
| `func.count`, `func.distinct`, `group_by`, `order_by`, `offset`/`limit` | aggregation and pagination |
| `URL.create(...)` | builds the connection URL and safely escapes special characters in the password |
| `mysql+pymysql://` dialect | SQLAlchemy dialect + PyMySQL driver |

---

## 7. Run it

```bash
cd backend
uvicorn app.main:app --reload      # --reload restarts on code changes
```

Try it:
```bash
curl http://localhost:8000/api/health
curl http://localhost:8000/api/dashboard/city-count
curl "http://localhost:8000/api/dashboard/category-count?city=Mumbai&limit=3"
curl "http://localhost:8000/api/listings?search=apollo&page_size=5"
```

➡️ Next: [Step 7 — load the data](07-loading-data.md)
