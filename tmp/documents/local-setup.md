# Local Development Quick Start

This document describes how to set up the Vendure e-commerce monorepo for local development as quickly as possible.

See [project-setup.md](project-setup.md) for project architecture overview.

## Prerequisites

### Software

| Software | Version | Purpose | Required |
| --- | --- | --- | --- |
| Node.js | `>= 18` (recommend 22.x LTS). Source: `engines` in [package.json](../../package.json). CI tests 20.x, 22.x, 24.x | JavaScript runtime for all build, test, and dev operations | Yes |
| npm | `>= 7` (bundled with Node.js). `lockfileVersion: 3` in [package-lock.json](../../package-lock.json) | Package manager. Lerna is configured to use npm exclusively ([lerna.json](../../lerna.json)) | Yes |
| Git | Any modern version | Version control. Husky git hooks enforce commit linting and lint-staged checks | Yes |
| Docker + Docker Compose | Any modern version | Runs infrastructure services (databases, Elasticsearch, Redis, etc.). Not needed if using SQLite | Yes (unless using SQLite) |
| TypeScript | `5.8.2` (installed automatically via npm). Source: [package.json](../../package.json) | Entire codebase is TypeScript | Automatic |
| Lerna | `^9.0.3` (installed automatically via npm). Source: [package.json](../../package.json) | Monorepo orchestration -- builds, tests, versioning across all packages | Automatic |
| k6 | Latest. See [k6 docs](https://docs.k6.io/) | Load testing scripts in `packages/dev-server/load-testing/` | No |
| Playwright | `^1.55.1`. Source: [dashboard package.json](../../packages/dashboard/package.json) | Dashboard E2E browser testing. Install: `npx playwright install-deps && npx playwright install chromium` | No |
| GitHub CLI (`gh`) | Latest | Convenient fork syncing and PR creation | No |

> **Note:** No `.nvmrc`, `.node-version`, or `.tool-versions` file exists in the repo. Node version management is up to the developer.

### Accesses

| Access | Purpose | How to Obtain |
| --- | --- | --- |
| GitHub account | Fork repo, submit PRs | [github.com](https://github.com) |
| CLA signature | Required before PRs can be merged | Automatically prompted by bot when you open a PR. Full text at [license/CLA.md](../../license/CLA.md) |
| Stripe test API keys | Only for Stripe payment plugin development | Create free test account at [stripe.com](https://stripe.com). Env vars: `STRIPE_APIKEY`, `STRIPE_WEBHOOK_SECRET` |
| Mollie test API key | Only for Mollie payment plugin development | Create account at [mollie.com](https://www.mollie.com). Env var: `MOLLIE_APIKEY` |
| Braintree sandbox | Only for Braintree payment plugin development | [developers.braintreepayments.com](https://developers.braintreepayments.com/start/overview) |
| Sentry DSN | Only for Sentry error tracking plugin development | Create free account at [sentry.io](https://sentry.io/signup/). Env var: `SENTRY_DSN` |
| AWS S3 credentials | Only for S3 asset storage strategy development | [AWS Console](https://aws.amazon.com/s3/) or use MinIO locally |

> **Bottom line:** A new developer needs **nothing more than Node.js >= 20, Docker, and a GitHub account** to start. All external service integrations are optional and only relevant when working on the specific plugin that uses them.

## Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/YOUR-USERNAME/vendure.git
   cd vendure
   # If contributing, add upstream:
   git remote add upstream https://github.com/vendurehq/vendure.git
   ```
   Source: [CONTRIBUTING.md](../../CONTRIBUTING.md) (lines 37-48)

2. **Install dependencies**
   ```bash
   npm install
   ```
   Installs all root-level and workspace-level dependencies across the entire monorepo. Husky git hooks are configured automatically. Warnings about missing platform-specific optional packages are expected and harmless.

3. **Build all packages**
   ```bash
   npm run build
   ```
   Runs `lerna run build` across all packages. The Angular admin-ui build is the slowest part. If you get `Error: Bindings not found.`, run `npm rebuild @swc/core`.
   Source: [CONTRIBUTING.md](../../CONTRIBUTING.md) (lines 107-112)

4. **Start the database**

   Choose one option:

   | Option | Command | Notes |
   | --- | --- | --- |
   | **MariaDB** (default) | `docker-compose up -d mariadb` | No `DB` env var needed |
   | **PostgreSQL 16** | `docker-compose up -d postgres_16` | Use `DB=postgres` for subsequent commands |
   | **SQLite** | (no Docker needed) | Use `DB=sqlite` for subsequent commands |

   > MariaDB and MySQL share port 3306; PostgreSQL options share port 5432. Only run one of each at a time.

5. **Populate test data**
   ```bash
   cd packages/dev-server
   npm run populate              # MariaDB (default)
   # DB=postgres npm run populate  # PostgreSQL
   # DB=sqlite npm run populate    # SQLite
   ```
   Clears all tables, bootstraps Vendure, imports initial data (products, collections, facets, shipping, payment, countries), imports product catalog from CSV, and creates 10 test customers. Source: [populate-dev-server.ts](../../packages/dev-server/populate-dev-server.ts)

6. **Run the development server**
   ```bash
   cd packages/dev-server
   npm run dev
   ```
   Starts both the API server and background worker concurrently via `concurrently npm:dev:server npm:dev:worker`.

## Running the Stack

| Service | Command | Directory | Port(s) | Notes |
| --- | --- | --- | --- | --- |
| **MariaDB** | `docker-compose up -d mariadb` | Root | 3306 | Default database |
| **PostgreSQL 16** | `docker-compose up -d postgres_16` | Root | 5432 | Alternative database |
| **Vendure Server + Worker** | `npm run dev` | `packages/dev-server` | 3000 | Main development command |
| **Vendure Server only** | `npm run dev:server` | `packages/dev-server` | 3000 | Without background worker |
| **Vendure Worker only** | `npm run dev:worker` | `packages/dev-server` | -- | Background job processing |
| **Dashboard Vite Dev** | `npm run dashboard:dev` | `packages/dev-server` | 5173 | React dashboard with HMR |
| **Admin UI Dev** | `npm run dev` | `packages/admin-ui` | 4200 | Legacy Angular admin with live reload |
| **Storybook** | `npm run storybook` | `packages/dashboard` | 6006 | Dashboard component development |
| **Instrumented Server** | `npm run dev:instrumented` | `packages/dev-server` | 3000 | With OpenTelemetry tracing |
| **Elasticsearch** | `docker-compose up -d elasticsearch` | Root | 9200 | Only for elasticsearch-plugin dev |
| **Redis** | `docker-compose up -d redis` | Root | 6379 | Only for BullMQ job queue plugin dev |
| **Keycloak** | `docker-compose up -d keycloak` | Root | 9000 | Only for Keycloak auth testing |
| **Jaeger** | `docker-compose up -d jaeger` | Root | 4318, 16686 | OTLP tracing (used with instrumented server) |
| **Loki + Grafana** | `docker-compose up -d loki grafana` | Root | 3100, 3200 | Log aggregation (used with instrumented server) |

Start order:

1. Database container (MariaDB, PostgreSQL, or skip for SQLite)
2. Populate data (first time only): `npm run populate`
3. Vendure Server + Worker: `npm run dev`
4. (Optional) Dashboard Vite Dev: `npm run dashboard:dev`
5. (Optional) Core/Common Watch: `npm run watch:core-common` from root

## URLs and Ports

| URL | Purpose |
| --- | --- |
| `http://localhost:3000/admin-api` | Admin GraphQL API (+ Playground) |
| `http://localhost:3000/shop-api` | Shop GraphQL API (+ Playground) |
| `http://localhost:3000/graphiql/admin` | GraphiQL IDE -- Admin API |
| `http://localhost:3000/graphiql/shop` | GraphiQL IDE -- Shop API |
| `http://localhost:3000/admin` | Admin UI (legacy Angular, served by AdminUiPlugin) |
| `http://localhost:3000/dashboard` | Dashboard (new React, served by DashboardPlugin) |
| `http://localhost:3000/mailbox` | Email Mailbox (dev mode -- emails written to disk) |
| `http://localhost:3000/assets/...` | Asset server (uploaded images/files) |
| `http://localhost:5173/dashboard/` | Dashboard Vite dev server (HMR, when running `dashboard:dev`) |
| `http://localhost:4200` | Angular Admin UI dev server (when running standalone) |
| `http://localhost:6006` | Storybook (dashboard components) |
| `http://localhost:16686` | Jaeger Web UI (traces, when running Jaeger) |
| `http://localhost:3200` | Grafana Web UI (logs/monitoring, when running Grafana) |
| `http://localhost:9000` | Keycloak Admin Console (when running Keycloak) |

### Port Summary

| Port | Service |
| --- | --- |
| 3000 | Vendure API Server (Admin API, Shop API, Assets, Mailbox, Admin UI, Dashboard) |
| 3100 | Loki (log aggregation) |
| 3200 | Grafana (monitoring) |
| 3306 | MariaDB / MySQL |
| 4200 | Angular Admin UI dev server |
| 4318 | Jaeger OTLP HTTP receiver |
| 5001 | AdminUiPlugin internal Express server |
| 5173 | Vite Dashboard dev server (HMR) |
| 5432 | PostgreSQL |
| 6006 | Storybook |
| 6379 | Redis |
| 9000 | Keycloak |
| 9200 | Elasticsearch |
| 16686 | Jaeger Web UI |

### Example Requests

> **Note:** There is no OpenAPI/Swagger endpoint. Vendure is GraphQL-only. Use GraphiQL at `/graphiql/admin` or `/graphiql/shop` for interactive exploration.

**Login as superadmin (Admin API):**
```bash
curl -X POST http://localhost:3000/admin-api \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation Login($username: String!, $password: String!) { login(username: $username, password: $password) { ... on CurrentUser { id identifier channels { code permissions } } ... on ErrorResult { errorCode message } } }",
    "variables": { "username": "superadmin", "password": "superadmin" }
  }' -v
# Save the vendure-auth-token header from the response
```

**List products (Admin API, authenticated):**
```bash
curl -X POST http://localhost:3000/admin-api \
  -H "Content-Type: application/json" \
  -H "vendure-auth-token: <your-token>" \
  -d '{
    "query": "{ products(options: { take: 10 }) { totalItems items { id name slug variants { id name sku priceWithTax } } } }"
  }'
```

**Search products (Shop API, no auth required):**
```bash
curl -X POST http://localhost:3000/shop-api \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ search(input: { take: 10, groupByProduct: true }) { totalItems items { productName price { ... on PriceRange { min max } } } } }"
  }'
```

**Get collections (Shop API, no auth required):**
```bash
curl -X POST http://localhost:3000/shop-api \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ collections { items { id name slug parent { name } } } }"
  }'
```

**Add item to cart (Shop API, creates anonymous session):**
```bash
curl -X POST http://localhost:3000/shop-api \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation { addItemToOrder(productVariantId: \"1\", quantity: 1) { ... on Order { id code lines { quantity productVariant { name } } } ... on ErrorResult { errorCode message } } }"
  }' -v
# Save the vendure-auth-token header to continue the cart session
```

## Infrastructure Credentials

> **Note:** All credentials below are for local development only.

### Databases

All database services share a consistent credential pattern:

| Field | MariaDB / MySQL | PostgreSQL | SQLite |
| --- | --- | --- | --- |
| Host | `127.0.0.1` | `localhost` | N/A |
| Port | `3306` | `5432` | N/A |
| Username | `vendure` | `vendure` | N/A |
| Password | `password` | `password` | N/A |
| Root Password | `password` | N/A | N/A |
| Database | `vendure-dev` | `vendure-dev` | File: `packages/dev-server/vendure.sqlite` |
| Schema | N/A | `public` | N/A |

Source: [docker-compose.yml](../../docker-compose.yml), [dev-config.ts](../../packages/dev-server/dev-config.ts) (lines 179-223)

### Vendure Application

| Field | Value |
| --- | --- |
| Superadmin username | `superadmin` |
| Superadmin password | `superadmin` |
| Auth token header | `vendure-auth-token` |
| Cookie name | `session` |
| Channel token header | `vendure-token` |
| Cookie secret (dev) | `abc` |

Source: [shared-constants.ts](../../packages/common/src/shared-constants.ts) (lines 5-12)

### Other Services

| Service | Credentials | Port |
| --- | --- | --- |
| Keycloak Admin Console | `admin` / `admin` | 9000 |
| Elasticsearch | No auth required | 9200 |
| Redis | No auth required | 6379 |
| Grafana | Anonymous access enabled (Admin role) | 3200 |
| Loki | No auth required | 3100 |
| Jaeger | No auth required | 16686 |

### E2E Test Database Credentials

| DB Type | Host | Port | Username | Password | Notes |
| --- | --- | --- | --- | --- | --- |
| sql.js (default) | N/A | N/A | N/A | N/A | In-memory, no credentials |
| PostgreSQL | `127.0.0.1` | `5432` (override: `E2E_POSTGRES_PORT`) | `vendure` | `password` | |
| MariaDB | `127.0.0.1` | `3306` (override: `E2E_MARIADB_PORT`) | `root` | `password` | Uses `root`, not `vendure` |
| MySQL | `127.0.0.1` | `3306` (override: `E2E_MYSQL_PORT`) | `root` | `password` | Uses `root`, not `vendure` |

Source: [e2e-common/test-config.ts](../../e2e-common/test-config.ts) (lines 89-123)

## Test Accounts

### Superadmin

| Username | Password | Role | API |
| --- | --- | --- | --- |
| `superadmin` | `superadmin` | SuperAdmin (all permissions) | Admin API |

Source: [shared-constants.ts](../../packages/common/src/shared-constants.ts) (lines 11-12)

### Faker-Generated Customers

All test customers share the password: **`test`**

Generated deterministically using `faker/locale/en_GB` with **seed `1`** ([mock-data.service.ts](../../packages/testing/src/data-population/mock-data.service.ts) line 22). Running `npm run populate` always produces the same 10 customers.

Known customer emails (confirmed from E2E test assertions):

| # | Email | Notes |
| --- | --- | --- |
| 1 | `hayden.zieme12@hotmail.com` | Most commonly used test customer in E2E tests |
| 2 | `trevor_donnelly96@hotmail.com` | Used in order modification and stock tests |
| 3 | `marques.sawayn@hotmail.com` | Used in concurrent checkout tests |
| 4 | `eliezer56@yahoo.com` | Used in customer list filter tests |
| 5-10 | (faker-generated) | Same deterministic pattern from seed `1` |

All customers have `countryCode: GB` addresses, created with `requireVerification: false` (immediately usable).

Source: [populate-customers.ts](../../packages/testing/src/data-population/populate-customers.ts), [mock-data.service.ts](../../packages/testing/src/data-population/mock-data.service.ts)

### Predefined Admin Roles

| Role Code | Description | Key Permissions |
| --- | --- | --- |
| `administrator` | Administrator | Full CRUD on Catalog, Settings, Customer, Order, System |
| `order-manager` | Order manager | CRUD Order + Read Customer, PaymentMethod, ShippingMethod, etc. |
| `inventory-manager` | Inventory manager | CRUD Catalog + CRUD Tag + Read Customer |

Source: [initial-data.ts](../../packages/core/mock-data/data-sources/initial-data.ts) (lines 8-69)

### Keycloak

| Field | Value |
| --- | --- |
| Admin Username | `admin` |
| Admin Password | `admin` |
| Realm | `myrealm` |
| Client ID | `vendure` (public client) |

> The realm and users must be configured manually through the Keycloak admin console at `http://localhost:9000`. Source: [keycloak-auth-plugin.ts](../../packages/dev-server/test-plugins/keycloak-auth/keycloak-auth-plugin.ts)

## Development Commands

See [CLAUDE.md](../../CLAUDE.md) for a quick command reference. See individual `package.json` files for the complete list of scripts per package.

### Essential Commands (from repo root)

| Command | Purpose |
| --- | --- |
| `npm run build` | Build all packages via Lerna |
| `npm run build:core-common` | Build only @vendure/common and @vendure/core |
| `npm run watch:core-common` | Watch-build core + common in parallel (most common during dev) |
| `npm run test` | Run all unit tests across all packages |
| `npm run e2e` | Run all E2E tests across all packages |
| `npm run lint` | ESLint with auto-fix |
| `npm run format` | Prettier with auto-fix |
| `npm run codegen` | Generate TypeScript types from GraphQL schemas |

### Dev Server Commands (from `packages/dev-server`)

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start server + worker (main development command) |
| `npm run dev:server` | Start only the API server |
| `npm run dev:worker` | Start only the background worker |
| `npm run populate` | Seed database with test data |
| `npm run dashboard:dev` | Start Vite dev server for React dashboard (HMR) |
| `npm run dev:instrumented` | Start server + worker with OpenTelemetry |
| `npm run dev:server:sentry` | Start server with Sentry instrumentation |

### Package-Level Commands (from `packages/<name>`)

| Command | Purpose |
| --- | --- |
| `npm run build` | Build the specific package |
| `npm run watch` | Watch-build for live development |
| `npm run test` | Run unit tests (Vitest for most, Karma for admin-ui) |
| `npm run e2e` | Run E2E tests (available in core, asset-server, elasticsearch, payments, graphiql) |
| `npm run e2e <file>` | Run a specific E2E test file |

### Payment Plugin Dev Servers (from `packages/payments-plugin`)

| Command | Purpose |
| --- | --- |
| `npm run dev-server:stripe` | Builds plugin, starts SQLite-backed dev server for Stripe testing |
| `npm run dev-server:mollie` | Builds plugin, starts SQLite-backed dev server for Mollie testing |

### Load Testing (from `packages/dev-server`)

Requires [k6](https://docs.k6.io/) installed and on PATH.

| Command | Purpose |
| --- | --- |
| `npm run load-test:1k` | Populate 1,000 products + run k6 load tests |
| `npm run load-test:10k` | Populate 10,000 products + run k6 load tests |
| `npm run load-test:100k` | Populate 100,000 products + run k6 load tests |

### Typical Workflows

**Editing core:**
```bash
# Terminal 1 (root):        npm run watch:core-common
# Terminal 2 (dev-server):  npm run dev  (restart after changes compile)
```

**Editing a plugin:**
```bash
# Terminal 1 (packages/<plugin>): npm run watch
# Terminal 2 (dev-server):        npm run dev  (restart after changes compile)
```

**Working on the React dashboard:**
```bash
# Terminal 1 (dev-server): npm run dev
# Terminal 2 (dev-server): npm run dashboard:dev  (HMR, no restart needed)
```

**Working on the Angular admin UI:**
```bash
# Terminal 1 (dev-server):  npm run dev
# Terminal 2 (admin-ui):    npm run dev  (live reload at http://localhost:4200)
```

### Debugging

VS Code launch configuration is available at [.vscode/launch.json](../../.vscode/launch.json) with a "Debug Dev Server" configuration that starts the API server with ts-node and attaches the debugger.

For E2E test debugging, set `E2E_DEBUG=true` to increase timeouts to 30 minutes:
```bash
E2E_DEBUG=true npm run e2e some-test.e2e-spec.ts
```

### E2E Test Cache

E2E seed data is cached in `packages/<name>/e2e/__data__/`. Delete this directory to force re-seeding after schema changes.

## Environment Variables

### Database Configuration

| Variable | Description | Default |
| --- | --- | --- |
| `DB` | Database type: `mysql`, `mariadb`, `postgres`, `sqlite`, `sqljs` | `mysql` (dev-server), `sqljs` (E2E tests) |
| `DB_HOST` | PostgreSQL host (only when `DB=postgres`) | `localhost` |
| `DB_PORT` | PostgreSQL port (only when `DB=postgres`) | `5432` |
| `DB_USERNAME` | PostgreSQL username (only when `DB=postgres`) | `vendure` |
| `DB_PASSWORD` | PostgreSQL password (only when `DB=postgres`) | `password` |
| `DB_NAME` | PostgreSQL database name (only when `DB=postgres`) | `vendure-dev` |
| `DB_SCHEMA` | PostgreSQL schema (only when `DB=postgres`) | `public` |

Source: [dev-config.ts](../../packages/dev-server/dev-config.ts) (lines 179-223)

> **Tip:** Create `packages/dev-server/.env` with `DB=postgres` (or `DB=sqlite`) instead of passing the env var each time. The populate and dev scripts both load `.env` automatically via `dotenv/config`.

### Optional Features

| Variable | Description | Default |
| --- | --- | --- |
| `RUN_JOB_QUEUE` | Set to `1` to run job queue in the server process (instead of separate worker) | Not set |
| `IS_INSTRUMENTED` | Set to `true` for OpenTelemetry instrumentation (set automatically by `dev:instrumented` scripts) | Not set |
| `ENABLE_SENTRY` | Set to `true` to enable Sentry plugin (requires `SENTRY_DSN`) | Not set |
| `NODE_ENV` | Standard Node.js environment variable | `development` |

### Sentry Plugin

| Variable | Description | Default |
| --- | --- | --- |
| `SENTRY_DSN` | Sentry Data Source Name URL | None (required when Sentry enabled) |
| `SENTRY_TRACES_SAMPLE_RATE` | Trace sampling rate (0.0-1.0) | Not set (tracing disabled) |
| `SENTRY_PROFILES_SAMPLE_RATE` | Profile sampling rate (0.0-1.0) | Not set (profiling disabled) |
| `SENTRY_ENABLE_LOGS` | Enable sending logs to Sentry (`true`/`false`) | `false` |
| `SENTRY_CAPTURE_LOG_LEVELS` | Comma-separated log levels to capture | `log,warn,error` |

### E2E Testing

| Variable | Description | Default |
| --- | --- | --- |
| `E2E_DEBUG` | Set to `true` for 30-minute test timeouts (for debugging) | Not set (15s local, 30s CI) |
| `CI` | CI environment indicator (adjusts timeouts and port overrides) | Not set |
| `E2E_POSTGRES_PORT` | Override PostgreSQL port in CI | `5432` |
| `E2E_MARIADB_PORT` | Override MariaDB port in CI | `3306` |
| `E2E_MYSQL_PORT` | Override MySQL port in CI | `3306` |

### Payment Plugins (optional)

| Variable | Description |
| --- | --- |
| `STRIPE_APIKEY` | Stripe API secret key |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret |
| `STRIPE_PUBLISHABLE_KEY` | Stripe publishable key |
| `MOLLIE_APIKEY` | Mollie API key (test format: `test_xxxx`) |

### Generated Project Template Variables

These are used in projects scaffolded by `@vendure/create`, not in the monorepo dev-server itself. Source: [.env.hbs](../../packages/create/templates/.env.hbs)

| Variable | Description |
| --- | --- |
| `APP_ENV` | Application environment (`dev`, `production`) |
| `PORT` | HTTP server port |
| `COOKIE_SECRET` | Session cookie signing secret |
| `SUPERADMIN_USERNAME` | Superadmin identifier |
| `SUPERADMIN_PASSWORD` | Superadmin password |

## Troubleshooting

| Problem | Solution |
| --- | --- |
| `Error: Bindings not found.` | Run `npm rebuild @swc/core` to rebuild native SWC bindings |
| Port conflicts | Only run one DB container per port (MariaDB/MySQL share 3306, PostgreSQL options share 5432) |
| Initial build is slow | Normal -- Angular admin-ui build is the bottleneck. Subsequent `watch` builds are faster |
| Changes not reflected | Restart the dev server after package recompilation (`Ctrl+C` + `npm run dev`) |
| Commit rejected | Husky + commitlint enforce conventional commits: `type(scope): Message`. See [CONTRIBUTING.md](../../CONTRIBUTING.md) |
| E2E tests fail after schema change | Delete `packages/<name>/e2e/__data__/` to force re-seeding |
| No `.env` file by default | Create `packages/dev-server/.env` with `DB=postgres` or `DB=sqlite` for persistent DB selection |
| Dashboard vs Admin UI confusion | Angular Admin UI at `/admin` (port 5001 internal). React Dashboard at `/dashboard` (port 3000). Both active simultaneously |
