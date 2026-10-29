# Step 5 — Data Cleaning and Validation

> **What we build:** `scraper/cleaner.py`. It merges both sources, normalises
> every field, rejects bad rows, removes duplicates, and writes
> `listings_clean.json` + `listings_clean.csv`.
>
> **Why:** *Data Accuracy* is 15% of the grade. Raw scraped data is messy:
> `"Bangalore"` vs `"Bengaluru"`, `"080-2222 3333"` vs `"+91 80 22223333"`,
> `"Pune,, Deccan"`, citation marks, duplicates. If we counted it as-is, the
> charts would be wrong.

---

## 1. The cleaning pipeline

```
raw rows (2,740)
   │  clean_text()       → trim, collapse spaces, remove [1] citations, fix ", ," and " ,"
   │  normalise_city()   → one spelling per city
   │  normalise_phone()  → +91XXXXXXXXXX or None
   │  validate           → reject rows without name / city / category
   │  truncate           → respect MySQL column lengths
   │  de-duplicate       → same name + city + address  → keep the first
   ▼
clean rows (2,733)  →  listings_clean.json / listings_clean.csv
```

Actual run:

```
cleaning stats: {'raw': 2740, 'removed_duplicates': 7, 'clean': 2733,
                 'with_phone': 583, 'with_address': 2510}
by source:   {'OpenStreetMap': 2254, 'Wikipedia': 479}
by city:     {'Mumbai': 434, 'Bengaluru': 415, 'Delhi': 384, 'Chennai': 383,
              'Pune': 362, 'Hyderabad': 346, 'Kolkata': 319, 'Ahmedabad': 90}
```

---

## 2. Rules in detail

### Text cleanup
```python
text = re.sub(r"\[\d+\]", "", text)            # "India[3]"          -> "India"
text = re.sub(r"\s+", " ", text)               # "a   b"             -> "a b"
text = re.sub(r"\s+([,;.)])", r"\1", text)     # "Wadala , Mumbai"   -> "Wadala, Mumbai"
text = re.sub(r"(?:,\s*){2,}", ", ", text)     # "Pune,, Deccan"     -> "Pune, Deccan"
```

### City normalisation
Indian cities have historical and alternate names. A lookup table in
`config.CITY_ALIASES` maps them to **one canonical value**:

| Raw | Canonical |
|---|---|
| Bangalore, bengaluru | **Bengaluru** |
| Bombay | **Mumbai** |
| Madras | **Chennai** |
| Calcutta | **Kolkata** |
| New Delhi | **Delhi** |
| Secunderabad | **Hyderabad** (twin city) |

Without this, `GROUP BY city` would split one city into two bars.

### Phone normalisation (E.164 format)
Phone numbers arrive in many formats. We convert them to the international
**E.164** form `+91XXXXXXXXXX`:

| Raw input | Rule | Output |
|---|---|---|
| `+91 80 2222 3333` | already has country code (12 digits starting 91) | `+918022223333` |
| `080-22223333` | leading 0 + STD code (11 digits) | `+918022223333` |
| `9845012345` | 10-digit mobile (starts 6–9) | `+919845012345` |
| `098450 12345; 9845012346` | several numbers → take the first valid one | `+919845012345` |
| `1800 102 1234` | toll-free, no country code | `18001021234` |
| `12345` | invalid | `None` |

Storing `None` for invalid phones is better than storing garbage. The brief
says phone is *"if available"*.

### Validation
A row without `business_name`, `city` or `category` is **rejected**. These
columns are `NOT NULL` in MySQL and are needed for the charts.

### Length limits
Values are truncated to the MySQL column sizes (name 255, address 500 …), so
the database never rejects a row for being too long.

### De-duplication
```python
def listing_key(row):
    norm = lambda s: re.sub(r"[^a-z0-9]", "", (s or "").lower())
    return norm(row["business_name"]), norm(row["city"]), norm(row["address"])
```
`"Cafe X, 1 MG Road"` and `"cafe  x, 1, MG Road"` give the **same key**, so
only the first one is kept. A `set` makes each lookup O(1).

> Note: chains like *Cafe Coffee Day* have many branches. Because the address is
> part of the key, different branches are kept as separate listings, which is correct.

---

## 3. Output files

| File | Format | Use |
|---|---|---|
| `listings_clean.json` | JSON array | input for the uploader (Step 7) |
| `listings_clean.csv` | CSV, UTF-8 with BOM | open in Excel / Google Sheets for review |

> **Concept — UTF-8 BOM (`utf-8-sig`):** Excel on Windows guesses the encoding
> wrongly unless the file starts with a byte-order mark. With it, names written
> in Indian scripts display correctly.

---

## 4. Concepts used

| Concept | Where |
|---|---|
| Regular expressions | `clean_text`, `normalise_phone`, `listing_key` |
| Dictionary lookup tables | `CITY_ALIASES` |
| `set` for O(1) duplicate checks | `clean()` |
| `collections.Counter` | statistics per source / city / category |
| `csv.DictWriter` | CSV export |
| Pure functions (input → output, no side effects) | easy to unit-test (see [Step 9](09-testing.md)) |

## 5. Run
```bash
python -m scraper.cleaner
```

➡️ Next: [Step 6 — FastAPI backend](06-backend-fastapi.md)
