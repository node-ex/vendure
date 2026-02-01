# Vendure Software Project Report

## 1. Applications

The monorepo contains **3 standalone applications**:

| Application | Location | Language | Framework | Purpose |
|---|---|---|---|---|
| **dev-server** | `packages/dev-server` | TypeScript | NestJS (via @vendure/core), Vite | Private (unpublished) development server for local testing. Bootstraps full Vendure server + worker, configures all plugins, seeds test data. |
| **@vendure/cli** | `packages/cli` | TypeScript | Commander.js, @clack/prompts | Published CLI tool (`vendure` command) for managing Vendure projects -- add plugins, extend functionality. |
| **@vendure/create** | `packages/create` | TypeScript | Commander.js, Handlebars | Project scaffolding tool (`npm create @vendure`) -- generates new Vendure projects from templates. |

- **Runtime**: Node.js >= 18 (root engines), >= 20 enforced by `@vendure/create`. CI tests: 20.x, 22.x, 24.x.
- **Package manager**: npm
- **Version files**: Root `package.json` engines only. No `.nvmrc` or `.node-version`.
- **Dockerfiles**: Template at `packages/create/templates/Dockerfile.hbs` (base: `node:20`) for generated projects. No Dockerfile for the framework itself.

---

## 2. First-Party Libraries

**18 libraries** total (17 in `packages/` + 1 in `docs/`). All share version **3.5.3** via Lerna fixed versioning (except `@vendure/docs` with independent versioning).

### Core Libraries

| Library | Location | Purpose |
|---|---|---|
| **@vendure/common** | `packages/common` | Shared types, utilities, generated GraphQL types. Root dependency for all packages. |
| **@vendure/core** | `packages/core` | Main server framework: NestJS 11 + Apollo GraphQL + TypeORM + Express 5. All e-commerce logic. |
| **@vendure/testing** | `packages/testing` | E2E test utilities: test server bootstrap, GraphQL test client, SQLite-based test DBs. |

### UI Libraries

| Library | Location | Framework | Purpose |
|---|---|---|---|
| **@vendure/admin-ui** | `packages/admin-ui` | Angular 19 + Clarity UI | Legacy admin interface (being replaced). |
| **@vendure/dashboard** | `packages/dashboard` | React 19 + Radix UI + TanStack Router/Query + Vite + Tailwind | New admin dashboard (replacing Angular UI). |
| **@vendure/admin-ui-plugin** | `packages/admin-ui-plugin` | Express | Serves compiled admin UI as static files. |
| **@vendure/ui-devkit** | `packages/ui-devkit` | Angular CLI + Rollup | Tooling for authoring Angular UI extensions. |

### Official Plugins

| Library | Location | Key Dependencies | Purpose |
|---|---|---|---|
| **@vendure/asset-server-plugin** | `packages/asset-server-plugin` | Sharp, AWS S3 SDK | Asset serving, image processing, S3 storage. |
| **@vendure/email-plugin** | `packages/email-plugin` | Nodemailer, Handlebars, MJML | Transactional email with templating. |
| **@vendure/elasticsearch-plugin** | `packages/elasticsearch-plugin` | @elastic/elasticsearch | Product search via Elasticsearch 7.x. |
| **@vendure/job-queue-plugin** | `packages/job-queue-plugin` | BullMQ, ioredis, GCP Pub/Sub (all peers) | Production job queue backends. |
| **@vendure/payments-plugin** | `packages/payments-plugin` | Stripe, Mollie, Braintree (all peers) | Payment provider integrations. |
| **@vendure/graphiql-plugin** | `packages/graphiql-plugin` | React, GraphiQL | Embedded GraphQL IDE. |
| **@vendure/harden-plugin** | `packages/harden-plugin` | graphql-query-complexity | Query complexity/depth limiting. |
| **@vendure/sentry-plugin** | `packages/sentry-plugin` | @sentry/nestjs | Error tracking and profiling. |
| **@vendure/stellate-plugin** | `packages/stellate-plugin` | node-fetch | Stellate GraphQL CDN cache purging. |
| **@vendure/telemetry-plugin** | `packages/telemetry-plugin` | OpenTelemetry SDK | Distributed tracing and structured logging. |

### Other

| Library | Location | Purpose |
|---|---|---|
| **@vendure/docs** | `docs/` | Documentation package (MDX content + auto-generated API reference). Independent versioning, not in Lerna workspace. |

**Dependency hierarchy**: `@vendure/common` -> `@vendure/core` -> all plugins. Most plugins use core/common as devDependencies (peer-like).

---

## 3. Architecture Patterns

### Backend (`@vendure/core`)

- **Architecture**: Three-layer architecture (API -> Service -> Entity/Data)
- **Module organization**: Layer-based (not feature-based). `src/api/` for all resolvers, `src/service/` for all business logic, `src/entity/` for all entities.
- **API design**: Dual GraphQL APIs -- Admin API (`/admin-api`) and Shop API (`/shop-api`). Schema-first with `.graphql` files.
- **Resolver pattern**: 29 admin resolvers, 7 shop resolvers, 26+ entity field resolvers. Custom decorators: `@Allow()` (permissions), `@Transaction()`, `@Ctx()` (RequestContext), `@Relations()`.
- **Service pattern**: 36 domain services + 28 helpers. All methods take `RequestContext` first (carries channel, language, auth, transaction).
- **Entity pattern**: All extend `VendureEntity` (id, createdAt, updatedAt). Patterns: Translatable (i18n), SoftDeletable, ChannelAware (multi-tenancy), HasCustomFields.
- **Error handling**: Dual model -- `I18nError` exceptions for true errors + GraphQL union `ErrorResult` types for business rule violations.
- **Event system**: RxJS-based `EventBus` with 60+ event types. Two modes: non-blocking (post-transaction) and blocking (in-transaction).
- **Plugin system**: `@VendurePlugin()` extends `@Module()` with: GraphQL API extensions, custom entities, config modifications, dashboard extensions. Static `init()` factory pattern.
- **Strategy pattern**: 50+ injectable strategy interfaces covering every extensible behavior (entity IDs, money, assets, auth, pricing, shipping, payments, caching, errors, etc.).
- **State machines**: Orders, fulfillments, payments, refunds use typed FSMs with configurable transitions and guards.
- **Job processing**: Producer/consumer pattern with pluggable backends (in-memory, SQL, BullMQ, Pub/Sub).

### Frontend -- Admin UI (Legacy Angular)

- **Module organization**: NgModule lazy-loading by functional area (catalog, orders, customers, marketing, settings, system).
- **State management**: Apollo Client cache as primary; Angular services for UI state.
- **Data fetching**: `BaseDataService` wrapping Apollo Angular with auto custom-field injection.

### Frontend -- Dashboard (New React)

- **Module organization**: File-based routing via TanStack Router under `routes/_authenticated/`.
- **State management**: TanStack React Query for server state; nested React Context providers for app state.
- **Data fetching**: `awesome-graphql-client` with gql.tada for type-safe GraphQL. Custom hooks: `useExtendedListQuery`, `useExtendedDetailQuery`, `usePaginatedList`.
- **Page architecture**: `ListPage` and `DetailPage` framework components standardizing views.
- **Extension system**: `defineDashboardExtension()` API with Global Registry singleton. Extensions can add routes, nav sections, page blocks, action bar items, widgets, form components, data table columns, etc.
- **Component library**: Radix UI primitives + Tailwind CSS + custom shared components.
- **i18n**: Lingui (21 locales).

---

## 4. Third-Party Systems & Remote Services

| System | Purpose | Packages | Inbound to Vendure? |
|---|---|---|---|
| **Stripe** | Payment processing | payments-plugin | Yes (webhooks at `/payments/stripe`) |
| **Mollie** | Payment processing | payments-plugin | Yes (webhooks at `/payments/mollie/...`) |
| **Braintree** | Payment processing (PayPal) | payments-plugin | No (nonce-based sync flow) |
| **Elasticsearch** | Product search | elasticsearch-plugin | No |
| **AWS S3 / MinIO** | Asset/file storage | asset-server-plugin | No |
| **Redis** | Job queue (BullMQ) + caching | job-queue-plugin, core | No |
| **Google Cloud Pub/Sub** | Job queue | job-queue-plugin | No |
| **SMTP servers** | Email delivery | email-plugin | No |
| **AWS SES** | Email delivery | email-plugin | No |
| **Sentry** | Error tracking / APM | sentry-plugin | No |
| **OpenTelemetry backends** (Jaeger, Loki) | Tracing / logging | telemetry-plugin | No |
| **Stellate** (GraphCDN) | GraphQL edge caching | stellate-plugin | No |
| **Keycloak** | SSO / OIDC auth | dev-server (example only) | No |
| **Google OAuth** | Customer auth | dev-server (example only) | No |

---

## 5. Major Development Tooling

| Category | Tool | Version |
|---|---|---|
| Monorepo | Lerna | v9 |
| Monorepo | npm Workspaces | -- |
| Build | TypeScript | 5.8.2 |
| Build | Vite | 6.3.6 |
| Build (legacy) | Angular CLI + ng-packagr | v19 |
| Build | Rollup | v4 |
| Build | SWC | v1.4.6 |
| CSS | Tailwind CSS | v4.1.5 |
| Testing | Vitest | v3.2.4 |
| Testing (legacy) | Karma + Jasmine | v6.4/v5.6 |
| Testing (browser) | Playwright | v1.55 |
| Linting | ESLint | v8/v9 |
| Formatting | Prettier | v3.2.5 |
| Git hooks | Husky + lint-staged + commitlint | v4.3 / v10.5 / v19.1 |
| Code generation | GraphQL Code Generator | v6.0 |
| Code generation | gql.tada | v1.8 |
| Code generation | TanStack Router Plugin | v1.154 |
| CI/CD | GitHub Actions | 9 workflows |
| Publishing | npm Trusted Publishing (OIDC) | -- |
| Smoke testing | Verdaccio (local npm registry) | -- |
| Component dev | Storybook | v10 beta |
| i18n | Lingui | v5.7 |
| Changelog | conventional-changelog-core | v7.0 |
| Code review | CodeRabbit | -- |
| Deployment | Vercel | -- (dashboard demo) |
| Custom scripts | check-imports, check-core-type-defs, check-angular-versions, check-lib-imports | -- |

---

## 6. Infrastructure

| Category | Name | Technology | Required? |
|---|---|---|---|
| Database | MariaDB / MySQL / PostgreSQL / SQLite | TypeORM multi-driver | One required |
| Cache | In-Memory / SQL / Redis | Pluggable strategy | Built-in default |
| Session | Cookie-based | cookie-session | Built-in |
| Job Queue | In-Memory / SQL / BullMQ+Redis / GCP Pub/Sub | Pluggable strategy | Built-in default |
| Scheduler | DB-based | TypeORM + croner | Optional plugin |
| Search | DB full-text / Elasticsearch | Pluggable strategy | Optional plugin |
| File Storage | Local FS / AWS S3 | Pluggable strategy | Default: local FS |
| Email | SMTP / AWS SES / Sendmail | Nodemailer transports | Optional |
| Runtime | Node.js >= 20 | Express 5 via NestJS | Required |
| Container | Docker | Dockerfile template | Optional |
| Observability | Jaeger / Loki / Grafana | Docker Compose (dev) | Optional |

---

## 7. Folder Structure

```
vendure/
├── packages/                    # All Lerna-managed packages (20 total)
│   ├── core/                    # Main server framework
│   │   └── src/
│   │       ├── api/             # GraphQL resolvers, guards, middleware, schema
│   │       ├── service/         # Business logic services + helpers
│   │       ├── entity/          # TypeORM entities
│   │       ├── event-bus/       # Event system + 60+ event types
│   │       ├── plugin/          # Plugin system + built-in plugins
│   │       ├── config/          # Configuration + strategy interfaces
│   │       ├── connection/      # DB connection wrappers
│   │       ├── job-queue/       # Background job processing
│   │       └── ...              # cache, scheduler, i18n, health-check, worker
│   ├── common/                  # Shared types and utilities
│   ├── testing/                 # E2E test utilities
│   ├── admin-ui/                # Legacy Angular admin (19 feature modules)
│   ├── dashboard/               # New React admin (TanStack Router + Radix UI)
│   │   └── src/
│   │       ├── app/routes/      # File-based routes
│   │       └── lib/             # Components, framework, hooks, graphql
│   ├── admin-ui-plugin/         # Serves admin UI
│   ├── ui-devkit/               # UI extension authoring
│   ├── [9 plugins]/             # Official plugins (asset, email, search, etc.)
│   ├── cli/                     # CLI tool
│   ├── create/                  # Project scaffolding + templates
│   └── dev-server/              # Unpublished dev environment
│       ├── example-plugins/     # Reference implementations
│       └── test-plugins/        # Test-only plugins
├── docs/                        # Standalone documentation project
├── e2e-common/                  # Shared E2E test config (root-level, not a package)
├── scripts/                     # Build/release automation
│   ├── changelogs/              # Changelog generation
│   ├── codegen/                 # GraphQL type generation
│   └── docs/                    # Doc generation scripts
├── license/                     # CLA + license files + signature tracking
├── .github/workflows/           # 9 CI/CD workflows
├── docker-compose.yml           # Dev infrastructure
└── [config files]               # lerna.json, tsconfig, eslint, prettier, etc.
```

**Notable patterns**:

- Layer-based organization in core (API/Service/Entity), not feature-based
- `e2e-common/` at root (anomaly -- shared test config outside `packages/`)
- Dashboard plugin embedded inside `packages/dashboard/plugin/` (unlike `admin-ui-plugin` as separate package)
- `docs/` operates independently from the Lerna workspace
- Three AI agent configs (`.claude/`, `.cursor/`, `.coderabbit.yaml`)
