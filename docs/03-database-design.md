# Step 3 — Database Design (MySQL)

> **What we build:** the `business_listings` database and the `listing_master`
> table (`database/schema.sql`), plus the matching SQLAlchemy model
> (`backend/app/models.py`).
>
> **Why:** MySQL is the single source of truth. The dashboard numbers are
> `GROUP BY` queries on this table, so the schema has to make those queries
> **correct** (no duplicates) and **fast** (indexes).

---

## 1. The schema

```sql
CREATE DATABASE IF NOT EXISTS business_listings
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS listing_master (
    id             INT UNSIGNED  NOT NULL AUTO_INCREMENT,
    business_name  VARCHAR(255)  NOT NULL,
    category       VARCHAR(100)  NOT NULL,
    city           VARCHAR(100)  NOT NULL,
    address        VARCHAR(500)  NULL,
    phone          VARCHAR(30)   NULL,
    source         VARCHAR(50)   NOT NULL,
    listing_hash   CHAR(64)      NOT NULL,
    created_at     TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (id),
    UNIQUE KEY uq_listing_hash (listing_hash),
    KEY idx_city (city),
    KEY idx_category (category),
    KEY idx_source (source)
) ENGINE = InnoDB;
```

It follows the **suggested schema from the PDF exactly**
(`id, business_name, category, city, address, phone, source, created_at`) and
adds one helper column, `listing_hash`, explained below.

---

## 2. Column-by-column decisions

| Column | Type | Why this type |
|---|---|---|
| `id` | `INT UNSIGNED AUTO_INCREMENT` | Surrogate primary key. Unsigned doubles the range, and MySQL assigns it automatically. |
| `business_name` | `VARCHAR(255) NOT NULL` | A listing without a name is useless, so it's required. 255 covers long names. |
| `category` | `VARCHAR(100) NOT NULL` | Used in `GROUP BY`. Short, so it can be indexed fully. |
| `city` | `VARCHAR(100) NOT NULL` | Same reasoning as category. |
| `address` | `VARCHAR(500) NULL` | Addresses can be long. Nullable, because some sources don't give one. |
| `phone` | `VARCHAR(30) NULL` | **String, not number**: phones have `+`, leading zeros and toll-free prefixes. Nullable because the brief says "if available". |
| `source` | `VARCHAR(50) NOT NULL` | Where the row came from (`OpenStreetMap`, `Wikipedia`). Needed for the source-wise chart. |
| `listing_hash` | `CHAR(64) UNIQUE` | SHA-256 hex digest. Always exactly 64 characters, so `CHAR` beats `VARCHAR`. |
| `created_at` | `TIMESTAMP DEFAULT CURRENT_TIMESTAMP` | MySQL fills it in automatically on insert. |

### `VARCHAR` vs `TEXT`
`TEXT` columns can't have a normal index or a default value, and MySQL may store
them off-page. For values with a sensible maximum length, `VARCHAR(n)` is the
better choice.

### `utf8mb4`
MySQL's old `utf8` stores at most 3 bytes per character and **cannot store
emoji and some scripts**. `utf8mb4` is real UTF-8, which matters for Indian
business names written in Hindi, Tamil, Kannada and other scripts.
`utf8mb4_unicode_ci` makes comparisons case-insensitive and accent-aware.

### `InnoDB`
The default MySQL engine. It supports **transactions** (a bulk insert is
all-or-nothing), row-level locking and crash recovery.

---

## 3. Indexes: why and which

The three dashboard queries look like:

```sql
SELECT city, COUNT(id) AS count
FROM listing_master
GROUP BY city
ORDER BY count DESC;
```

Without an index, MySQL scans and sorts the whole table. With `idx_city`, it
reads the already-sorted index and counts as it goes (an *index scan*). This
doesn't matter for 2,000 rows, but it's the correct design and scales to millions.

| Index | Serves |
|---|---|
| `PRIMARY (id)` | row lookup, `COUNT(id)` |
| `idx_city` | `GROUP BY city`, `WHERE city = ?` filters |
| `idx_category` | `GROUP BY category`, `WHERE category = ?` |
| `idx_source` | `GROUP BY source`, `WHERE source = ?` |
| `uq_listing_hash` | fast duplicate checks + **enforces uniqueness** |

You can check that an index is used with `EXPLAIN`:
```sql
EXPLAIN SELECT city, COUNT(*) FROM listing_master GROUP BY city;
-- key: idx_city, Extra: Using index
```

---

## 4. Duplicate prevention: the `listing_hash` idea

**Problem:** the same business can appear twice: scraped twice, present in two
sources, or written slightly differently (`"Cafe Coffee Day"` vs `"cafe  coffee-day"`).
Duplicates would inflate the counts and hurt *Data Accuracy*.

**Naive fix:** `UNIQUE (business_name, city, address)`. This **fails** in MySQL:
with `utf8mb4` each character can take 4 bytes, so 255 + 100 + 500 characters =
3,420 bytes, which is over InnoDB's 3,072-byte index limit. It would also treat
`"Cafe X"` and `"cafe x"` as different.

**Our fix:** the API computes

```python
key  = normalise(name) + "|" + normalise(city) + "|" + normalise(address)
hash = sha256(key).hexdigest()       # 64 hex chars
```

where `normalise` lowercases and removes everything except letters and digits.
The **unique index on the 64-character hash** is small, fast, and catches
"same business, different formatting".

> **Concept — defence in depth:** duplicates are removed in three places: the
> scraper's cleaner, the API's check before insert, and finally the database's
> `UNIQUE` constraint, which guarantees it even if the code has a bug.

---

## 5. The same table in Python (SQLAlchemy ORM)

```python
class Listing(Base):
    __tablename__ = "listing_master"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    business_name: Mapped[str] = mapped_column(String(255))
    category: Mapped[str] = mapped_column(String(100))
    city: Mapped[str] = mapped_column(String(100))
    address: Mapped[str | None] = mapped_column(String(500))
    phone: Mapped[str | None] = mapped_column(String(30))
    source: Mapped[str] = mapped_column(String(50))
    listing_hash: Mapped[str] = mapped_column(CHAR(64), unique=True)
    created_at: Mapped[datetime] = mapped_column(DateTime, server_default=func.now())
```

> **Concept — ORM (Object-Relational Mapper):** a Python class represents a
> table, and an instance represents a row. SQLAlchemy 2.0's typed `Mapped[...]`
> syntax gives editor autocompletion and infers `NULL` / `NOT NULL` from the type:
> `str | None` means nullable.

> **Concept — `server_default=func.now()`:** MySQL itself sets the timestamp,
> not Python, so every row gets the database server's clock.

On startup FastAPI calls `Base.metadata.create_all()`, which **creates the
table only if it doesn't exist**. So the app also works on a fresh MySQL where
`schema.sql` was never run.

---

## 6. Useful SQL for checking the data

```sql
USE business_listings;

SELECT COUNT(*) FROM listing_master;                                   -- total
SELECT city, COUNT(*) c FROM listing_master GROUP BY city ORDER BY c DESC;
SELECT category, COUNT(*) c FROM listing_master GROUP BY category ORDER BY c DESC;
SELECT source, COUNT(*) c FROM listing_master GROUP BY source;
SELECT COUNT(phone) AS with_phone FROM listing_master;                 -- COUNT(col) skips NULLs
SELECT listing_hash, COUNT(*) FROM listing_master GROUP BY listing_hash HAVING COUNT(*) > 1;  -- must be empty
```

✅ **Result of this step:** an empty, indexed, duplicate-proof `listing_master`
table in MySQL.
