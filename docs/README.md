# Documentation — Step by Step

This folder explains **every step** of the Business Listings Dashboard
assignment: what was built, why, and which Python / FastAPI / SQL / React
concepts were used. Read the steps in order. Each one builds on the previous.

| Step | Topic | Key concepts |
|---|---|---|
| [00](00-assignment-analysis.md) | Understanding the assignment | requirement analysis, architecture, grading weights |
| [01](01-environment-setup.md) | Environment setup | venv, pinned dependencies, Docker MySQL, `.env` config |
| [02](02-project-structure.md) | Project structure | separation of concerns, packages, layered design |
| [03](03-database-design.md) | Database design (MySQL) | data types, utf8mb4, InnoDB, indexes, unique hash, ORM model |
| [04](04-web-scraping.md) | Web scraping | ToS-friendly sources, Overpass QL, MediaWiki API, BeautifulSoup, retries, caching |
| [05](05-data-cleaning.md) | Data cleaning | regex, normalisation, E.164 phones, de-duplication |
| [06](06-backend-fastapi.md) | Backend API (FastAPI) | routers, Pydantic validation, dependency injection, bulk insert, GROUP BY, CORS |
| [07](07-loading-data.md) | Loading data through the API | batching, generators, idempotency |
| [08](08-frontend-react.md) | Frontend (React + Recharts) | components, custom hooks, effects and cleanup, charts, accessibility, dark mode |
| [09](09-testing.md) | Testing | pytest fixtures, TestClient, parametrised tests, E2E checks |
| [10](10-database-dump.md) | Database dump | mysqldump flags, restore |
| [11](11-submission-and-demo.md) | Submission and demo video | checklist, video script, email |

**Final numbers:** 2,733 listings · 8 cities · 13 categories · 2 sources · 583
with phone numbers · 7 required and extra API endpoints · 33 automated tests.
