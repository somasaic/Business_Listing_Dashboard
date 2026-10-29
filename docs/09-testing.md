# Step 9 — Testing and Verification

> **What we build:** **33 automated tests** (15 API + 18 scraper/cleaner), plus a
> manual end-to-end checklist.
>
> **Why:** tests prove the code does what the docs claim, catch regressions when
> code changes, and count toward *Code Quality*. One test caught a real bug
> during development (see §4).

---

## 1. Running the tests

```bash
# from the project root, with the virtualenv active
cd backend && python -m pytest -q          # 15 API tests
cd ..      && python -m pytest scraper/tests -q   # 18 scraper tests
```

```
backend:  15 passed
scraper:  18 passed
```

---

## 2. API tests (`backend/tests/test_api.py`)

### How they run without MySQL
`conftest.py` sets `DATABASE_URL=sqlite+pysqlite:///:memory:` **before** importing
the app. SQLAlchemy then talks to an in-memory SQLite database, and each test
gets a fresh, empty table via a **pytest fixture**:

```python
@pytest.fixture()
def client():
    Base.metadata.drop_all(bind=engine)
    Base.metadata.create_all(bind=engine)
    with TestClient(app) as test_client:
        yield test_client
```

This works because the app only uses **portable SQL** (INSERT, SELECT, GROUP BY,
COUNT, LIKE). The same code path was then verified manually against real MySQL
(§3).

> **Concept — `TestClient`:** calls the FastAPI app in-process, without starting
> a server. The full request pipeline (validation, dependencies, response models)
> still runs.

### What is covered

| Test | Proves |
|---|---|
| `test_bulk_insert_inserts_all_rows` | happy path returns `201` and the right counts |
| `test_bulk_insert_is_idempotent` | the second identical upload inserts 0 |
| `test_duplicates_inside_one_request_are_skipped` | `"Test Cafe"` and `"  TEST cafe "` count as one |
| `test_bulk_insert_rejects_invalid_rows` | `422`, and **nothing** is written (all-or-nothing) |
| `test_bulk_insert_rejects_empty_list` | `min_length=1` is enforced |
| `test_blank_optional_fields_become_null` | `""` phone/address are stored as `NULL` |
| `test_city_count` / `test_source_count` | correct `GROUP BY` output and shape |
| `test_category_count_sorted_by_count_desc` | ordering contract |
| `test_count_filters_and_limit` | filters narrow results; `limit` keeps the full `total` |
| `test_summary` | distinct counts and `COUNT(phone)` skips NULL |
| `test_empty_database_returns_zeroes` | no crash on an empty table |
| `test_list_listings_pagination_and_search` | `pages` maths, `LIKE` search |
| `test_page_size_is_capped` | `page_size=1000` returns `422` |
| `test_health` | health endpoint |

---

## 3. Scraper tests (`scraper/tests/test_cleaner.py`)

Pure functions are easy to test with **parametrised** tests:

```python
@pytest.mark.parametrize("raw, expected", [
    ("+91 80 2222 3333", "+918022223333"),
    ("080-22223333", "+918022223333"),
    ("098450 12345; 9845012346", "+919845012345"),
    ("1800 102 1234", "18001021234"),
    ("12345", None),
])
def test_normalise_phone(raw, expected):
    assert normalise_phone(raw) == expected
```

Covered: phone normalisation (9 cases), city aliases, text cleanup, validation
and de-dupe, OSM tag → listing conversion, address building, and **infobox HTML
parsing with BeautifulSoup** on a small inline HTML sample (no network needed).

---

## 4. A bug the tests caught

The first version of `test_parse_infobox_and_display_name` failed:

```
{'location': 'Adyar , Chennai'} != {'location': 'Adyar, Chennai'}
```

BeautifulSoup's `get_text(" ")` puts a space between every HTML node, so text
after a removed `<sup>` citation ended up as `"Adyar , Chennai"`. The fix was a
regex in both `wikipedia_scraper.clean_text` and `cleaner.clean_text`:
`re.sub(r"\s+([,;.)])", r"\1", text)`. A spot-check of the real data also found
`"Pune,, Deccan"` (double commas from OSM), which got its own rule and test.

---

## 5. Manual end-to-end verification (done against real MySQL)

| Check | Result |
|---|---|
| `GET /api/health` | `{"status":"ok","database":"connected"}` |
| `SHOW CREATE TABLE listing_master` | matches `schema.sql` (indexes, unique hash, utf8mb4) |
| Upload 2,733 rows via API | `inserted: 2733` |
| Upload again | `inserted: 0, duplicates_skipped: 2733` |
| `SELECT city, COUNT(*) … GROUP BY city` vs `/api/dashboard/city-count` | identical |
| Dashboard in browser (desktop light, mobile dark) | all charts, cards and table render; no horizontal scroll on mobile |
| Dump restored into a scratch database | 2,733 rows, 2,733 unique hashes |
| `npm run lint` / `npm run build` | 0 warnings / success |

➡️ Next: [Step 10 — database dump](10-database-dump.md)
