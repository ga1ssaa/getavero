# AVERO — Finance Dashboard

A deployed full-stack personal finance application for tracking income, expenses, budgets and subscriptions. Built with React and TypeScript, a FastAPI REST API, SQLAlchemy and PostgreSQL.

**[Live Demo](https://getavero.app)** · [API Documentation](https://api.getavero.app/docs) · [CI Workflow](.github/workflows/ci.yml)

## Overview

AVERO brings account-based financial tracking into one workspace. A React frontend calls a versioned FastAPI API; the backend validates requests, enforces data ownership and persists records in PostgreSQL through SQLAlchemy.

The project demonstrates API design, relational modeling, authentication, exact monetary conversion, automated testing, containerization and deployment. Its manually verified production workflow includes registering an account, saving transactions, a budget and subscriptions, then signing in from another device and retrieving the saved data.

## Key Features

- **Financial overview:** income, expenses, balance, category breakdowns, recent transactions and observations based on recorded data. Totals come from backend aggregates rather than the current page of transactions.
- **Transaction management:** create, edit and delete income or expenses; search, filter, sort and paginate records through the API.
- **Budget planning:** manually maintained monthly income, needs, wants and savings, with progress against the 50/30/20 allocation. Budget entries are independent of transaction totals.
- **Subscription tracking:** monthly costs, active/paused status, manually maintained next-payment dates and an active recurring-cost summary. Subscriptions do not automatically create transactions or charge payments.
- **Multi-currency input and display:** USD, EUR, KZT and RUB, with canonical USD storage. Foreign-currency inputs use exact integer/decimal arithmetic and round once to eight USD decimal places using round-half-to-even. Unchanged edit fields preserve their original stored values.
- **Account data controls:** JSON export, validated import previews, confirmed replacement/reset and operation receipts for uncertain outcomes. Destructive actions require the account password and typed confirmation.
- **Responsive interface:** Light/Dark themes, accessible navigation and forms, reduced-motion-aware section reveals, persisted display currency and account-scoped browser-local profile photos.

Financial records and currency preferences are stored on the server. Theme, exchange-rate cache and profile photos remain local to each browser; photos do not synchronize between devices. Non-USD conversion requires fresh, validated quotes from the external exchange-rate provider; USD remains usable without it.

## Technology Stack

| Layer | Technologies |
| --- | --- |
| Frontend | React 19, TypeScript, Vite, React Router, Tailwind CSS, Recharts, Lucide |
| Backend | Python 3.13, FastAPI, Uvicorn, Pydantic 2 |
| Persistence | SQLAlchemy 2.0, psycopg 3, PostgreSQL, Alembic |
| Authentication | Argon2id password hashing, JWT access tokens, rotating refresh tokens |
| Local infrastructure | Docker Compose, multi-stage Docker builds, Nginx |
| Verification | Node.js test runner, pytest, ESLint, TypeScript, npm audit, pip-audit |
| Hosting and CI | Vercel, Heroku, Heroku Postgres, GitHub Actions |

Production uses PostgreSQL 18.4; the isolated local Compose stack uses PostgreSQL 17. Node.js 24 is used by CI and the frontend tests.

## Architecture

```mermaid
flowchart LR
    Browser["User browser"] --> Frontend["React + TypeScript SPA<br/>Vercel"]
    Frontend -->|"HTTPS /api/v1"| API["FastAPI REST API<br/>Heroku"]
    API --> Services["Authentication and financial services"]
    Services --> ORM["SQLAlchemy + psycopg"]
    ORM -->|"Verified TLS"| DB[("PostgreSQL<br/>Heroku Postgres")]
    Frontend -->|"Public USD exchange rates"| Rates["Exchange-rate provider"]
    Migrations["Alembic<br/>Explicit production authorization"] --> DB
```

React pages use shared providers, custom hooks and an API client. FastAPI handlers validate request schemas and define transaction boundaries; services implement owner-scoped queries and mutations. SQLAlchemy models define database relationships and constraints, while Alembic versions the schema.

```text
src/
  api/                 HTTP client, sessions and account data operations
  components/          Shared controls, navigation and modals
  features/            Dashboard, transaction, budget and subscription UI
  hooks/               Resource loading, forms, preferences and theme
  pages/               Routed application and authentication pages
  state/               Authentication and financial providers
  utils/               Money, exchange rates, dates and avatar utilities
backend/
  app/api/             FastAPI routes
  app/services/        Financial, authentication and data-operation services
  app/models/          SQLAlchemy models
  alembic/             Versioned migrations
  tests/               Backend tests
  verification/        Database safety and production preflight tools
tests/                 Frontend regression tests
docker/                Nginx configuration and local-stack tooling
.github/               CI workflow and dependency-audit policy
```

## Authentication and Security

- Registration creates a new user and a separate financial owner namespace. The API derives ownership from the authenticated session, not a client-supplied owner ID; record queries include that owner scope.
- Passwords are hashed with **Argon2id**. Five-minute JWT access tokens stay in frontend memory, and protected requests also check that the server session remains active.
- Refresh tokens rotate on use and are stored as hashes in PostgreSQL. Production refresh cookies are host-only, **HttpOnly**, **Secure** and **SameSite=Lax**. Logout revokes the session.
- Cookie-based authentication POSTs require an explicitly allowed origin and the `X-CSRF-Protection` header. Production uses an exact CORS allowlist for the frontend origin.
- Production authentication rate limits are shared through PostgreSQL and fail closed if the protection cannot operate. Development uses a process-local limiter.
- Production database connections require `sslmode=verify-full`, an approved CA bundle and TLS 1.2 or later. Migrations require separate target confirmation and never run automatically on application startup.

Refresh operations are coordinated within one browser tab. Cross-tab refresh coordination is a remaining limitation; simultaneous refreshes can require signing in again. See the [production runbook](docs/production.md) for deployment, backup and recovery boundaries.

## REST API Overview

All paths below are relative to `/api/v1`. Financial and account-data endpoints require Bearer authentication. Authentication POSTs use the origin/CSRF rules described above.

| Methods | Endpoint | Purpose |
| --- | --- | --- |
| `POST` | `/auth/register`, `/auth/login` | Create an account or start a session |
| `POST` | `/auth/refresh`, `/auth/logout` | Rotate refresh tokens or revoke a session |
| `GET` | `/auth/me` | Retrieve the authenticated user |
| `GET`, `POST` | `/transactions`, `/subscriptions` | List or create records |
| `GET`, `PATCH`, `DELETE` | `/transactions/{id}`, `/subscriptions/{id}` | Retrieve, update or delete an owned record |
| `GET` | `/dashboard`, `/subscriptions/summary` | Financial aggregates and recurring-cost totals |
| `GET`, `PATCH` | `/budget`, `/preferences` | Read or update manual budget inputs and display currency |
| `GET` | `/data/export`, `/data/operations/{key}` | Export account data or retrieve an operation receipt |
| `POST` | `/data/import/preview`, `/data/reset/preview` | Validate and preview an account data operation |
| `POST` | `/data/import`, `/data/reset` | Apply a separately confirmed, idempotent operation |
| `GET` | `/health`, `/health/ready` | Liveness and database/schema readiness |

List endpoints support bounded `limit`/`offset` pagination, date filters and allowlisted sort fields. Transactions additionally support title/category search and type/category filters. Monetary API fields are decimal strings, avoiding JSON floating-point loss.

For request schemas, responses and authentication requirements, use [Swagger UI](https://api.getavero.app/docs) or the [OpenAPI specification](https://api.getavero.app/openapi.json). Local documentation is available at `http://127.0.0.1:8080/docs` in Compose.

## Database Architecture

| Tables | Responsibility |
| --- | --- |
| `users`, `owners` | Login identity and a unique financial namespace for each account |
| `transactions`, `subscriptions` | Owner-scoped financial records and recurring-payment information |
| `budget_profiles`, `preferences` | One manual budget profile and one preference record per owner |
| `auth_sessions`, `auth_refresh_tokens` | Session expiry/revocation and refresh-token rotation |
| `data_operation_receipts` | Owner-scoped, idempotent import/reset results |
| `auth_rate_limits` | Shared production authentication quotas |

UUID identifiers, foreign keys, uniqueness constraints and check constraints protect relationships and valid amounts. Money is stored as PostgreSQL `NUMERIC(20,8)` and handled with Python `Decimal` on the backend. Transactions and subscriptions have owner/date indexes; authentication and quota tables have supporting indexes.

The migration chain is:

```text
0001_infrastructure → 0002_models → 0003_auth
  → 0004_data_operations → 0005_auth_rate_limits
```

Alembic records the active revision in `alembic_version`. The production initial-migration runner verifies the actual target, privileges and schema conflicts, including the protected Heroku namespace. It is separate from the isolated local Compose migration service. See [migration sources](backend/alembic/versions/) and the [production migration procedure](docs/production.md#explicit-new-database-migration-approval).

## Local Development

### Docker Compose

Requires Docker Engine/Desktop with Compose v2 and Python 3 for the local secret generator. From a fresh checkout:

```sh
git clone --branch feat/fullstack https://github.com/ga1ssaa/finance-dashboard.git
cd finance-dashboard
python3 docker/init-env.py
docker compose --env-file .env.compose.local up --build -d --wait
```

Open **http://127.0.0.1:8080** and create a local account. Use this exact host consistently for authentication cookies.

The generator creates an ignored `.env.compose.local` without printing secrets or overwriting an existing file. Compose builds the frontend, serves it with Nginx, proxies `/api/` to FastAPI and applies the migration chain to its dedicated `avero_compose` database. Health checks order database, migration, API and frontend startup. Only Nginx publishes a host port, bound to loopback.

The `avero-local` project uses its own named PostgreSQL volume and does not reuse an existing development or production database. Preserve its environment file when reusing the volume. To stop without deleting data:

```sh
docker compose --env-file .env.compose.local stop
```

Changing `AVERO_HTTP_PORT` requires rebuilding/recreating the stack. Do not use volume-pruning commands or print resolved Compose configuration containing secrets.

### Native frontend and backend

Use Node.js 24 and Python 3.13. Install dependencies from the repository root:

```sh
npm ci
python3.13 -m venv backend/.venv
backend/.venv/bin/python -m pip install -e './backend[dev]'
cp -n .env.example .env.local
```

Follow the [backend setup guide](backend/README.md#configure-the-local-database) to configure a dedicated local PostgreSQL database, private `backend/.env` and explicitly approved local migrations. Once that database is prepared, run the API and frontend in separate terminals:

```sh
backend/.venv/bin/uvicorn --app-dir backend app.main:create_app --factory --host 127.0.0.1 --port 8000 --no-proxy-headers
```

```sh
npm run dev -- --host 127.0.0.1 --port 5173 --strictPort
```

The frontend uses `http://127.0.0.1:8000/api/v1`; local API documentation is at `http://127.0.0.1:8000/docs`. Restart Vite after changing frontend environment variables.

## Environment Variables

Only public build configuration belongs in `VITE_*` variables. Keep signing keys, database credentials and private environment files outside Git.

| Variable | Scope | Purpose / configuration |
| --- | --- | --- |
| `VITE_API_BASE_URL` | Frontend | Local: `http://127.0.0.1:8000/api/v1`; production: `https://api.getavero.app/api/v1` |
| `VITE_APP_ORIGIN` | Frontend | Production browser origin: `https://getavero.app` |
| `AVERO_DB_PASSWORD` | Local Compose | Generated private database password |
| `AVERO_AUTH_JWT_SECRET` | Local Compose | Generated private signing key, passed to the backend |
| `AVERO_HTTP_PORT` | Local Compose | Loopback frontend port; defaults to `8080` |
| `DATABASE_URL` | Backend | Private SQLAlchemy/psycopg connection URL; provider-managed on Heroku |
| `AUTH_JWT_SECRET` | Backend | Private signing key, at least 32 UTF-8 bytes |
| `APP_ENV` | Backend | `development`, `test` or `production` |
| `CORS_ALLOWED_ORIGINS` | Backend | JSON array of exact permitted browser origins |
| `PRODUCTION_SITE_DOMAIN` | Production backend | `getavero.app` |
| `PRODUCTION_API_ORIGIN` | Production backend | `https://api.getavero.app` |
| `TRUSTED_PROXY` | Production backend | `heroku`, enabling the reviewed router-header handling |
| `PRODUCTION_DATABASE_HOST`, `PRODUCTION_DATABASE_NAME` | Production backend | Privately approved target identifiers; never substitute a development database |
| `DATABASE_SSLROOTCERT` | Production backend | Approved CA bundle path; Heroku-24 configuration uses `/usr/lib/ssl/certs/ca-certificates.crt` |
| `PORT`, `DYNO` | Heroku runtime | Supplied by Heroku; not manually configured |

Variable templates: [frontend](.env.example), [local Compose](.env.compose.example), [native backend](backend/.env.example) and [production backend](backend/production.env.example). Native database-container setup also uses `POSTGRES_USER`, `POSTGRES_PASSWORD` and `POSTGRES_DB`; follow the backend guide rather than mixing its environment with the root Compose stack.

## Testing and CI/CD

After installing the native development dependencies, run frontend verification from the repository root:

```sh
RUN_PHASE4_E2E=0 node --test tests/*.test.mjs
npm run lint
npx --no-install tsc -b --pretty false
npm run build
```

Run backend tests offline with an intentionally non-routable synthetic database target:

```sh
APP_ENV=test RUN_DATABASE_TESTS=0 \
  DATABASE_URL='postgresql+psycopg://test:unused@invalid.invalid:1/avero_tests_disabled' \
  backend/.venv/bin/python -m pytest -q backend/tests
```

Database integration tests are opt-in and retain separate target/fixture safeguards. Never point them at production or an existing development database. Offline migration tests generate SQL without executing it. Tests cover API validation, ownership, authentication, monetary conversion/rounding, unchanged-value preservation, data operations, navigation and presentation behavior.

[GitHub Actions](.github/workflows/ci.yml) runs on pushes to `feat/fullstack`, pull requests targeting `feat/fullstack` or `main`, and manual dispatch:

| Job | Checks |
| --- | --- |
| Frontend | Locked npm install, frontend tests, ESLint, TypeScript and production build |
| Backend | Python dependency installation and offline pytest suite |
| Docker | Compose validation, isolation/configuration tests and frontend/backend image builds; no service startup |
| Dependency Security | npm audit, pip-audit and tested audit-report enforcement |

Actions are pinned to immutable commit SHAs with minimal repository permissions. High/critical npm findings and every Python advisory fail the security gate; findings are not automatically suppressed or upgraded. CI uses no production database or deployment credentials. Deployment is manual rather than part of the workflow.

Additional local configuration checks:

```sh
python3 docker/test_config.py
python3 -m unittest discover -s .github/scripts -p 'test_*.py' -q
```

## Deployment Architecture

| Component | Production deployment |
| --- | --- |
| Frontend | Vercel Hobby, Vite static build, SPA routing, `https://getavero.app` |
| Backend | Heroku Basic, FastAPI/Uvicorn, `https://api.getavero.app` |
| Database | Heroku Postgres Essential-0, PostgreSQL 18.4, Alembic revision `0005_auth_rate_limits` |
| CI | GitHub Actions checks before a separately controlled release |

Frontend and API use HTTPS subdomains on the same site. The Heroku Python buildpack installs the backend from the repository root; [Procfile](Procfile) starts `python -m app.production`, which respects Heroku's `PORT` and uses one API worker. The database provides persistent storage independently of dyno filesystems.

Nginx serves the production frontend build **in local Docker Compose**; the live frontend is served by Vercel. Production migrations and backup/recovery procedures are documented in the [deployment runbook](docs/production.md). No migration is performed during web startup.

## Screenshots

Current AVERO screenshots are pending. The existing images in [src/screenshots](src/screenshots/) show the earlier FinanceApp interface and are not presented as the deployed design.

| View | Current screenshot |
| --- | --- |
| Dashboard | Pending capture of the AVERO overview and financial summaries |
| Transactions | Pending capture of the transaction list and filters |
| Budgets | Pending capture of the manual 50/30/20 budget view |
| Subscriptions | Pending capture of recurring costs and payment dates |

## Future Improvements

- Add browser-based end-to-end tests against an isolated environment and synthetic accounts.
- Coordinate refresh-token rotation across browser tabs.
- Add isolated PostgreSQL integration testing to CI alongside the offline suite.
- Introduce a reproducible Python dependency lockfile.
- Replace historical screenshots with current desktop and mobile AVERO captures.
