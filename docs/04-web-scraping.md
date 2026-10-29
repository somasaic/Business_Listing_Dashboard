# Step 4 — Data Collection (Web Scraping)

> **What we build:** two scrapers that together collect **2,740 raw business
> listings** (2,733 after cleaning) across **8 Indian cities** and **13
> categories**, each with name, category, city, address, phone and source.
>
> **Why:** Part 1 of the brief asks for 500+ listings. We collected about 5×
> that from **two sources**, so the *source-wise* chart is meaningful, and we
> did it **without breaking any website's terms of service**.

Files: `scraper/config.py`, `scraper/http_client.py`, `scraper/osm_scraper.py`,
`scraper/wikipedia_scraper.py`

---

## 1. Choosing the sources (the most important decision)

| Candidate | Allowed? | Decision |
|---|---|---|
| Google Maps | ❌ Terms of Service forbid scraping. Heavy JavaScript, CAPTCHAs, IP bans. The official Places API is paid. | Rejected |
| Justdial | ❌ ToS forbids automated access. Aggressive bot protection, phone numbers hidden behind obfuscation. | Rejected |
| Sulekha | ⚠️ Restrictive ToS, dynamic pages. | Rejected |
| **OpenStreetMap** (Overpass API) | ✅ Open data under the **ODbL** license. The Overpass API exists precisely for querying it. | **Source 1** |
| **Wikipedia** (MediaWiki API + HTML) | ✅ Content is **CC BY-SA**. The API is designed for programmatic access. | **Source 2** |

The brief says *"Avoid scraping in a way that violates website terms of service"*,
so picking legal, open sources shows **judgement**, which is worth more than
fighting CAPTCHAs. Both sources are real, crowd-maintained business
directories. OpenStreetMap is the same kind of map data that powers many map apps.

---

## 2. Polite HTTP client (`http_client.py`)

Every request from both scrapers goes through one class:

```python
class PoliteClient:
    def __init__(self, delay=1.0):
        self.session = requests.Session()
        self.session.headers.update({"User-Agent": config.USER_AGENT})

    def request(self, method, url, **kwargs):
        for attempt in range(1, MAX_RETRIES + 1):
            self._wait_turn()                       # keep `delay` seconds between calls
            response = self.session.request(method, url, **kwargs)
            if response.status_code not in {429, 500, 502, 503, 504}:
                response.raise_for_status()
                return response
            time.sleep(2 ** attempt)                # exponential back-off
```

| Feature | Concept | Why |
|---|---|---|
| `requests.Session()` | connection pooling | re-uses the TCP/TLS connection, so it's faster and lighter on the server |
| `User-Agent` header with contact email | scraping etiquette | Wikimedia and OSM **require** clients to identify themselves. The main Overpass server returned *406 Not Acceptable* to our first, anonymous request. |
| fixed delay between calls | rate limiting | never hammer a free community service |
| retry on 429/5xx with `2 ** attempt` sleep | **exponential back-off** | handles temporary overload gracefully instead of crashing |
| `time.monotonic()` | correct timing | unaffected by system clock changes |

---

## 3. Source 1: OpenStreetMap via the Overpass API (`osm_scraper.py`)

### How OSM stores businesses
Each place is a *node* (point), *way* (building outline) or *relation*, with
**tags** (key=value pairs):

```
amenity=hospital   name=Manipal Hospital   addr:street=HAL Airport Road
addr:postcode=560017   phone=+91 80 2502 4444
```

### The query (Overpass QL)
For each city we send **one** query that covers all 12 categories:

```
[out:json][timeout:120];
nwr["amenity"="hospital"]["name"]["addr:street"](12.85,77.50,13.10,77.75);
out tags 30;
nwr["amenity"="restaurant"]["name"]["addr:street"](12.85,77.50,13.10,77.75);
out tags 30;
...
```

| Part | Meaning |
|---|---|
| `nwr` | search **n**odes, **w**ays and **r**elations |
| `["amenity"="hospital"]` | category filter |
| `["name"]` | must have a name |
| `["addr:street"]` | must have a street address. This is a **data-quality filter applied at the source**. |
| `(south,west,north,east)` | the city's bounding box, defined in `config.CITIES` |
| `out tags 30` | return only tags (no geometry) and at most 30 per category, so one huge category doesn't dominate the charts |

The category mapping lives in config, not code:

```python
OSM_CATEGORIES = {
    "Restaurant": ("amenity", "restaurant"),
    "Gym":        ("leisure", "fitness_centre"),
    "Hotel":      ("tourism", "hotel"),
    "Salon":      ("shop", "hairdresser"),
    ...
}
```

### Turning tags into a listing

```python
def element_to_listing(element, city):
    tags = element.get("tags", {})
    return {
        "business_name": tags.get("name:en") or tags["name"],   # prefer English name
        "category": detect_category(tags),
        "city": city,
        "address": build_address(tags),       # joins addr:housenumber, street, suburb, postcode …
        "phone": tags.get("phone") or tags.get("contact:phone") or tags.get("contact:mobile"),
        "source": "OpenStreetMap",
    }
```

`build_address()` joins the `addr:*` parts in a natural order and **skips
repeated parts** (for example `suburb=Pune` and `city=pune`).

### Reliability features
* **Mirror fallback:** three public Overpass servers are tried in order.
* **Response cache:** each city's raw JSON is saved to `scraper/data/raw/`. Re-running
  the pipeline reuses it instead of querying again. This is faster, kinder to the
  server, and gives a reproducible dataset.
* **Walrus operator:** `[row for el in elements if (row := element_to_listing(el, city))]`
  converts and filters in one pass.

**Result:** 2,259 listings (Bengaluru, Mumbai, Delhi, Chennai, Hyderabad, Kolkata, Pune, Ahmedabad).

---

## 4. Source 2: Wikipedia, with real HTML scraping (`wikipedia_scraper.py`)

This scraper shows **classic HTML scraping with BeautifulSoup**.

### Step A — list articles in a category (JSON API)
```python
params = {"action": "query", "list": "categorymembers",
          "cmtitle": "Category:Hospitals in Chennai", "cmtype": "page", "cmlimit": 500}
```
The API **paginates**. While the response contains a `continue` token, we merge
it into the params and request the next page.

### Step B — download each article's HTML
```python
{"action": "parse", "page": title, "prop": "text", "section": 0}
```
`section=0` downloads only the lead section, where the **infobox** lives. That's
about 10× less data than the full page.

### Step C — scrape the infobox with BeautifulSoup
The infobox is the summary table on the right of a Wikipedia article:

```html
<table class="infobox">
  <tr><th>Location</th><td>Adyar<sup>[1]</sup>, Chennai</td></tr>
  <tr><th>Phone</th><td>044 2222 3333</td></tr>
</table>
```

```python
soup = BeautifulSoup(html, "lxml")
infobox = soup.select_one("table.infobox")          # CSS selector
for row in infobox.select("tr"):
    label, value = row.find("th"), row.find("td")
    if label and value:
        fields[clean_text(label).lower()] = clean_text(value)
```

`clean_text()` removes noise **before** extracting text:
* `<sup>` citation markers like `[1]`
* hidden elements (`.noprint`, `.geo-inline-hidden`)
* `get_text(" ")` inserts a space between nodes, which leaves `"Adyar , Chennai"`,
  so a regex removes the space before punctuation

Then `pick(fields, ("address", "location"))` and `pick(fields, ("phone", "telephone", "contact"))`
pick the right rows, because different infobox templates use different labels.

### Other details
* `display_name("Apollo Hospital (Chennai)")` returns `"Apollo Hospital"`. It removes Wikipedia's disambiguation suffix.
* Titles starting with "List of" are skipped, since they are lists, not businesses.
* **Progress cache:** parsed infoboxes are saved after every category, so a crash
  or network drop loses almost nothing.

**Result:** 481 listings (hospitals, schools, hotels, restaurants, shopping malls, banks).

---

## 5. Python concepts used in this step

| Concept | Where |
|---|---|
| Packages and relative imports (`from . import config`) | whole `scraper/` folder |
| `requests.Session`, headers, POST form data | `http_client.py`, Overpass query |
| Exception handling and retry loops | `PoliteClient.request`, mirror fallback |
| f-strings for building queries | `build_query()` |
| Dict / list comprehensions, walrus `:=` | `scrape()` |
| Regular expressions (`re.sub`) | citation removal, name cleanup |
| `pathlib.Path` for OS-independent paths | `config.py` |
| `json` for caching and output | both scrapers |
| `logging` instead of `print` | progress messages with levels |
| `if __name__ == "__main__":` | each module can run on its own |
| Type hints (`list[dict]`, `str | None`) | all functions |

---

## 6. How to run

```bash
python -m scraper.osm_scraper          # ~30 s (cached afterwards)
python -m scraper.wikipedia_scraper    # ~8 min first time (polite 0.5 s delay × ~480 pages)
# or everything at once:
python -m scraper.run_pipeline
```

Output: `scraper/data/osm_listings.json`, `scraper/data/wikipedia_listings.json`.

➡️ Next: [Step 5 — cleaning](05-data-cleaning.md)
