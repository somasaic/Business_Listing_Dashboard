# Step 10 — Database Dump (`.sql` file)

> **What we build:** `database/listings_dump.sql`, a complete, restorable export
> of the `business_listings` database (schema + all 2,733 rows).
>
> **Why:** submission requirement #3. It lets reviewers load the exact data
> without running the scraper.

---

## 1. Creating the dump

**MySQL in Docker (what we used):**
```bash
docker exec listings-mysql mysqldump -uroot -proot123 \
  --databases business_listings \
  --single-transaction --no-tablespaces --set-gtid-purged=OFF \
  --default-character-set=utf8mb4 > database/listings_dump.sql
```

**Local MySQL install:**
```bash
mysqldump -u root -p --databases business_listings --single-transaction \
  --no-tablespaces --default-character-set=utf8mb4 > database/listings_dump.sql
```

| Flag | Why |
|---|---|
| `--databases business_listings` | includes `CREATE DATABASE` + `USE`, so one command restores everything |
| `--single-transaction` | consistent snapshot of InnoDB tables **without locking** them |
| `--no-tablespaces` | avoids needing the `PROCESS` privilege |
| `--set-gtid-purged=OFF` | no replication metadata, so the dump restores cleanly on any server |
| `--default-character-set=utf8mb4` | Indian-script names survive the round trip |

The dump contains `DROP TABLE IF EXISTS`, `CREATE TABLE listing_master (…)` with
all indexes, and multi-row `INSERT` statements (mysqldump's default
*extended inserts*, which load quickly).

---

## 2. Restoring the dump

```bash
# Docker
docker exec -i listings-mysql mysql -uroot -proot123 < database/listings_dump.sql

# Local MySQL
mysql -u root -p < database/listings_dump.sql

# MySQL Workbench: Server → Data Import → "Import from Self-Contained File"
```

Check:
```sql
SELECT COUNT(*) FROM business_listings.listing_master;   -- 2733
```

**Verified:** we restored the dump into a scratch database (`restore_test`) and
got 2,733 rows with 2,733 unique hashes, then dropped the scratch database.

---

## 3. `schema.sql` vs `listings_dump.sql`

| File | Contains | Use when |
|---|---|---|
| `database/schema.sql` | structure only | starting fresh and loading data with the scraper + API |
| `database/listings_dump.sql` | structure **and** data | reviewing the project quickly with the final dataset |

➡️ Next: [Step 11 — submission and demo video](11-submission-and-demo.md)
