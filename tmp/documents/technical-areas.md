# Vendure Technical Areas Report

## Table of Contents

- [Section 1: General Concerns](#section-1-general-concerns)
  - [G1: Frameworks / Meta Frameworks](#g1-frameworks--meta-frameworks)
  - [G2: Environment Variables / Secrets](#g2-environment-variables--secrets)
  - [G3: Communication](#g3-communication)
  - [G4: Data Validation](#g4-data-validation)
  - [G5: Data Serialization](#g5-data-serialization)
  - [G6: Data Mapping](#g6-data-mapping)
  - [G7: Data Synchronization](#g7-data-synchronization)
  - [G8: Data Backup](#g8-data-backup)
  - [G9: i18n/l10n](#g9-i18nl10n)
  - [G10: User Analytics](#g10-user-analytics)
  - [G11: Error Tracking](#g11-error-tracking)
  - [G12: Error Handling](#g12-error-handling)
  - [G13: Telemetry](#g13-telemetry)
  - [G14: Logging](#g14-logging)
  - [G15: Security](#g15-security)
  - [G16: Notifications](#g16-notifications)
  - [G17: Dependency Management](#g17-dependency-management)
  - [G18: Configuration](#g18-configuration)
  - [G19: Event Management](#g19-event-management)
  - [G20: Concurrency](#g20-concurrency)
  - [G21: Parallelism](#g21-parallelism)
  - [G22: Documentation](#g22-documentation)
  - [G23: Datetime Management](#g23-datetime-management)
  - [G24: File Upload](#g24-file-upload)
  - [G25: File Download](#g25-file-download)
  - [G26: Feature Flags](#g26-feature-flags)
  - [G27: Other Technical Concerns (General)](#g27-other-technical-concerns-general)
- [Section 2: Frontend Concerns](#section-2-frontend-concerns)
  - [F1: Client-Side Routing](#f1-client-side-routing)
  - [F2: Styling Architecture](#f2-styling-architecture)
  - [F3: State Management](#f3-state-management)
  - [F4: Form Management](#f4-form-management)
  - [F5: Fonts](#f5-fonts)
  - [F6: Icon Libraries](#f6-icon-libraries)
  - [F7: Animations/Transitions](#f7-animationstransitions)
  - [F8: Graphics (2D/3D)](#f8-graphics-2d3d)
  - [F9: Image Management](#f9-image-management)
  - [F10: SEO](#f10-seo)
  - [F11: Printing](#f11-printing)
  - [F12: Accessibility (a11y)](#f12-accessibility-a11y)
  - [F13: Other Technical Concerns (Frontend)](#f13-other-technical-concerns-frontend)
- [Section 3: Backend Concerns](#section-3-backend-concerns)
  - [B1: Entrypoints](#b1-entrypoints)
  - [B2: Middleware](#b2-middleware)
  - [B3: Request Management](#b3-request-management)
  - [B4: Request Routing](#b4-request-routing)
  - [B5: Response Management](#b5-response-management)
  - [B6: Public API Management](#b6-public-api-management)
  - [B7: Caching](#b7-caching)
  - [B8: Application Lifecycle](#b8-application-lifecycle)
  - [B9: DX Tools](#b9-dx-tools)
  - [B10: Persistence](#b10-persistence)
  - [B11: Search](#b11-search)
  - [B12: Scheduling](#b12-scheduling)
  - [B13: Queues/Jobs](#b13-queuesjobs)
  - [B14: Multi-tenancy](#b14-multi-tenancy)
  - [B15: Health Checks](#b15-health-checks)
  - [B16: Data Streaming](#b16-data-streaming)
  - [B17: Other Technical Concerns (Backend)](#b17-other-technical-concerns-backend)
- [Summary](#summary)

---

# Section 1: General Concerns

## G1: Frameworks / Meta Frameworks

### G1.1 -- NestJS (Backend Server Framework)

- **Name:** NestJS Backend Framework
- **Purpose:** Provides the core server infrastructure including dependency injection, module system, middleware pipeline, guards, interceptors, and HTTP server.
- **Implementation:** NestJS 11 with `@nestjs/platform-express` (Express 5 adapter) and `@nestjs/apollo` (Apollo GraphQL). Bootstrap via `NestFactory.create()` for the server process and `NestFactory.createApplicationContext()` for the headless worker process. The `@VendurePlugin()` decorator extends `@Module()` for plugin registration. Cookie-based sessions via `cookie-session`.
- **Location:** `packages/core/src/bootstrap.ts`, `packages/core/src/app.module.ts`, `packages/core/src/worker/vendure-worker.ts`, `packages/core/src/plugin/vendure-plugin.ts`

### G1.2 -- Angular 19 (Legacy Admin UI)

- **Name:** Angular Admin UI Framework
- **Purpose:** Provides the legacy admin interface as a single-page application using Angular's module system, routing, and component framework.
- **Implementation:** Angular 19 with `platformBrowserDynamic().bootstrapModule(AppModule)`, Clarity UI design system, NgModule-based lazy-loaded feature modules, Angular Router with `loadChildren()`.
- **Location:** `packages/admin-ui/src/main.ts`, `packages/admin-ui/src/app/app.module.ts`, `packages/admin-ui/src/app/app.routes.ts`

### G1.3 -- React 19 + TanStack + Vite (New Dashboard)

- **Name:** React Dashboard Framework
- **Purpose:** Provides the new admin dashboard as a modern SPA with file-based routing, server state management, and a component library.
- **Implementation:** React 19 with `ReactDOM.createRoot()`, TanStack Router (file-based routing), TanStack React Query for server state, Radix UI for components, Tailwind CSS v4 for styling, Vite 6 as the build tool with HMR. Extensions via `defineDashboardExtension()` API and `virtual:dashboard-extensions` Vite module.
- **Location:** `packages/dashboard/src/app/main.tsx`, `packages/dashboard/src/app/routeTree.gen.ts`, `packages/dashboard/src/app/app-providers.tsx`, `packages/dashboard/src/lib/framework/`

---

## G2: Environment Variables / Secrets

### G2.1 -- VendureConfig Object Pattern (Backend)

- **Name:** VendureConfig-Based Configuration
- **Purpose:** Centralizes all backend configuration in a typed TypeScript configuration object rather than relying on environment variables directly.
- **Implementation:** The `VendureConfig` type definition is the single source of truth. The `ConfigService` (NestJS injectable) wraps `getConfig()`. Environment variables (`process.env`) are consumed only in the dev-server's `dev-config.ts` and a handful of framework-level checks (`NODE_ENV`, `VENDURE_RUNNING_IN_CLI`). The core framework does NOT use dotenv or `@nestjs/config`. The `dotenv/config` package is loaded via `-r dotenv/config` in dev-server npm scripts.
- **Location:** `packages/core/src/config/vendure-config.ts`, `packages/core/src/config/config.service.ts`, `packages/dev-server/dev-config.ts`

### G2.2 -- Dev Server Environment Variables

- **Name:** Dev Server Environment Variables
- **Purpose:** Provides runtime environment configuration for the development server.
- **Implementation:** Direct `process.env` reads in `dev-config.ts` with fallback defaults. Key variables: `DB` (database type), `DB_HOST/PORT/USERNAME/PASSWORD/NAME/SCHEMA`, `RUN_JOB_QUEUE`, `IS_INSTRUMENTED`, `ENABLE_SENTRY`/`SENTRY_DSN`, `STRIPE_APIKEY`/`STRIPE_WEBHOOK_SECRET`. No `.env` file committed; developers create `packages/dev-server/.env`.
- **Location:** `packages/dev-server/dev-config.ts`

### G2.3 -- Vite Build-Time Variables (Dashboard)

- **Name:** Vite Environment Variables
- **Purpose:** Provides build-time configuration for the React dashboard.
- **Implementation:** Uses `import.meta.env.BASE_URL` and `import.meta.env.PROD`. Configuration flows through `virtual:vendure-ui-config` Vite module rather than `VITE_*` custom variables.
- **Location:** `packages/dashboard/src/app/main.tsx`, `packages/dashboard/src/lib/graphql/api.ts`

---

## G3: Communication

### G3.1 -- GraphQL API (Primary Protocol)

- **Name:** GraphQL API Communication
- **Purpose:** Provides the primary communication protocol between all clients and the backend.
- **Implementation:** Dual GraphQL schema (Admin API at `/admin-api`, Shop API at `/shop-api`) via Apollo Server (`@nestjs/apollo`). Schema-first with `.graphql` files. Authentication via `vendure-auth-token` bearer header or cookie sessions. Channel selection via `vendure-token` header.
- **Location:** `packages/core/src/api/config/configure-graphql-module.ts`, `packages/core/src/api/resolvers/`

### G3.2 -- Apollo Angular Client (Legacy Admin UI)

- **Name:** Apollo Angular GraphQL Client
- **Purpose:** Provides GraphQL data fetching for the Angular admin UI.
- **Implementation:** `apollo-angular` wrapping Apollo Client. `BaseDataService` wraps `Apollo.watchQuery()` and `Apollo.mutate()` with automatic custom field injection and read-only field removal. `cache-and-network` fetch policy by default.
- **Location:** `packages/admin-ui/src/lib/core/src/data/providers/base-data.service.ts`, `packages/admin-ui/src/lib/core/src/data/data.module.ts`

### G3.3 -- awesome-graphql-client (New Dashboard)

- **Name:** Dashboard GraphQL Client
- **Purpose:** Provides type-safe GraphQL data fetching for the React dashboard.
- **Implementation:** `awesome-graphql-client` with custom fetch wrapper handling session tokens, channel tokens, and CORS. Type safety via `gql.tada` and `@graphql-typed-document-node/core`.
- **Location:** `packages/dashboard/src/lib/graphql/api.ts`

### G3.4 -- Backend Outbound HTTP

- **Name:** Backend Outbound HTTP Communication
- **Purpose:** Makes HTTP requests from the backend to external services.
- **Implementation:** `node-fetch` for general HTTP calls, Node.js built-in `http`/`https` for asset import and proxy health checks. Payment SDKs (Stripe, Mollie, Braintree), Elasticsearch client, AWS S3 SDK, and Sentry SDK handle their own transport.
- **Location:** `packages/core/src/health-check/http-health-check-strategy.ts`, `packages/core/src/config/asset-import-strategy/default-asset-import-strategy.ts`

### G3.5 -- SimpleGraphQLClient (Testing)

- **Name:** Test GraphQL Client
- **Purpose:** Provides a lightweight GraphQL client for E2E tests.
- **Implementation:** Custom `SimpleGraphQLClient` class in `@vendure/testing` using `node-fetch`, with `query()`, `mutate()`, `asUserWithCredentials()`, and file upload support.
- **Location:** `packages/testing/src/simple-graphql-client.ts`

---

## G4: Data Validation

### G4.1 -- GraphQL Schema Validation (Primary)

- **Name:** GraphQL Schema-Level Input Validation
- **Purpose:** Validates all API inputs against the GraphQL schema type system as the primary validation layer.
- **Implementation:** Schema-first GraphQL with strongly typed input types. Apollo Server validates incoming queries/mutations against the schema automatically. Vendure does NOT use `class-validator`, NestJS validation pipes, or decorator-based validators.
- **Location:** `packages/core/src/api/schema/`

### G4.2 -- Custom Field Configuration Validation

- **Name:** Custom Fields Config Validator
- **Purpose:** Validates developer-provided custom field configurations at bootstrap time.
- **Implementation:** Pure TypeScript validation in `validateCustomFieldsConfig()` asserting valid names, no conflicts, non-nullable defaults, etc. `CustomFieldsValidationSubscriber` (TypeORM subscriber) validates custom field data at runtime on insert/update.
- **Location:** `packages/core/src/entity/validate-custom-fields-config.ts`, `packages/core/src/connection/custom-fields-validation-subscriber.ts`

### G4.3 -- Zod Form Validation (Dashboard)

- **Name:** Zod Schema Validation in Dashboard
- **Purpose:** Provides client-side form validation for the React dashboard.
- **Implementation:** `zod` library to dynamically generate validation schemas from GraphQL document structures and custom field configurations. `createFormSchemaFromFields()` maps GraphQL scalar types to Zod types.
- **Location:** `packages/dashboard/src/lib/framework/form-engine/form-schema-tools.ts`

### G4.4 -- Business Rule Validation via ErrorResult Types

- **Name:** ErrorResult Business Validation
- **Purpose:** Validates business rules and returns structured error objects rather than throwing exceptions.
- **Implementation:** GraphQL union types combining success results with typed `ErrorResult` classes (e.g., `Order | OrderModificationError`). Checked via `isGraphQlErrorResult()` type guard. Separates "true errors" (`I18nError` exceptions) from "expected business rule violations" (ErrorResult union types).
- **Location:** `packages/core/src/common/error/error-result.ts`, `packages/core/src/common/error/generated-graphql-admin-errors.ts`, `packages/core/src/common/error/errors.ts`

---

## G5: Data Serialization

### G5.1 -- GraphQL Type Serialization

- **Name:** GraphQL Schema Serialization
- **Purpose:** Serializes backend entity data into GraphQL-compatible response types.
- **Implementation:** Apollo Server handles serialization from resolver return values to JSON. The `Money` custom scalar (`GraphQLMoney`) serializes monetary values as numbers. Entity field resolvers handle lazy relation resolution.
- **Location:** `packages/core/src/api/config/money-scalar.ts`, `packages/core/src/api/resolvers/entity/`

### G5.2 -- RequestContext Serialization

- **Name:** RequestContext Serialize/Deserialize
- **Purpose:** Enables passing request context across process boundaries (e.g., to job queues).
- **Implementation:** `RequestContext.serialize()` creates JSON-compatible objects via `JSON.parse(JSON.stringify())`. `RequestContext.deserialize()` reconstructs from serialized form, re-instantiating `Channel` entities and converting date strings.
- **Location:** `packages/core/src/api/common/request-context.ts`

### G5.3 -- Job Data Serialization

- **Name:** Job Queue Data Serialization
- **Purpose:** Ensures all data passed to background jobs is JSON-serializable.
- **Implementation:** `Job.ensureDataIsSerializable()` recursively traverses job data handling `Date` -> `toISOString()`, `toJSON()` methods, class instance getters, circular reference detection via `WeakMap`, and max depth limiting (10 levels).
- **Location:** `packages/core/src/job-queue/job.ts`

### G5.4 -- CSV Import Parsing

- **Name:** CSV Product Import Parser
- **Purpose:** Deserializes CSV product data into intermediate representations for bulk import.
- **Implementation:** `csv-parse` library to parse CSV streams into `ParsedProduct` and `ParsedOptionGroup` objects with multi-language translation support.
- **Location:** `packages/core/src/data-import/providers/import-parser/import-parser.ts`

---

## G6: Data Mapping

### G6.1 -- Translation Entity Unwrapping

- **Name:** Translatable Entity Mapping
- **Purpose:** Maps translatable entities with `*Translation` relation arrays into flattened "translated" entities.
- **Implementation:** `translateEntity()` finds the matching translation for a `LanguageCode` (with fallback chain), creates a shallow copy, and copies string properties from the translation. `translateDeep()` handles nested relations (up to 2 levels). `translateTree()` handles recursive tree structures.
- **Location:** `packages/core/src/service/helpers/utils/translate-entity.ts`

### G6.2 -- Input-to-Entity Patching

- **Name:** Input-to-Entity Patch Mapping
- **Purpose:** Maps GraphQL input fields onto existing TypeORM entities for update operations.
- **Implementation:** `patchEntity()` iterates entity keys, copies corresponding input values, skips `undefined`, recursively patches `customFields`, never overwrites `id`.
- **Location:** `packages/core/src/service/helpers/utils/patch-entity.ts`

### G6.3 -- TranslatableSaver

- **Name:** Translatable Entity Saver
- **Purpose:** Maps GraphQL create/update inputs containing `translations[]` into `Translatable` entities with their `*Translation` entities.
- **Implementation:** `TranslatableSaver` service with `create()` and `update()` methods. Computes translation diffs via `TranslationDiffer` (add/update/remove).
- **Location:** `packages/core/src/service/helpers/translatable-saver/translatable-saver.ts`, `packages/core/src/service/helpers/translatable-saver/translation-differ.ts`

### G6.4 -- Custom Field Input Transformation (Angular)

- **Name:** Custom Fields Auto-Injection and Transformation
- **Purpose:** Automatically adds custom field selections to GraphQL queries and transforms custom field inputs in mutations.
- **Implementation:** `addCustomFields()` modifies GraphQL document AST at runtime. `removeReadonlyCustomFields()` strips read-only fields. `transformRelationCustomFieldInputs()` converts relation objects to ID references.
- **Location:** `packages/admin-ui/src/lib/core/src/data/utils/add-custom-fields.ts`, `packages/admin-ui/src/lib/core/src/data/utils/remove-readonly-custom-fields.ts`

---

## G7: Data Synchronization

### G7.1 -- Search Index Synchronization

- **Name:** Search Index Event-Driven Sync
- **Purpose:** Keeps the product search index synchronized with canonical entities in real time.
- **Implementation:** `DefaultSearchPlugin` subscribes to 8 event types via `EventBus.ofType()`. Each event dispatches a typed job to the `update-search-index` job queue. `IndexerController` processes jobs, updating the `SearchIndexItem` table. `CollectionModificationEvent` events are debounced (50ms). Optional `JobBuffer` de-duplication.
- **Location:** `packages/core/src/plugin/default-search-plugin/`

### G7.2 -- Elasticsearch Index Synchronization

- **Name:** Elasticsearch Plugin Sync
- **Purpose:** Alternative search that synchronizes product data to an Elasticsearch 7.x index.
- **Implementation:** Same event-driven pattern. Uses `@elastic/elasticsearch` client with `client.bulk()` for batch operations. Supports custom field mappings and script fields.
- **Location:** `packages/elasticsearch-plugin/src/`

### G7.3 -- Stellate CDN Cache Purging

- **Name:** Stellate CDN Cache Sync
- **Purpose:** Synchronizes the Stellate GraphQL CDN cache by purging stale entries on entity changes.
- **Implementation:** Event-driven `PurgeRule` classes with RxJS buffering/debouncing. HTTP POST to Stellate Purging API. 8 built-in rules.
- **Location:** `packages/stellate-plugin/src/`

---

## G8: Data Backup

**Not identified.** Vendure does not include built-in data backup, database dump, or data export functionality. No export/dump commands in the CLI tool. Database backup is expected at the infrastructure level (e.g., `pg_dump`, `mysqldump`, cloud provider tools).

---

## G9: i18n/l10n

### G9.1 -- Backend i18n (i18next + ICU)

- **Name:** Backend Server-Side Internationalization
- **Purpose:** Translates error messages and business rule violation messages based on the client's preferred language.
- **Implementation:** `i18next` with `i18next-http-middleware` (detects language from `languageCode` query parameter), `i18next-fs-backend` (JSON files from disk), and `i18next-icu` (ICU message format). All server errors extend `I18nError` with i18n message keys. Preloaded locales: en, de, ru, uk, fr, es, pt_BR, pt_PT.
- **Location:** `packages/core/src/i18n/i18n.service.ts`, `packages/core/src/i18n/i18n-error.ts`, `packages/core/src/i18n/messages/`

### G9.2 -- Backend Entity Translations (Translatable Pattern)

- **Name:** Translatable Entity Localization
- **Purpose:** Supports multi-language content for e-commerce entities stored as separate translation rows.
- **Implementation:** `Translatable` interface with `translations` relation to `*Translation` entities. `LocaleString` opaque type. `TranslatableSaver` for CRUD with translation diffing. `translateEntity()`/`translateDeep()` unwrap for requested `LanguageCode`.
- **Location:** `packages/core/src/common/types/locale-types.ts`, `packages/core/src/service/helpers/translatable-saver/`, `packages/core/src/service/helpers/utils/translate-entity.ts`

### G9.3 -- Angular Admin UI i18n (ngx-translate)

- **Name:** Angular Admin UI Internationalization
- **Purpose:** UI text translation for the legacy admin interface.
- **Implementation:** `@ngx-translate/core` with custom HTTP loader. `I18nService` wraps `TranslateService`. `_()` marker function for extractable keys. Synchronous via `ngxTranslate.instant()`.
- **Location:** `packages/admin-ui/src/lib/core/src/providers/i18n/i18n.service.ts`

### G9.4 -- React Dashboard i18n (Lingui)

- **Name:** Dashboard Internationalization
- **Purpose:** UI text translation for the new React dashboard with 25 locales.
- **Implementation:** `@lingui/core` and `@lingui/react` with `.po` message catalogs. Dynamic locale loading via `dynamicActivate()`. Custom Vite plugin for translation compilation. RTL support via Radix `DirectionProvider`. 25 locales: ar, bg, cs, de, en, es, fa, fr, he, hr, it, ja, ko, nb, ne, nl, pl, pt_BR, pt_PT, ru, sv, tr, uk, zh_Hans, zh_Hant.
- **Location:** `packages/dashboard/src/lib/providers/i18n-provider.tsx`, `packages/dashboard/src/i18n/locales/`, `packages/dashboard/vite/vite-plugin-translations.ts`

---

## G10: User Analytics

**Not identified.** Vendure is a headless e-commerce framework, not a consumer-facing application. No analytics libraries (Google Analytics, PostHog, Mixpanel, etc.) are present. The storefront where analytics would be implemented is built separately by the user.

---

## G11: Error Tracking

- **Name:** Sentry Error Tracking Plugin
- **Purpose:** Captures unhandled exceptions, GraphQL errors, and worker errors and sends them to Sentry via the pluggable `ErrorHandlerStrategy` interface.
- **Implementation:** `@sentry/node` + `@sentry/nestjs` in `@vendure/sentry-plugin`. Core abstraction: `ErrorHandlerStrategy` interface in `@vendure/core` with `handleServerError()` and `handleWorkerError()` hooks. Preload pattern: `node --import @vendure/sentry-plugin/instrument`. `ExceptionLoggerFilter` iterates all registered `errorHandlers`. `SentryErrorHandlerStrategy` enriches context with GraphQL field names/variables. Log-level filtering: only errors ≤ Warn are reported. Optional `createTestError` mutation for verification.
- **Location:** `packages/core/src/config/system/error-handler-strategy.ts` (interface), `packages/core/src/api/middleware/exception-logger.filter.ts` (caller), `packages/sentry-plugin/src/` (plugin)

---

## G12: Error Handling

- **Name:** Dual-Model Error Handling (Exceptions + GraphQL ErrorResults)
- **Purpose:** Two-layer error handling: `I18nError` exceptions for true server errors caught by NestJS filters, and GraphQL union `ErrorResult` types for business rule violations.
- **Implementation:**
  - **Layer 1 -- I18nError Hierarchy:** `I18nError` extends `GraphQLError` with i18n key, variables, code, and `logLevel`. Concrete types: `InternalServerError`, `UserInputError`, `IllegalOperationError`, `UnauthorizedError`, `ForbiddenError`, `EntityNotFoundError`. `ExceptionLoggerFilter` (`@Catch()`) routes log output by level, dispatches to `ErrorHandlerStrategy` instances.
  - **Layer 2 -- ErrorResult Unions:** Mutations return unions like `Order | OrderModificationError`. `isGraphQlErrorResult()` type guard. Error classes have `errorCode` and `message`.
  - **Frontend:** React dashboard uses TanStack Router's `errorComponent` per route. Angular uses `NotificationService` for error toasts.
- **Location:** `packages/core/src/i18n/i18n-error.ts`, `packages/core/src/common/error/errors.ts`, `packages/core/src/common/error/error-result.ts`, `packages/core/src/api/middleware/exception-logger.filter.ts`, `packages/dashboard/src/lib/components/shared/error-page.tsx`

---

## G13: Telemetry

- **Name:** OpenTelemetry Distributed Tracing and Structured Logging
- **Purpose:** Opt-in distributed tracing (spans) and structured log export to OTLP-compatible backends via the pluggable `InstrumentationStrategy` and `@Instrument()` decorator.
- **Implementation:** `InstrumentationStrategy` interface with `wrapMethod()`. All 36+ core services decorated with `@Instrument()` which wraps every method via JavaScript `Proxy` (no-op without `VENDURE_ENABLE_INSTRUMENTATION` env var). `OtelInstrumentationStrategy` creates spans via `tracer.startActiveSpan()`. SDK setup: `BatchSpanProcessor` + `OTLPTraceExporter`, `BatchLogRecordProcessor` + `OTLPLogExporter`. Preload via `node --require ./instrumentation.js`.
- **Location:** `packages/core/src/common/instrument-decorator.ts`, `packages/core/src/config/system/instrumentation-strategy.ts`, `packages/telemetry-plugin/src/`

---

## G14: Logging

- **Name:** Pluggable VendureLogger System
- **Purpose:** Centralized, pluggable logging with configurable log levels, a static `Logger` class facade, and adapter implementations.
- **Implementation:** `Logger` static class delegates to a `VendureLogger` instance. 5 levels: Error, Warn, Info, Verbose, Debug. Implementations: `DefaultLogger` (console with `picocolors` coloring), `NoopLogger` (testing), `TypeOrmLogger` (TypeORM adapter), `OtelLogger` (OTLP log records). Configured via `VendureConfig.logger`.
- **Location:** `packages/core/src/config/logger/vendure-logger.ts`, `packages/core/src/config/logger/default-logger.ts`, `packages/core/src/config/logger/typeorm-logger.ts`, `packages/telemetry-plugin/src/config/otel-logger.ts`

---

## G15: Security

- **Name:** Multi-Layered Security System
- **Purpose:** Authentication, authorization, session management, password hashing, CORS, and query complexity limiting.
- **Implementation:**
  - **Authentication:** Pluggable `AuthenticationStrategy<Data>` interface. Built-in: `NativeAuthenticationStrategy` (bcrypt). External examples: Keycloak OIDC, Google OAuth. Dual strategy arrays for Shop/Admin APIs.
  - **Session:** Cookie-based (`cookie-session`) and/or bearer token. Configurable `tokenMethod`. Session caching via `SessionCacheStrategy`.
  - **Authorization:** `@Allow(Permission.*)` decorator on resolvers. `AuthGuard` (NestJS `CanActivate`) validates session, builds `RequestContext`, checks permissions (OR logic). Granular RBAC with CRUD per entity type + custom permissions.
  - **Password:** Pluggable `PasswordHashingStrategy` (default: bcrypt, 12 rounds). `PasswordValidationStrategy` for complexity rules.
  - **Network:** Configurable CORS via `apiOptions.cors`. `trustProxy` for reverse proxy.
  - **Hardening (`@vendure/harden-plugin`):** `QueryComplexityPlugin` (graphql-query-complexity, max 1000, Shop API only). `HideValidationErrorsPlugin` removes field suggestions. Introspection/playground/debug toggle.
- **Location:** `packages/core/src/api/middleware/auth-guard.ts`, `packages/core/src/config/auth/`, `packages/core/src/api/decorators/allow.decorator.ts`, `packages/harden-plugin/src/`

---

## G16: Notifications

### G16.1 -- Email Notifications (Backend)

- **Name:** Event-Driven Transactional Email System
- **Purpose:** Delivers transactional emails triggered by business events with templated content.
- **Implementation:** `EmailEventListener` DSL subscribing to `EventBus` events. MJML + Handlebars templating. Transport via Nodemailer: SMTP, AWS SES, Sendmail, File (dev), Testing, Noop. Async job queue processing with retries. Dev mailbox viewer.
- **Location:** `packages/email-plugin/src/`

### G16.2 -- Frontend Toast Notifications

- **Name:** In-App Toast Notifications
- **Purpose:** Displays success/error/info messages to admin users.
- **Implementation:** React dashboard: `sonner` toast library. Angular admin: custom `NotificationService` with dynamic `NotificationComponent` creation, configurable duration, stacking.
- **Location:** `packages/dashboard/src/lib/components/ui/sonner.tsx`, `packages/admin-ui/src/lib/core/src/providers/notification/notification.service.ts`

### G16.3 -- Dashboard Alert System

- **Name:** Dashboard Alert Indicators
- **Purpose:** Displays persistent alert indicators for system conditions requiring attention.
- **Implementation:** `useAlerts()` hook backed by `AlertsContext` with `DashboardAlertDefinition` objects (shouldShow, severity, auto-refresh). `AlertsIndicator` shows severity-colored badge.
- **Location:** `packages/dashboard/src/lib/hooks/use-alerts.ts`, `packages/dashboard/src/lib/framework/alert/`

---

## G17: Dependency Management

- **Name:** NestJS Dependency Injection + Vendure Injector Bridge
- **Purpose:** NestJS DI container for all service wiring, supplemented by a custom `Injector` class bridging DI access for strategy objects outside the module system.
- **Implementation:** Standard NestJS `@Injectable()` / `@Module()` patterns. `Injector` class wraps NestJS `ModuleRef` with `get<T>()` and `resolve<T>()` (strict: false for cross-module access). `InjectableStrategy` interface with `init(injector)` and `destroy()` lifecycle hooks allows 50+ strategy implementations to access DI-managed services. `@VendurePlugin()` extends `@Module()` with plugin-specific metadata. Static `init()` factory pattern for plugin configuration.
- **Location:** `packages/core/src/common/injector.ts`, `packages/core/src/common/types/injectable-strategy.ts`, `packages/core/src/plugin/vendure-plugin.ts`, `packages/core/src/config/config.service.ts`

---

## G18: Configuration

- **Name:** VendureConfig Centralized Configuration System
- **Purpose:** Single typed configuration object governing all server behavior, merged with defaults at bootstrap, accessible via injectable `ConfigService`.
- **Implementation:** `VendureConfig` interface with ~16 option groups (`apiOptions`, `authOptions`, `catalogOptions`, `customFields`, `dbConnectionOptions`, `entityOptions`, `orderOptions`, `paymentOptions`, `plugins`, etc.). `RuntimeVendureConfig` makes all properties required. Module-level singleton managed by `getConfig()`/`setConfig()`. At bootstrap, user config is deep-merged with `defaultConfig` via `mergeConfig()`, then plugin `configuration` callbacks modify it sequentially. Each option group contains strategy interfaces extending `InjectableStrategy`.
- **Location:** `packages/core/src/config/vendure-config.ts`, `packages/core/src/config/default-config.ts`, `packages/core/src/config/config-helpers.ts`, `packages/core/src/config/merge-config.ts`, `packages/core/src/config/config.service.ts`

---

## G19: Event Management

- **Name:** RxJS-based EventBus with Blocking and Non-Blocking Handlers
- **Purpose:** Globally-available in-process event system decoupling core operations from side effects, supporting both post-transaction and in-transaction handlers.
- **Implementation:** RxJS `Subject<VendureEvent>`. Non-blocking: `eventBus.ofType(EventType)` returns Observable, defers delivery until transaction commits via `TransactionSubscriber`. Events from rolled-back transactions are dropped. Blocking: `registerBlockingEventHandler()` executes in-transaction with topological ordering by `before`/`after` dependencies. 60+ event types. Also: TypeORM `EntitySubscriberInterface` for DB lifecycle events, Email `EmailEventListener` DSL, FSM transition hooks.
- **Location:** `packages/core/src/event-bus/event-bus.ts`, `packages/core/src/event-bus/events/` (60+ files), `packages/core/src/connection/transaction-subscriber.ts`

---

## G20: Concurrency

- **Name:** Transaction-Based Concurrency with Pessimistic Locking and Deadlock Retry
- **Purpose:** Ensures data consistency through database transactions, pessimistic row locking, and automatic deadlock retry.
- **Implementation:** `@Transaction()` decorator wraps resolvers in TypeORM transactions. Configurable isolation levels. Pessimistic `FOR UPDATE` locking in `SqlJobQueueStrategy` (job claiming) and `DefaultSchedulerStrategy` (task locking). Deadlock retry: 5 attempts for MySQL `ER_LOCK_DEADLOCK` and PostgreSQL `deadlock_detected`. SQLite transaction start retry: 25 attempts with progressive delay. `AsyncQueue` for in-process concurrency limiting. Entity read retry via `GetEntityOrThrowOptions.retries`. No optimistic locking (`@VersionColumn`) used.
- **Location:** `packages/core/src/api/decorators/transaction.decorator.ts`, `packages/core/src/connection/transaction-wrapper.ts`, `packages/core/src/plugin/default-job-queue-plugin/sql-job-queue-strategy.ts`, `packages/core/src/common/async-queue.ts`

---

## G21: Parallelism

- **Name:** Process-Level Worker Separation with Configurable Job Concurrency
- **Purpose:** Offloads background tasks to a separate worker process with configurable concurrent job processing.
- **Implementation:** Separate OS process via `bootstrapWorker()` (NestFactory.createApplicationContext, no HTTP). `ProcessContext` tracks 'server' vs 'worker'. No `worker_threads`, `cluster`, or `child_process.fork()`. Job queue concurrency: `PollingJobQueueStrategy` (default 1), `BullMQJobQueueStrategy` (default 3), `PubSubJobQueueStrategy` (maxMessages). Build-time parallelism via `concurrently` npm package and Lerna.
- **Location:** `packages/core/src/worker/vendure-worker.ts`, `packages/core/src/bootstrap.ts`, `packages/core/src/process-context/process-context.ts`, `packages/core/src/job-queue/polling-job-queue-strategy.ts`

---

## G22: Documentation

- **Name:** Custom TypeScript Compiler API-Based Documentation Generation
- **Purpose:** Generates API reference documentation from TypeScript source annotations and GraphQL schema introspection.
- **Implementation:** JSDoc-style comments with custom tags (`@docsCategory`, `@docsPage`, `@docsWeight`, `@since`, `@internal`). Custom `TypescriptDocsParser` uses TypeScript Compiler API (`ts.createProgram`). `TypescriptDocsRenderer` renders to markdown. `generate-graphql-docs.ts` uses GraphQL introspection. Standalone `docs/` directory independent from Lerna workspace. No Swagger/OpenAPI or TypeDoc.
- **Location:** `scripts/docs/generate-typescript-docs.ts`, `scripts/docs/typescript-docs-parser.ts`, `scripts/docs/generate-graphql-docs.ts`, `docs/`

---

## G23: Datetime Management

- **Name:** Multi-Library Datetime Handling with Intl API Formatting
- **Purpose:** Handles date/time representation, formatting, manipulation, and localized display across backend and both frontends.
- **Implementation:**
  - **Backend:** `GraphQLDateTime` custom scalar from `graphql-scalars`. TypeORM `@CreateDateColumn()`/`@UpdateDateColumn()`. `cron-time-generator` + `croner` for scheduling.
  - **Angular Admin:** `dayjs` for manipulation (calendar, relative time, date ranges). `Intl.DateTimeFormat` via `LocaleDatePipe` for display.
  - **React Dashboard:** `date-fns` v4 for manipulation. `useLocalFormat()` hook wrapping `Intl.DateTimeFormat` and `Intl.RelativeTimeFormat`. `react-day-picker` for calendar UI.
  - No dedicated timezone library.
- **Location:** `packages/core/src/api/config/generate-resolvers.ts`, `packages/admin-ui/src/lib/core/src/shared/pipes/locale-date.pipe.ts`, `packages/dashboard/src/lib/hooks/use-local-format.ts`

---

## G24: File Upload

- **Name:** GraphQL Upload Scalar with Express Middleware
- **Purpose:** Enables file uploads through GraphQL using multipart form data.
- **Implementation:** `graphql-upload` v17 provides Express middleware and GraphQL scalar. Registered in `ApiModule.configure()` as `graphqlUploadExpress({ maxFileSize })`. `Upload` scalar in `common-types.graphql`. Two mutations: `createAssets(input: [CreateAssetInput!]!)` and `importProducts(csvFile: Upload!)`. `AssetService.create()` validates MIME types, generates previews, stores via `AssetStorageStrategy`. No `multer`.
- **Location:** `packages/core/src/api/api.module.ts`, `packages/core/src/api/schema/common/common-types.graphql`, `packages/core/src/service/services/asset.service.ts`

---

## G25: File Download

- **Name:** Express Router-Based Asset Serving with Image Transformation
- **Purpose:** Serves uploaded assets over HTTP with on-the-fly image transformation, caching, and pluggable storage.
- **Implementation:** `AssetServerPlugin` registers Express Router via NestJS middleware. `Sharp` library for resize, crop (focal point), format conversion (JPEG/PNG/WebP/AVIF), quality. URL query params: `?w=&h=&mode=crop&fpx=&fpy=&format=&q=`. Named presets: tiny, thumb, small, medium, large. Transform caching in `cache/` directory. Pluggable storage: `LocalAssetStorageStrategy` or `S3AssetStorageStrategy`. `AdminUiPlugin` and `DashboardPlugin` serve compiled UIs via `express.static()`.
- **Location:** `packages/asset-server-plugin/src/`, `packages/admin-ui-plugin/src/plugin.ts`, `packages/dashboard/plugin/dashboard.plugin.ts`

---

## G26: Feature Flags

**Not identified.** No feature flag system, feature toggle library, or A/B testing mechanism. The codebase uses plugin-based extensibility (enable/disable entire plugins) and strategy-based configuration rather than runtime feature flags.

---

## G27: Other Technical Concerns (General)

### G27.1: Testing Infrastructure

- **Name:** Vitest-Based Testing with Custom E2E Framework
- **Purpose:** Comprehensive testing for unit tests, E2E tests, and browser tests.
- **Implementation:** Vitest v3.2.4 for unit and E2E tests, `unplugin-swc` for decorator support. Legacy Angular uses Karma + Jasmine. `@vendure/testing` package provides `TestServer`, `createTestEnvironment()`, `SimpleGraphQLClient`, per-DB initializers, `MockDataService` (deterministic faker seed), `ErrorResultGuard`. E2E data caching in `__data__/`. Playwright v1.55 for dashboard browser tests. 4 database backends in CI (sqljs, MariaDB, MySQL, PostgreSQL).
- **Location:** `e2e-common/vitest.config.mts`, `e2e-common/test-config.ts`, `packages/testing/src/`

### G27.2: Build / Bundling

- **Name:** Multi-Tool Build System
- **Purpose:** Builds diverse package types using appropriate tools per technology.
- **Implementation:** TypeScript 5.8.2 for type checking. Vite 6.3.6 for React dashboard. SWC 1.4.6 for fast transpilation. Angular CLI + ng-packagr v19 for Angular admin. Rollup v4 for GraphiQL plugin. Lerna v9 for monorepo orchestration. Tailwind CSS v4.1.5. Storybook v10 beta.
- **Location:** Root `package.json`, per-package configs, `packages/dashboard/vite/`

### G27.3: CI/CD

- **Name:** GitHub Actions with Multi-DB Multi-Node Matrix
- **Purpose:** Automated build verification, testing, publishing, and deployment.
- **Implementation:** 9 workflows including `build_and_test.yml` (primary), `publish_to_npm.yml` (OIDC trusted publishing), `deploy_dashboard.yml` (Vercel), `publish_and_install.yml` (Verdaccio smoke test). Build matrix: Node.js 20.x, 22.x, 24.x. E2E matrix: 4 databases x 3 Node versions. `cancel-in-progress: true`.
- **Location:** `.github/workflows/`

### G27.4: ID Generation

- **Name:** Pluggable Entity ID Strategy
- **Purpose:** Configures how all entity primary keys are generated, stored, and exposed.
- **Implementation:** `EntityIdStrategy<T>` interface with `encodeId()` and `decodeId()`. Built-in: `AutoIncrementIdStrategy` (default), `UuidIdStrategy`, `Base64IdStrategy`. `IdInterceptor` transforms IDs at the API boundary.
- **Location:** `packages/core/src/config/entity/entity-id-strategy.ts`, `packages/core/src/config/entity/auto-increment-id-strategy.ts`, `packages/core/src/config/entity/uuid-id-strategy.ts`

### G27.5: Pagination

- **Name:** Offset-Based Pagination via ListQueryBuilder
- **Purpose:** Standardized paginated list queries with auto-generated filters and sorts.
- **Implementation:** Offset-based (`skip` + `take`), not cursor-based. All lists return `PaginatedList<T>` with `{ items, totalItems }`. `ListQueryBuilder` builds TypeORM queries from GraphQL `ListQueryOptions`. Auto-generated `*ListOptions`, `*FilterParameter`, `*SortParameter` types. Configurable `shopListQueryLimit` / `adminListQueryLimit`.
- **Location:** `packages/core/src/service/helpers/list-query-builder/list-query-builder.ts`

### G27.6: Code Quality

- **Name:** ESLint + Prettier + Husky + Commitlint Pipeline
- **Purpose:** Enforces consistent code style and conventional commits.
- **Implementation:** ESLint v8 with `@typescript-eslint`, Dashboard uses ESLint v9. Prettier v3.2.5 with `prettier-plugin-organize-imports` (single quotes, 4 spaces, 110 width, trailing commas). Husky v4.3 pre-commit runs lint-staged v10.5.4. Commitlint v19.1 with `@commitlint/config-conventional`. Custom scripts: `check-imports`, `check-core-type-defs`, `check-angular-versions`, `check-lib-imports`.
- **Location:** Root `package.json`, per-package `.eslintrc.js` / `eslint.config.js`

### G27.7: Retry / Resilience Patterns

- **Name:** Multi-Level Retry Strategies
- **Purpose:** Resilience against transient failures through retry logic at multiple levels.
- **Implementation:** Transaction deadlock retry (5 attempts, MySQL/PostgreSQL). SQLite transaction start retry (25 attempts, progressive delay). Job queue retries with `BackoffStrategy` (configurable count + delay). Entity read retry via `GetEntityOrThrowOptions.retries`. No circuit breaker library.
- **Location:** `packages/core/src/connection/transaction-wrapper.ts`, `packages/core/src/job-queue/polling-job-queue-strategy.ts`, `packages/core/src/connection/types.ts`

---

# Section 2: Frontend Concerns

## F1: Client-Side Routing

### Angular Admin UI (Legacy)

- **Name:** Angular Router with Lazy-Loaded Feature Modules
- **Purpose:** Navigation with lazy-loaded NgModules for each admin section, guarded by authentication.
- **Implementation:** Angular Router with `loadChildren` for code-splitting across 7 feature modules. `AuthGuard` checks authentication, redirects to `/login`. `ROUTES` multi-provider pattern and `PageService` for dynamic tab registration. `CanDeactivateDetailGuard` prevents navigation from unsaved forms.
- **Location:** `packages/admin-ui/src/app/app.routes.ts`, `packages/admin-ui/src/lib/core/src/providers/guard/auth.guard.ts`

### React Dashboard (New)

- **Name:** TanStack Router with File-Based Routing
- **Purpose:** Type-safe file-based routing with `beforeLoad` guards and route-level data loaders.
- **Implementation:** TanStack Router under `src/app/routes/`. `_authenticated` layout uses `beforeLoad` for auth check. Route files export `createFileRoute(...)` with `component`, `loader`, `errorComponent`. `useExtendedRouter` hook injects extension routes dynamically. `detailPageRouteLoader` prefetches via `queryClient.ensureQueryData()`. `useBlocker` for dirty form navigation blocking.
- **Location:** `packages/dashboard/src/app/routes/`, `packages/dashboard/src/lib/framework/page/use-extended-router.tsx`

---

## F2: Styling Architecture

### Angular Admin UI (Legacy)

- **Name:** Clarity Design System + SCSS Custom Properties
- **Purpose:** Design system with CSS custom property theming for light/dark modes.
- **Implementation:** Clarity UI (`@cds/core`, `@clr/icons`) + `@angular/cdk`. SCSS with hierarchical imports. ~100+ CSS custom property theme variables for colors, spacing, typography. Dark theme overrides. `ThemeSwitcherComponent` with `LocalStorageService` persistence. Component-level `.scss` via Angular view encapsulation.
- **Location:** `packages/admin-ui/src/lib/static/styles/styles.scss`, `packages/admin-ui/src/lib/static/styles/theme/`

### React Dashboard (New)

- **Name:** Tailwind CSS v4 + Radix UI + CSS Variables via Vite Plugin
- **Purpose:** Utility-first styling with shadcn/ui-compatible components and build-time theme injection.
- **Implementation:** Tailwind CSS v4 + `tw-animate-css`. Radix UI primitives (26 packages) wrapped as shadcn/ui-style components. Custom Vite plugin (`vite-plugin-theme.ts`) injects CSS variable definitions in `oklch` color space. Dark mode via `.dark` class. `ThemeProvider` React Context. `cn()` utility (`clsx` + `tailwind-merge`).
- **Location:** `packages/dashboard/src/app/styles.css`, `packages/dashboard/vite/vite-plugin-theme.ts`, `packages/dashboard/src/lib/components/ui/`

---

## F3: State Management

### Angular Admin UI (Legacy)

- **Name:** Apollo Client Cache + Angular Services
- **Purpose:** Apollo Client's normalized cache for server state, Angular services for UI state.
- **Implementation:** `apollo-angular` with `BaseDataService` wrapping queries/mutations. Apollo `InMemoryCache` for client defaults (language, locale, theme). `LocalStorageService` for browser persistence (auth tokens, channel tokens, table config, widget layouts).
- **Location:** `packages/admin-ui/src/lib/core/src/data/providers/base-data.service.ts`, `packages/admin-ui/src/lib/core/src/providers/local-storage/local-storage.service.ts`

### React Dashboard (New)

- **Name:** TanStack React Query + React Context + awesome-graphql-client
- **Purpose:** TanStack Query for server state caching, React Context for application state.
- **Implementation:** `@tanstack/react-query` with `useSuspenseQuery` / `useMutation`. `awesome-graphql-client` with `gql.tada` for type-safe GraphQL. Nested Context providers: Auth, Channel, ServerConfig, Theme, UserSettings, Alerts, I18n. `useDetailPage` hook combines query + mutation + form state.
- **Location:** `packages/dashboard/src/lib/graphql/api.ts`, `packages/dashboard/src/app/app-providers.tsx`, `packages/dashboard/src/lib/framework/page/use-detail-page.ts`

---

## F4: Form Management

### Angular Admin UI (Legacy)

- **Name:** Angular Reactive Forms
- **Purpose:** Form state, validation, and dirty tracking for entity detail pages.
- **Implementation:** `FormGroup`, `FormControl`, `FormArray`, `FormBuilder` from `@angular/forms`. `CanDeactivateDetailGuard` checks `detailForm.dirty`. `DynamicFormInputComponent` renders type-appropriate controls based on custom field config. Extension system via `registerFormInputComponent()`.
- **Location:** `packages/admin-ui/src/lib/core/src/shared/dynamic-form-inputs/`, feature module components

### React Dashboard (New)

- **Name:** react-hook-form + Auto-Generated Form Engine
- **Purpose:** Form management with automatic form generation from GraphQL schema.
- **Implementation:** `react-hook-form` (`useForm`, `useFormContext`). `useDetailPage` integrates with TanStack Query. `useGeneratedForm` auto-generates fields from GraphQL documents. `DashboardFormComponent` type for custom form components registered via `defineDashboardExtension({ formComponents })`. `useBlocker` for dirty form navigation blocking.
- **Location:** `packages/dashboard/src/lib/framework/form-engine/`, `packages/dashboard/src/lib/components/ui/form.tsx`

---

## F5: Fonts

### Angular Admin UI (Legacy)

- **Name:** Self-Hosted Inter Font (woff2)
- **Purpose:** Consistent typeface via locally-bundled font files.
- **Implementation:** Inter font as `woff2` files. 14+ `@font-face` declarations covering variable weights (100-900) for 7 Unicode ranges. No external CDN.
- **Location:** `packages/admin-ui/src/lib/static/fonts/fonts.scss`, `packages/admin-ui/src/lib/static/fonts/*.woff2`

### React Dashboard (New)

- **Name:** Google Fonts CDN (Inter + Geist Mono)
- **Purpose:** Primary typeface and monospace font from Google Fonts CDN.
- **Implementation:** `@import url(...)` in `styles.css` for Inter and Geist Mono (variable weight 100-900). Mapped to CSS custom properties `--font-sans` and `--font-mono` via Vite theme plugin. `display=swap`.
- **Location:** `packages/dashboard/src/app/styles.css`

---

## F6: Icon Libraries

### Angular Admin UI (Legacy)

- **Name:** Clarity Icons
- **Purpose:** Icon system throughout the legacy admin interface.
- **Implementation:** `@clr/icons` via `<clr-icon>` custom element. ~395 occurrences across 152 files. Global CSS import `@clr/icons/clr-icons.min.css`.
- **Location:** `packages/admin-ui/src/lib/static/styles/styles.scss`, all `.component.html` templates

### React Dashboard (New)

- **Name:** Lucide React Icons
- **Purpose:** SVG icon library as React components.
- **Implementation:** `lucide-react` with tree-shakeable individual imports. ~189 occurrences across 185 files. `LucideIcon` type used in extension API types.
- **Location:** `packages/dashboard/src/` (185 files)

---

## F7: Animations/Transitions

### Angular Admin UI (Legacy)

- **Name:** Minimal Animations
- **Purpose:** Basic animation support.
- **Implementation:** `BrowserAnimationsModule` imported but no Angular animation DSL used. SVG animation via Chartist's `element.animate()` for chart entry. Standard CSS transitions for hover effects.
- **Location:** `packages/admin-ui/src/lib/core/src/shared/components/chart/chart.component.ts`

### React Dashboard (New)

- **Name:** TailwindCSS Animate + Motion Library + CSS Keyframes
- **Purpose:** Enter/exit animations and physics-based number animations.
- **Implementation:** Three layers: (1) `tw-animate-css` for enter/exit keyframe animations (used by Radix component wrappers). (2) `motion` v12 (framer-motion) for spring-physics animated numbers in dashboard widgets. (3) Custom CSS `rotate` keyframe for loading spinners.
- **Location:** `packages/dashboard/src/app/tailwindcss-animate.css`, `packages/dashboard/src/lib/components/shared/animated-number.tsx`

---

## F8: Graphics (2D/3D)

### Angular Admin UI (Legacy)

- **Name:** Chartist.js Line Charts
- **Purpose:** Order metrics time-series charts on the dashboard.
- **Implementation:** `chartist` package for SVG line charts with area fill, custom tooltip plugin, SVG gradients, animated drawing.
- **Location:** `packages/admin-ui/src/lib/core/src/shared/components/chart/chart.component.ts`

### React Dashboard (New)

- **Name:** Recharts
- **Purpose:** Responsive chart visualizations for dashboard metrics widgets.
- **Implementation:** `recharts` with `ChartContainer` wrapping `ResponsiveContainer`, theme-aware colors (light/dark) via `ChartContext`. Chart colors as CSS custom properties (`--color-chart-1` through `--color-chart-5`).
- **Location:** `packages/dashboard/src/lib/components/ui/chart.tsx`, `packages/dashboard/src/lib/framework/dashboard-widget/metrics-widget/chart.tsx`

---

## F9: Image Management

### Angular Admin UI (Legacy)

- **Name:** Asset Preview with Focal Point Control
- **Purpose:** Displays asset images with focal point selection and size presets.
- **Implementation:** `AssetPreviewComponent` with `FocalPointControlComponent`. `AssetPreviewPipe` appends query params for presets, focal point, size. `AssetGalleryComponent` with CDK drag-drop reordering. No native lazy loading.
- **Location:** `packages/admin-ui/src/lib/core/src/shared/components/asset-preview/`, `packages/admin-ui/src/lib/core/src/shared/pipes/asset-preview.pipe.ts`

### React Dashboard (New)

- **Name:** VendureImage with Responsive Images, Lazy Loading, and Focal Point
- **Purpose:** Comprehensive image component with presets, srcset, focal point, and native lazy loading.
- **Implementation:** `VendureImage` component supports named presets, custom dimensions, focal point, format conversion, quality control, and `loading="lazy"`. `ResponsiveImage` generates srcSet with 5 breakpoints (320-1280w). `AssetFocalPointEditor` for interactive focal point selection.
- **Location:** `packages/dashboard/src/lib/components/shared/vendure-image.tsx`, `packages/dashboard/src/lib/components/shared/asset/`

---

## F10: SEO

**Not applicable.** Both frontends are internal admin interfaces behind authentication, not public-facing websites. The React dashboard sets `document.title` for tab identification only. No meta tags, SSR/SSG, structured data, or sitemap generation.

---

## F11: Printing

**Not identified.** No `@media print` CSS, `window.print()` calls, or PDF generation in either frontend.

---

## F12: Accessibility (a11y)

### Angular Admin UI (Legacy)

- **Name:** Clarity-Inherited Accessibility
- **Purpose:** Baseline accessibility through Clarity design system.
- **Implementation:** Inherited from Clarity UI (built-in ARIA, keyboard navigation, focus management). Custom ARIA sparse (~20 occurrences: `role="dialog"`, `aria-hidden`, `tabindex`). `@angular/cdk` overlay/focus trap. No dedicated a11y testing.
- **Location:** `packages/admin-ui/src/lib/core/src/shared/components/`

### React Dashboard (New)

- **Name:** Radix UI-Inherited Accessibility
- **Purpose:** Strong baseline from Radix UI's accessibility-first design.
- **Implementation:** Radix UI primitives provide comprehensive ARIA, keyboard navigation, focus trapping. Custom additions: `sr-only` (Tailwind), `aria-label` on pagination/nav, `tabIndex` on interactive elements, `role="alert"` on notifications. ~68 ARIA-related occurrences. No dedicated a11y testing tools.
- **Location:** `packages/dashboard/src/lib/components/ui/` (all Radix wrappers)

---

## F13: Other Technical Concerns (Frontend)

### F13.1: Browser Storage

- **Angular:** `LocalStorageService` with typed API, `vnd_` prefix, admin-ID namespacing. Keys: `activeChannelToken`, `authToken`, `uiLanguageCode`, `dashboardWidgetLayout`, `activeTheme`, `dataTableConfig`.
- **React:** Direct `localStorage` / `sessionStorage` with constants. Auth tokens, channel tokens, user settings, widget layouts in localStorage. Job queue polling state in sessionStorage.
- **Location:** `packages/admin-ui/src/lib/core/src/providers/local-storage/local-storage.service.ts`, `packages/dashboard/src/lib/providers/user-settings.tsx`

### F13.2: Drag and Drop

- **Angular:** `@angular/cdk/drag-drop` for collection tree reordering, dashboard widgets, asset ordering, data table columns, filter presets, list custom fields (~18 files).
- **React:** `@dnd-kit/core` + `@dnd-kit/sortable` with custom `useDragAndDrop` hook (optimistic updates with error rollback). Used for entity assets, galleries, data table views/columns, string list inputs. Dashboard widgets use a separate custom `GridLayout`.
- **Location:** `packages/admin-ui/src/lib/catalog/src/components/collection-tree/`, `packages/dashboard/src/lib/hooks/use-drag-and-drop.ts`

### F13.3: Keyboard Shortcuts

- **Angular:** Not identified. Standard browser/Clarity defaults only.
- **React:** No global hotkey system. Component-level `onKeyDown` handlers (Enter/Escape in inputs). `cmdk`-based Command component for keyboard-navigable menu. `@dnd-kit/sortable` keyboard sensor.
- **Location:** `packages/dashboard/src/lib/components/ui/command.tsx`

### F13.4: Clipboard Operations

- **Angular:** Not identified.
- **React:** `useCopyToClipboard` from `@uidotdev/usehooks` in `CopyableText` component and `PageLayout` (entity ID copying).
- **Location:** `packages/dashboard/src/lib/components/shared/copyable-text.tsx`

### F13.5: Code Splitting / Lazy Loading

- **Angular:** Route-level code splitting via `loadChildren` for 7 feature modules.
- **React:** TanStack Router auto-splits per route file. `React.Suspense` for DataTable filters. `useSuspenseQuery` for data loading. Dynamic `import()` for i18n locale loading. No `React.lazy()`.
- **Location:** `packages/admin-ui/src/app/app.routes.ts`, `packages/dashboard/src/lib/lib/load-i18n-messages.ts`

### F13.6: Virtualization

**Not identified.** Neither frontend uses virtual scrolling. Lists rely on server-side pagination via `PaginatedList` queries.

### F13.7: Error Boundaries

- **Angular:** Not applicable (Angular pattern).
- **React:** TanStack Router's `errorComponent` per route (~20+ routes define it). Global `defaultErrorComponent` on router. `ErrorPage` component renders user-friendly error. No `react-error-boundary` library.
- **Location:** `packages/dashboard/src/app/main.tsx`, `packages/dashboard/src/lib/components/shared/error-page.tsx`

### F13.8: Deep Linking

- **Angular:** URL-based via Angular Router. Tab-based navigation via `PageService.registerPageTab()`. Per-page state in localStorage.
- **React:** Predictable path patterns (`/products/{id}`, `/orders/{id}`). Login `redirect` search parameter. All entity pages directly addressable via file-based routing.
- **Location:** `packages/dashboard/src/app/routes/_authenticated/`

---

# Section 3: Backend Concerns

## B1: Entrypoints

- **Name:** Multi-Type API Entrypoints
- **Purpose:** Vendure exposes 6 primary entrypoint types for handling incoming requests.
- **Implementation:**
  1. **GraphQL API (Dual Schema):** Schema-first `.graphql` files via Apollo Server. 32 admin resolvers, 7 shop resolvers, 28+ entity field resolvers.
  2. **REST Controllers:** `@Controller()` + `@Get()`/`@Post()` for health checks, payment webhooks, plugin endpoints.
  3. **Express Router Middleware:** Plugins mount Express Routers via `NestModule.configure()` (asset server, admin UI, dashboard, GraphiQL, dev mailbox).
  4. **CLI Entrypoints:** `bootstrap()`, `bootstrapWorker()`, Commander.js CLI tools.
  5. **Job Processing:** `JobQueueService.createQueue()` with `process` callbacks.
  6. **Scheduled Tasks:** `ScheduledTask` with cron expressions via `croner`.
- **Location:** `packages/core/src/api/resolvers/`, `packages/core/src/health-check/health-check.controller.ts`, `packages/core/src/bootstrap.ts`, `packages/core/src/scheduler/`

---

## B2: Middleware

- **Name:** Multi-Layer Request Processing Pipeline
- **Purpose:** Layered middleware combining guards, interceptors, exception filters, decorators, and Apollo plugins.
- **Implementation:**
  - **Guards:** `AuthGuard` (global) -- session validation, permission checking, `RequestContext` creation.
  - **Interceptors:** `IdInterceptor` (ID decoding), `CustomFieldProcessingInterceptor` (defaults + validation), `TranslateErrorResultInterceptor` (i18n), `TransactionInterceptor` (per-handler via `@Transaction()`).
  - **Apollo Plugins:** `TranslateErrorsPlugin`, `AssetInterceptorPlugin` (URL rewriting), `IdCodecPlugin` (ID encoding).
  - **Exception Filter:** `ExceptionLoggerFilter` (logging, ErrorHandlerStrategy dispatch, REST formatting).
  - **Decorators:** `@Allow()`, `@Transaction()`, `@Ctx()`, `@Relations()`, `@Api()`.
  - **Other:** `graphql-upload` middleware, `cookie-session`, `i18next` handler.
- **Location:** `packages/core/src/api/middleware/`, `packages/core/src/api/decorators/`

---

## B3: Request Management

- **Name:** RequestContext-Centered Request Management
- **Purpose:** `RequestContext` carries all per-request state (channel, language, currency, session, user, auth) through the entire call stack.
- **Implementation:** `RequestContext` holds channel, language/currency codes, session (user ID, permissions, active order), API type, authorization flags, translation function, replication mode. Created by `AuthGuard` via `RequestContextService.fromRequest()`. Stored on Express `Request` via symbol key. Retrieved by `@Ctx()` decorator. Serializable for job queue passage. `RequestContextService.create()` for non-request contexts (scheduled tasks, jobs).
- **Location:** `packages/core/src/api/common/request-context.ts`, `packages/core/src/service/helpers/request-context/request-context.service.ts`

---

## B4: Request Routing

- **Name:** Dual GraphQL Schema Routing
- **Purpose:** Routes requests to correct handlers via two separate Apollo Server instances at configurable paths.
- **Implementation:** `ApiModule` calls `configureGraphQLModule()` twice (shop + admin). Each produces `GraphQLModule.forRootAsync()` with `ApolloDriver` at `'/' + apiPath`. Final schema composed by `getFinalVendureSchema()` merging base types, custom fields, plugin extensions, and generated types. REST via NestJS routing. Express routers via plugin middleware.
- **Location:** `packages/core/src/api/api.module.ts`, `packages/core/src/api/config/configure-graphql-module.ts`, `packages/core/src/api/schema/`

---

## B5: Response Management

- **Name:** Apollo-Plugin-Based Response Transformation
- **Purpose:** Shapes outgoing responses through Apollo plugins and NestJS interceptors.
- **Implementation:** `TranslateErrorResultInterceptor` (i18n ErrorResult messages), `IdCodecPlugin` (encode outgoing IDs), `AssetInterceptorPlugin` (relative-to-absolute URL rewriting), `TranslateErrorsPlugin` (i18n exception messages), `ExceptionLoggerFilter` (REST JSON errors). Custom scalars: Money (integer minor units), DateTime, JSON, Upload.
- **Location:** `packages/core/src/api/middleware/`

---

## B6: Public API Management

- **Name:** Schema-First GraphQL with Deprecation and Production Hardening
- **Purpose:** Manages API surface through schema-first definitions, deprecation directives, and configurable security.
- **Implementation:** 78+ `.graphql` schema files in `admin-api/`, `shop-api/`, `common/`. Standard `@deprecated(reason: "...")` directives on fields. No API versioning (deprecation for evolution). Harden plugin: `QueryComplexityPlugin` (max 1000, Shop API only), introspection/playground/debug toggles. CLI `vendure schema` for schema export.
- **Location:** `packages/core/src/api/schema/`, `packages/harden-plugin/src/`, `packages/core/src/api/config/get-final-vendure-schema.ts`

---

## B7: Caching

- **Name:** Multi-Level Pluggable Caching System
- **Purpose:** Three caching layers for different performance needs.
- **Implementation:**
  1. **Global Cache** (`CacheService` + `CacheStrategy`): `get()`, `set()`, `delete()`, `invalidateTags()` with TTL. Backends: `SqlCacheStrategy` (default), `RedisCacheStrategy` (ioredis), `InMemoryCacheStrategy`.
  2. **Per-Request Cache** (`RequestContextCacheService`): `WeakMap<RequestContext, Map>` auto-GC'd when request ends.
  3. **Session Cache** (`SessionCacheStrategy`): Caches serialized sessions to avoid per-request DB joins. Configurable TTL.
  4. **TtlCache** utility: In-memory TTL cache for internal use (5min TTL, 500 entries).
- **Location:** `packages/core/src/cache/`, `packages/core/src/plugin/default-cache-plugin/`, `packages/core/src/plugin/redis-cache-plugin/`, `packages/core/src/config/session-cache/`

---

## B8: Application Lifecycle

- **Name:** NestJS Lifecycle with Strategy Init/Destroy
- **Purpose:** Manages startup and shutdown through NestJS hooks combined with `InjectableStrategy` lifecycle pattern.
- **Implementation:**
  - **Startup (Server):** `preBootstrapConfig()` (merge config, validate, register entities, run plugin config callbacks) -> `NestFactory.create()` -> configure middleware -> `app.listen()` -> `enableShutdownHooks()`. NestJS hooks: `OnModuleInit` -> `OnApplicationBootstrap` (calls `init(injector)` on all 50+ strategies).
  - **Startup (Worker):** Same preBootstrapConfig + `NestFactory.createApplicationContext()` (no HTTP) -> `validateDbTablesForWorker()` -> returns `VendureWorker`.
  - **Shutdown:** `OnApplicationShutdown` calls `destroy()` on all strategies, stops cron jobs, stops job queues, completes EventBus subjects, closes health check server.
- **Location:** `packages/core/src/bootstrap.ts`, `packages/core/src/config/config.module.ts`, `packages/core/src/worker/vendure-worker.ts`

---

## B9: DX Tools

- **Name:** Developer Experience Tooling
- **Purpose:** API exploration, email testing, component development, and plugin prototyping tools.
- **Implementation:** GraphiQL IDE at `/graphiql/admin` and `/graphiql/shop`. Dev mailbox at `/mailbox` (email plugin file transport). Apollo Server playground (configurable per API). Debug mode toggle. Dev server test plugins for reference implementations. Storybook v10 for dashboard components. Vite HMR dashboard dev server. k6 load testing scripts.
- **Location:** `packages/graphiql-plugin/src/`, `packages/email-plugin/src/dev-mailbox.ts`, `packages/dev-server/test-plugins/`, `packages/dev-server/load-testing/`

---

## B10: Persistence

- **Name:** TypeORM-Based Persistence with Transaction-Aware Abstraction
- **Purpose:** Comprehensive data access layer supporting multi-database, transactions, channel filtering, pagination, hydration, custom fields, and migrations.
- **Implementation:** `TransactionalConnection` wraps TypeORM `DataSource` with transaction-aware `getRepository(ctx, Entity)`. `ListQueryBuilder` for paginated queries with auto-generated filters/sorts. `EntityHydrator` for lazy relation loading. `TranslatableSaver` for i18n entities. `VendureEntity` base class. Dynamic custom field registration via TypeORM decorators at bootstrap. Multi-DB: MySQL/MariaDB, PostgreSQL, SQLite. `@Transaction()` decorator with `TransactionWrapper` (deadlock retry). Migration API: `generateMigration()`, `runMigrations()`, `revertLastMigration()`.
- **Location:** `packages/core/src/connection/`, `packages/core/src/entity/`, `packages/core/src/service/helpers/list-query-builder/`, `packages/core/src/migrate.ts`

---

## B11: Search

- **Name:** Pluggable Full-Text Product Search
- **Purpose:** Product search with database-native implementation and optional Elasticsearch replacement.
- **Implementation:** `DefaultSearchPlugin` with `SearchIndexItem` entity. Event-driven index updates via job queue. `SearchStrategy` interface. DB-specific implementations: MySQL (`MATCH...AGAINST`), PostgreSQL (`to_tsvector/plainto_tsquery`), SQLite (`LIKE`/`REGEXP`). Optional `bufferUpdates` with `SearchJobBufferService` for de-duplication. `@vendure/elasticsearch-plugin` replaces default with Elasticsearch 7.x.
- **Location:** `packages/core/src/plugin/default-search-plugin/`, `packages/elasticsearch-plugin/src/`

---

## B12: Scheduling

- **Name:** Cron-Based Task Scheduling with Distributed Locking
- **Purpose:** Recurring background task scheduling with overlap prevention and multi-instance safety.
- **Implementation:** `ScheduledTask` with `schedule` (cron string or `cron-time-generator` builder), `timeout`, `preventOverlap`. `SchedulerService` creates `croner` Cron instances. `SchedulerStrategy` for distributed locking (default: DB-based pessimistic locking). Built-in tasks: session cleanup, old job cleanup, orphaned settings cleanup. Admin API: list, enable/disable, manual trigger.
- **Location:** `packages/core/src/scheduler/`, `packages/core/src/plugin/default-scheduler-plugin/`

---

## B13: Queues/Jobs

- **Name:** Pluggable Producer/Consumer Job Queue
- **Purpose:** Background job processing with typed queues, retry logic, pluggable backends, and monitoring.
- **Implementation:** `JobQueueService.createQueue({ name, process })`. `JobQueue.add(data, { retries })` returns `SubscribableJob`. Backends: `SqlJobQueueStrategy` (SQL polling, pessimistic locking), `BullMQJobQueueStrategy` (Redis, concurrency 3), `PubSubJobQueueStrategy` (GCP). `JobBuffer` for aggregation/de-duplication. Admin API monitoring. Old job cleanup via scheduled task.
- **Location:** `packages/core/src/job-queue/`, `packages/core/src/plugin/default-job-queue-plugin/`, `packages/job-queue-plugin/src/`

---

## B14: Multi-tenancy

- **Name:** Channel-Based Multi-Tenancy
- **Purpose:** Multiple storefronts/tenants share a single database with isolated catalogs, pricing, tax rules, and permissions.
- **Implementation:** `Channel` entity with own language, currency, tax/shipping zones, seller. `ChannelAware` interface (`{ channels: Channel[] }`) on entities with many-to-many relation. `vendure-token` header resolves active channel. `findOneInChannel()` / `findByIdsInChannel()` add channel JOIN + WHERE. `ListQueryBuilder` supports `channelId`. Per-channel permissions via `CachedSessionUser.channelPermissions[]`.
- **Location:** `packages/core/src/entity/channel/`, `packages/core/src/service/services/channel.service.ts`, `packages/core/src/connection/transactional-connection.ts`

---

## B15: Health Checks

- **Name:** NestJS Terminus-Based Health Checks
- **Purpose:** `/health` endpoint for application and dependency monitoring.
- **Implementation:** NestJS `@nestjs/terminus` with `@HealthCheck()` decorator. `HealthCheckRegistryService` registry. Built-in: `TypeormHealthCheckStrategy` (`SELECT 1`), `HttpHealthCheckStrategy` (HTTP GET). Plugin integration (Elasticsearch, Redis health checks). Separate worker health check server (plain Express).
- **Location:** `packages/core/src/health-check/`

---

## B16: Data Streaming

**Not identified.** No WebSocket, SSE, GraphQL Subscriptions, or real-time streaming. All client data fetching is request-response. Frontends use polling for near-real-time updates.

---

## B17: Other Technical Concerns (Backend)

### B17.1: Database Seeding

- **Name:** Programmatic Data Population
- **Purpose:** Database seeding for development, testing, and initial production setup.
- **Implementation:** `Populator` service handles countries, zones, tax rates, shipping/payment methods, roles. `Importer` for CSV product data. `populate()` CLI function. `@vendure/testing` provides `MockDataService` (deterministic faker seed) and `populateCustomers()`.
- **Location:** `packages/core/src/data-import/`, `packages/core/src/cli/populate.ts`, `packages/testing/src/data-population/`

### B17.2: Soft Delete

- **Name:** SoftDeletable Entity Pattern
- **Purpose:** Logical deletion via `deletedAt` timestamp, retaining records in database.
- **Implementation:** `SoftDeletable` interface `{ deletedAt: Date | null }`. `getEntityOrThrow()` auto-excludes soft-deleted (unless `includeSoftDeleted: true`). Services set `deletedAt = new Date()` on delete.
- **Location:** `packages/core/src/common/types/common-types.ts`, `packages/core/src/connection/transactional-connection.ts`

### B17.3: Audit Trail / History

- **Name:** Polymorphic History Entry System
- **Purpose:** Timestamped audit trail for orders and customers.
- **Implementation:** `HistoryEntry` abstract entity with `@TableInheritance({ column: 'discriminator' })`. Concrete: `OrderHistoryEntry`, `CustomerHistoryEntry`. Records `type` (enum), `data` (JSON), `isPublic`, optional `administrator` reference. `HistoryService` creates entries for state transitions, notes, modifications.
- **Location:** `packages/core/src/entity/history-entry/`, `packages/core/src/service/services/history.service.ts`

### B17.4: Static File Serving

- **Name:** Plugin-Based Static File Serving
- **Purpose:** Serves compiled admin UIs and assets as static files.
- **Implementation:** `AdminUiPlugin` and `DashboardPlugin` mount Express static middleware or proxy to Vite dev servers. `AssetServerPlugin` serves assets via separate Express router.
- **Location:** `packages/admin-ui-plugin/src/plugin.ts`, `packages/dashboard/plugin/dashboard.plugin.ts`, `packages/asset-server-plugin/src/plugin.ts`

### B17.5: Session Management

- **Name:** Dual-Mode Session Management (Cookie + Bearer)
- **Purpose:** Cookie-based and bearer token authentication.
- **Implementation:** Configurable `tokenMethod: 'cookie' | 'bearer' | both`. `extractSessionToken()` handles both. `cookie-session` middleware. Session caching via `SessionCacheStrategy`. Anonymous sessions for guest cart (`Permission.Owner`), upgradeable on login.
- **Location:** `packages/core/src/api/common/extract-session-token.ts`, `packages/core/src/service/services/session.service.ts`, `packages/core/src/bootstrap.ts`

### B17.6: Entity Duplication

- **Name:** Configurable Entity Duplicator System
- **Purpose:** Deep-copying entities with configurable relation inclusion.
- **Implementation:** `EntityDuplicator` as `ConfigurableOperationDef` with `forEntities`, `requiresPermission`, boolean `args` for which relations to copy. `EntityDuplicatorService` executes in a transaction. Built-in duplicators: Product, Facet, Collection, Promotion. Admin API `duplicateEntity` mutation.
- **Location:** `packages/core/src/service/helpers/entity-duplicator/`, `packages/core/src/config/entity/entity-duplicators/`

### B17.7: Money Handling

- **Name:** Pluggable Money Strategy
- **Purpose:** Configures monetary value storage, representation, and rounding.
- **Implementation:** `MoneyStrategy` interface with `moneyColumnOptions`, `precision` (default 2), `round(value, quantity)`. `DefaultMoneyStrategy`: `int` column, `Math.round()`. `BigIntMoneyStrategy`: `bigint` column. `Money` GraphQL scalar represents integers in minor units (cents). `setMoneyStrategy()` dynamically applies column type to `@Money()` fields.
- **Location:** `packages/core/src/config/entity/money-strategy.ts`, `packages/core/src/config/entity/default-money-strategy.ts`, `packages/core/src/entity/set-money-strategy.ts`

---

# Summary

## Technical Areas Summary

| Section | Identified | Not Identified | Not Applicable |
|---|---|---|---|
| **General (G1-G27)** | 24 | 3 (G8 Data Backup, G10 User Analytics, G26 Feature Flags) | 0 |
| **Frontend (F1-F13)** | 10 | 3 (F11 Printing, F13.3 Keyboard Shortcuts [Angular], F13.6 Virtualization) | 2 (F10 SEO) |
| **Backend (B1-B17)** | 16 | 1 (B16 Data Streaming) | 0 |
| **Total** | **50** | **7** | **2** |

## Key Architectural Patterns

| Pattern | Description |
|---|---|
| **Strategy Pattern** | 50+ `InjectableStrategy` interfaces for all extensible behaviors, with lifecycle management (`init`/`destroy`) |
| **Plugin System** | `@VendurePlugin()` extends NestJS `@Module()` with API extensions, entities, dashboard extensions, and config modification callbacks |
| **Dual Error Model** | `I18nError` exceptions for true errors + `ErrorResult` GraphQL unions for business rule violations |
| **Event-Driven Architecture** | RxJS EventBus with 60+ event types, non-blocking (post-transaction) and blocking (in-transaction) modes |
| **Channel-Based Multi-Tenancy** | `ChannelAware` entities with automatic channel filtering in all queries |
| **Translatable Entity Pattern** | `*Translation` entities with `translateEntity()` unwrapping for multi-language content |
| **RequestContext Threading** | All per-request state passed explicitly via `RequestContext` parameter (no thread-local storage) |
| **Pluggable Infrastructure** | Caching, job queues, search, scheduling, session caching, error handling, and instrumentation all use pluggable strategy interfaces |
