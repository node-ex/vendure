# Software Patterns Report - Vendure

## Overview

- Total patterns identified: 52
- Architectural styles: 6
- Architectural patterns: 14
- Enterprise integration patterns: 12
- Design patterns: 12
- Custom patterns: 8
- Components analysed: Backend (`@vendure/core`), Angular Admin UI (`@vendure/admin-ui`), React Dashboard (`@vendure/dashboard`), CLI (`@vendure/cli`), Create (`@vendure/create`), Common (`@vendure/common`), Testing (`@vendure/testing`), 10 official plugins

---

## Architectural Styles

### Three-Layer Architecture

- **Scope:** per-component: `@vendure/core`
- **Description:** The backend is organised into three distinct top-level directories corresponding to the classic presentation-business-data layers: `api/` (GraphQL resolvers, REST controllers, guards, interceptors, middleware, schema), `service/` (36 domain services + 28 helpers containing all business logic), and `entity/` (40+ TypeORM entity directories). All resolvers live under `api/resolvers/`, all services under `service/services/`, and all entities under `entity/`. This is layer-based organisation, not feature-based.
- **Evidence:** The `ApiModule` JSDoc explicitly states: "The ApiModule is responsible for the public API of the application. This is where requests come in, are parsed and then handed over to the ServiceModule classes which take care of the business logic."
- **Components:** `@vendure/core`
- **Trade-offs observed:** Clean separation of concerns at the cost of navigating between directories when working on a single feature. A product change requires touching files in `api/resolvers/`, `service/services/`, and `entity/product/` simultaneously.

### Modular Monolith with Plugin Architecture

- **Scope:** system-wide
- **Description:** The backend is a NestJS modular monolith where the core application (`AppModule`) composes sub-modules (`ApiModule`, `ServiceModule`, `ConnectionModule`, `PluginModule`, etc.) and plugins extend functionality via `@VendurePlugin()` decorator. Each plugin is an independent npm package that can add GraphQL schema extensions, entities, controllers, dashboard UI, and configuration modifications. The `PluginModule.forRoot()` dynamic module discovers and imports all configured plugins at bootstrap.
- **Evidence:** `packages/core/src/app.module.ts` composes 8 sub-modules; `PluginModule.forRoot()` dynamically imports from `getConfig().plugins`; 10 official plugin packages in `packages/`; `@VendurePlugin()` extends NestJS `@Module()` with Vendure-specific metadata.
- **Components:** `@vendure/core`, all 10 official plugins
- **Trade-offs observed:** High extensibility through well-defined plugin contracts. Plugin isolation is at the NestJS module level but they share the same process, database, and event bus -- not full microservice isolation.

### Headless / Client-Server

- **Scope:** system-wide
- **Description:** The backend exposes only GraphQL APIs (Admin API at `/admin-api`, Shop API at `/shop-api`). Two completely independent frontend applications (Angular Admin UI, React Dashboard) communicate with the backend exclusively via GraphQL. Both frontends are separate npm packages with their own build pipelines. External storefronts built by users also consume the Shop API.
- **Evidence:** No server-side rendering in the backend; dual GraphQL APIs via separate Apollo Server instances; Angular and React frontends as separate packages in `packages/admin-ui/` and `packages/dashboard/`; both can run simultaneously against the same backend.
- **Components:** `@vendure/core` (server), `@vendure/admin-ui` (Angular client), `@vendure/dashboard` (React client)
- **Trade-offs observed:** Complete frontend flexibility -- any technology can consume the API. The cost is maintaining two parallel admin UIs during the migration period.

### Monorepo (Lerna + npm Workspaces)

- **Scope:** system-wide
- **Description:** All 20 packages are managed in a single repository using Lerna v9 with fixed versioning (all packages share version 3.5.3) and npm workspaces. Coordinated releases publish all packages together. Shared build scripts, linting, and CI/CD configuration at the root level.
- **Evidence:** `lerna.json` with `"packages": ["packages/*"]` and `"version": "3.5.3"`; root `package.json` with orchestration scripts; 9 GitHub Actions CI/CD workflows; shared `e2e-common/` test configuration.
- **Components:** All packages
- **Trade-offs observed:** Coordinated versioning ensures compatibility across packages but means every release publishes all packages even if only one changed. The fixed versioning model simplifies dependency management at the cost of release granularity.

### Dual-Process Architecture (Server + Worker)

- **Scope:** per-component: `@vendure/core`
- **Description:** The same codebase supports two distinct runtime modes: a full HTTP server process (via `bootstrap()` → `NestFactory.create()`) and a headless worker process (via `bootstrapWorker()` → `NestFactory.createApplicationContext()`). The server handles API requests while the worker consumes background jobs. `ProcessContext` tracks `'server'` vs `'worker'` mode. `AppModule` is used for the server, `WorkerModule` for the worker.
- **Evidence:** `packages/core/src/bootstrap.ts` exports both `bootstrap()` and `bootstrapWorker()`; `setProcessContext('server')` vs `setProcessContext('worker')`; `packages/dev-server/index.ts` vs `packages/dev-server/index-worker.ts`.
- **Components:** `@vendure/core`, `@vendure/dev-server`
- **Trade-offs observed:** Clean separation of API serving and background processing without requiring separate codebases. Both processes share the same entity definitions and service implementations.

### Event-Driven Architecture (supplementary)

- **Scope:** per-component: `@vendure/core`
- **Description:** An RxJS-based `EventBus` with 60+ event types provides in-process event-driven communication. Two subscription modes: non-blocking (post-transaction, for side effects) and blocking (in-transaction, for synchronous business rules). Events drive search index updates, email notifications, CDN cache purging, and plugin integrations.
- **Evidence:** `packages/core/src/event-bus/event-bus.ts` with `Subject<VendureEvent>`; `packages/core/src/event-bus/events/` directory with 60+ event type files; `DefaultSearchPlugin` subscribes to 8 event types; `EmailPlugin` uses `EmailEventListener` DSL on top of `EventBus`.
- **Components:** `@vendure/core`, `@vendure/email-plugin`, `@vendure/elasticsearch-plugin`, `@vendure/stellate-plugin`
- **Trade-offs observed:** Loose coupling between core operations and side effects. The in-process nature means events are not durable -- if the process crashes mid-event, subscribers may not execute. This is acceptable for the use cases served (search indexing, email, cache invalidation) since they are eventually consistent.

---

## Architectural Patterns

### Dual GraphQL API (Backend-for-Frontend Variant)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Backend for Frontend (BFF)
- **Problem solved:** Different API consumers (admin users vs shop customers) need different data access patterns, permissions, and schema surfaces.
- **Implementation:** `configureGraphQLModule()` is called twice in `ApiModule` -- once for the Shop API and once for the Admin API. Each has separate schema type paths, resolver modules (`ShopApiModule`, `AdminApiModule`), validation rules, playground settings, and debug toggles. Both share the same service layer.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | API Module | `packages/core/src/api/api.module.ts` | Composes both GraphQL modules |
  | Configuration Factory | `packages/core/src/api/config/configure-graphql-module.ts` | Two-level factory producing `DynamicModule` |
  | Admin Resolvers | `packages/core/src/api/resolvers/admin/` | 29 admin-specific resolvers |
  | Shop Resolvers | `packages/core/src/api/resolvers/shop/` | 7 shop-specific resolvers |
  | Schema Files | `packages/core/src/api/schema/admin-api/`, `shop-api/` | Separate schema definitions per API |
- **Variations:** Both APIs share the same entity field resolvers (28+ in `resolvers/entity/`), service layer, and database. This is a "shared backend, split API surface" variant rather than separate backends.
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`

### Middleware Pipeline (Guard → Interceptor → Resolver → Filter)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Middleware Pipeline / Interceptor Chain
- **Problem solved:** Cross-cutting concerns (authentication, ID encoding, custom field processing, transaction management, error handling, i18n) must be applied consistently to all API operations without polluting business logic.
- **Implementation:** NestJS's request lifecycle chain with 5 globally-registered providers in `ApiModule`: `AuthGuard` (APP_GUARD), `IdInterceptor` (APP_INTERCEPTOR), `CustomFieldProcessingInterceptor` (APP_INTERCEPTOR), `TranslateErrorResultInterceptor` (APP_INTERCEPTOR), and `ExceptionLoggerFilter` (APP_FILTER). Additionally, 5 Apollo Server plugins handle GraphQL-specific concerns (`TranslateErrorsPlugin`, `AssetInterceptorPlugin`, `IdCodecPlugin`). Per-resolver `@Transaction()` decorator adds `TransactionInterceptor`.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Authentication Guard | `packages/core/src/api/middleware/auth-guard.ts` | Session validation, permission checking, RequestContext creation |
  | ID Interceptor | `packages/core/src/api/middleware/id-interceptor.ts` | Decodes incoming entity IDs |
  | Transaction Interceptor | `packages/core/src/api/middleware/transaction-interceptor.ts` | Wraps resolver in DB transaction |
  | Exception Filter | `packages/core/src/api/middleware/exception-logger.filter.ts` | Error logging, i18n, ErrorHandlerStrategy dispatch |
  | Apollo Plugins | `packages/core/src/api/middleware/` | Response-phase transformations |
- **Variations:** The `@Transaction()` decorator is opt-in per resolver method, not globally applied. Some resolvers deliberately omit it for read-only operations.
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`

### Plugin System (Microkernel)

- **Scope:** cross-component
- **Well-known name:** Microkernel / Plugin Architecture
- **Problem solved:** The core framework must be extensible without modification. Third-party and first-party extensions need well-defined contracts for adding API endpoints, entities, UI components, and configuration.
- **Implementation:** `@VendurePlugin()` class decorator extends NestJS `@Module()` with additional metadata: `configuration` (config modification callback), `shopApiExtensions` / `adminApiExtensions` (GraphQL schema + resolvers), `entities` (TypeORM entities), `dashboard` (React UI extension), `compatibility` (semver version constraint). `PluginModule.forRoot()` dynamically discovers and imports all plugins at bootstrap.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Plugin Decorator | `packages/core/src/plugin/vendure-plugin.ts` | Defines the plugin contract via `VendurePluginMetadata` |
  | Plugin Module | `packages/core/src/plugin/plugin.module.ts` | Dynamic aggregator module that imports all plugins |
  | Plugin Metadata | `packages/core/src/plugin/plugin-metadata.ts` | Metadata key constants for Reflect API |
  | Official Plugins | `packages/*-plugin/src/` | 10 first-party plugin implementations |
  | Static Init Pattern | All plugin files with `static init()` | Factory method for plugin configuration |
- **Variations:** The `static init(options)` factory pattern is universally used across all 10+ official plugins. Some plugins also implement `NestModule.configure()` to mount Express middleware.
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`, all 10 official plugins, `@vendure/dev-server` test plugins

### Strategy Pattern System (50+ Strategies)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Strategy Pattern (GoF) + Registry
- **Problem solved:** Every extensible behaviour in the framework must be swappable via configuration without modifying core code.
- **Implementation:** All strategies implement the `InjectableStrategy` interface with optional `init(injector)` and `destroy()` lifecycle hooks. `ConfigModule` acts as the strategy registry -- it collects ~45 strategy instances from `ConfigService` and calls `init()` on bootstrap, `destroy()` on shutdown. Strategies are selected via `VendureConfig` properties, not at runtime.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Base Interface | `packages/core/src/common/types/injectable-strategy.ts` | `init()` / `destroy()` lifecycle contract |
  | Registry/Lifecycle | `packages/core/src/config/config.module.ts` | Collects and initialises all strategies |
  | Injector Bridge | `packages/core/src/common/injector.ts` | Allows strategies to access DI container |
  | Config Service | `packages/core/src/config/config.service.ts` | Provides typed access to all config options |
  | Example Strategies | `packages/core/src/config/auth/`, `config/catalog/`, `config/order/`, `config/tax/`, etc. | 20+ strategy directories |
- **Variations:** Two tiers exist: (1) `InjectableStrategy` for infrastructure behaviours (auth, caching, storage, ID generation, etc.) and (2) `ConfigurableOperationDef` for user-configurable business rules (promotion conditions/actions, shipping calculators, collection filters, etc.) that additionally have typed arguments and UI metadata.
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`, all plugins that provide strategy implementations

### Context Object (RequestContext)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Context Object (PoEAA)
- **Problem solved:** Per-request state (channel, language, currency, session, user, permissions, transaction manager) must be threaded through all layers without polluting method signatures with individual parameters.
- **Implementation:** `RequestContext` class carries all request-scoped state. Created by `AuthGuard` via `RequestContextService.fromRequest()`. Stored on Express `Request` via symbol key. Retrieved by `@Ctx()` parameter decorator. Supports serialization for job queue passage. Every service method takes `ctx: RequestContext` as its first parameter.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Context Class | `packages/core/src/api/common/request-context.ts` | Carries channel, session, language, transaction |
  | Context Creation | `packages/core/src/service/helpers/request-context/request-context.service.ts` | Factory for creating contexts from HTTP requests |
  | Context Decorator | `packages/core/src/api/decorators/ctx.decorator.ts` | `@Ctx()` extracts context in resolvers |
  | Context Storage | `packages/core/src/api/common/request-context.ts:internal_setRequestContext()` | Stored on `req` object keyed by handler reference |
- **Variations:** `RequestContextStore` supports separate `default` and `withTransactionManager` slots per handler, enabling multiple resolvers within a single GraphQL request to have independent transaction contexts.
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`

### Transaction-Aware Repository Facade

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Repository Pattern + Unit of Work
- **Problem solved:** Services need transaction-aware data access without knowing whether they are running inside a transaction or not.
- **Implementation:** `TransactionalConnection` wraps TypeORM's `DataSource` and provides `getRepository(ctx, Entity)` that transparently returns a repository bound to the current transaction (extracted from `RequestContext` via `TRANSACTION_MANAGER_KEY` symbol). The `@Transaction()` decorator on resolvers creates the transaction and attaches it to the context.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Repository Facade | `packages/core/src/connection/transactional-connection.ts` | Transaction-aware repository provider |
  | Transaction Decorator | `packages/core/src/api/decorators/transaction.decorator.ts` | Composite decorator attaching `TransactionInterceptor` |
  | Transaction Wrapper | `packages/core/src/connection/transaction-wrapper.ts` | Executes work in transaction with deadlock retry |
  | List Query Builder | `packages/core/src/service/helpers/list-query-builder/list-query-builder.ts` | Paginated queries with auto-generated filters/sorts |
  | Entity Hydrator | `packages/core/src/service/helpers/entity-hydrator/entity-hydrator.service.ts` | Lazy relation loading |
- **Variations:** `ListQueryBuilder` extends the repository concept with a domain-specific query builder that translates GraphQL `ListQueryOptions` into TypeORM `SelectQueryBuilder` chains with auto-generated filter/sort clauses.
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`

### Finite State Machine

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** State Machine / State Pattern (GoF variant)
- **Problem solved:** Orders, payments, fulfillments, and refunds must follow well-defined lifecycles with configurable transitions and guards.
- **Implementation:** Generic `FSM<T extends string, Data>` class with configuration-driven transitions. Supports `onTransitionStart` guards (can cancel), `onTransitionEnd` callbacks, and `onError` handlers. Two-phase transition: `transitionTo()` changes state immediately but defers `onTransitionEnd` to a returned `finalize()` function, allowing callers to persist the state change before finalization.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | FSM Core | `packages/core/src/common/finite-state-machine/finite-state-machine.ts` | Generic state machine with guards and lifecycle hooks |
  | Order Process | `packages/core/src/config/order/order-process.ts` | Order state machine definition |
  | Payment Process | `packages/core/src/config/payment/payment-process.ts` | Payment state machine definition |
  | Fulfillment Process | `packages/core/src/config/fulfillment/fulfillment-process.ts` | Fulfillment state machine definition |
  | Refund Process | `packages/core/src/config/refund/refund-process.ts` | Refund state machine definition |
- **Variations:** Transitions are configurable via `VendureConfig`, allowing users to add custom states and transitions. The `jumpTo()` method provides an escape hatch that bypasses all guards.
- **Usage frequency:** common (4 entity lifecycles)
- **Components:** `@vendure/core`

### Provider Composition (React Dashboard)

- **Scope:** per-component: `@vendure/dashboard`
- **Well-known name:** Dependency Injection (React Context variant)
- **Problem solved:** Cross-cutting application state (auth, channel, theme, i18n, alerts, server config, user settings) must be available to all components without prop drilling.
- **Implementation:** 8 nested React Context providers in `AppProviders` component form a dependency hierarchy where the nesting order encodes implicit dependencies. `QueryClient` is a module-level singleton shared across the React tree and route loaders.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Provider Composition | `packages/dashboard/src/app/app-providers.tsx` | 8 nested providers |
  | Auth Provider | `packages/dashboard/src/lib/providers/auth.ts` | Authentication state |
  | Channel Provider | `packages/dashboard/src/lib/providers/channel-provider.tsx` | Active channel context |
  | Theme Provider | `packages/dashboard/src/lib/providers/theme-provider.tsx` | Light/dark theme |
  | Alerts Provider | `packages/dashboard/src/lib/providers/alerts-provider.tsx` | System alert indicators |
- **Usage frequency:** ubiquitous (in React Dashboard)
- **Components:** `@vendure/dashboard`

### Dashboard Extension System (Registry + Deferred Registration)

- **Scope:** per-component: `@vendure/dashboard`
- **Well-known name:** Registry + Plugin Architecture
- **Problem solved:** Third-party plugins need to extend the React dashboard with routes, navigation items, page blocks, widgets, form components, data table columns, and alerts without modifying core code.
- **Implementation:** `GlobalRegistry` singleton (pinned to `globalThis` for Vite bundle isolation) stores extension registrations. `defineDashboardExtension()` facade function pushes registration callbacks into the registry. `executeDashboardExtensionCallbacks()` triggers actual registration later (deferred execution). 9 registrar functions handle different extension types.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Global Registry | `packages/dashboard/src/lib/framework/registry/global-registry.ts` | Singleton registry with `globalThis` pinning |
  | Extension Facade | `packages/dashboard/src/lib/framework/extension-api/define-dashboard-extension.ts` | Single entry point delegating to 9 registrars |
  | Document Extension | `packages/dashboard/src/lib/hooks/use-extended-list-query.ts` | Runtime GraphQL AST decoration with fallback |
  | Route Extension | `packages/dashboard/src/lib/framework/page/use-extended-router.tsx` | Dynamic TanStack route injection |
- **Variations:** Extensions gracefully degrade: if a query extension fails, the system falls back to the original query with a toast notification. This resilience pattern is critical for plugin architectures.
- **Usage frequency:** ubiquitous (in React Dashboard)
- **Components:** `@vendure/dashboard`

### Angular Admin UI Extension System (PageService + App Initializer)

- **Scope:** per-component: `@vendure/admin-ui`
- **Well-known name:** Registry + Service Locator
- **Problem solved:** Plugins need to add tabs to existing pages, register custom form inputs, add navigation items, and extend the Angular admin UI without modifying core templates.
- **Implementation:** `PageService` maintains a `Map<PageLocationId, PageTabConfig[]>` registry. `registerPageTab()` returns Angular's `provideAppInitializer()` to ensure registration runs during bootstrap. `ComponentRegistryService` maps component IDs to Angular component types for dynamic form inputs. `DynamicFormInputComponent` uses `ViewContainerRef.createComponent()` for runtime component instantiation.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Page Tab Registry | `packages/admin-ui/src/lib/core/src/providers/page/page.service.ts` | Map-based tab registration |
  | Registration Helper | `packages/admin-ui/src/lib/core/src/extension/register-page-tab.ts` | App initializer wrapper |
  | Component Registry | `packages/admin-ui/src/lib/core/src/providers/component-registry/component-registry.service.ts` | Form input component registry |
  | Dynamic Host | `packages/admin-ui/src/lib/core/src/shared/dynamic-form-inputs/dynamic-form-input/dynamic-form-input.component.ts` | Runtime component instantiation |
- **Usage frequency:** ubiquitous (in Angular Admin UI)
- **Components:** `@vendure/admin-ui`

### GraphQL Data Gateway (Angular BaseDataService)

- **Scope:** per-component: `@vendure/admin-ui`
- **Well-known name:** Gateway / Facade
- **Problem solved:** All GraphQL operations must transparently handle custom field injection, read-only field removal, and relation input transformation without requiring awareness in each component.
- **Implementation:** `BaseDataService` wraps Apollo Angular. `query()` automatically injects custom field selections into the GraphQL AST. `mutate()` additionally strips read-only custom fields and transforms relation inputs. No component talks to Apollo directly.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Data Gateway | `packages/admin-ui/src/lib/core/src/data/providers/base-data.service.ts` | Wraps Apollo with custom field pipeline |
  | AST Modifier | `packages/admin-ui/src/lib/core/src/data/utils/add-custom-fields.ts` | Injects custom fields into GraphQL documents |
  | Input Transformer | `packages/admin-ui/src/lib/core/src/data/utils/remove-readonly-custom-fields.ts` | Strips read-only fields from mutation variables |
- **Usage frequency:** ubiquitous (in Angular Admin UI)
- **Components:** `@vendure/admin-ui`

### useDetailPage / ListPage Framework (React Dashboard)

- **Scope:** per-component: `@vendure/dashboard`
- **Well-known name:** Template Method (hook-based variant)
- **Problem solved:** Entity detail and list pages share identical structure (query, form, mutations, navigation blocking, custom fields) but differ in their GraphQL documents and entity types.
- **Implementation:** `useDetailPage` custom hook orchestrates `useSuspenseQuery`, `useMutation`, `useGeneratedForm`, `useExtendedDetailQuery`, and `useCustomFieldConfig`. Callers provide GraphQL documents and transformation functions; the hook handles the lifecycle. `ListPage` is a framework component standardising list views with `PageLayout`, `PaginatedListDataTable`, bulk actions, and action bars.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Detail Page Hook | `packages/dashboard/src/lib/framework/page/use-detail-page.ts` | Orchestrates query + mutation + form for detail pages |
  | List Page Component | `packages/dashboard/src/lib/framework/page/list-page.tsx` | Standardised list view shell |
  | Page Layout Engine | `packages/dashboard/src/lib/framework/layout-engine/page-layout.tsx` | Composable layout primitives |
  | Route Loader | `packages/dashboard/src/lib/framework/page/detail-page-route-loader.tsx` | Data prefetching via TanStack Router loaders |
- **Usage frequency:** ubiquitous (in React Dashboard -- all 22+ pages)
- **Components:** `@vendure/dashboard`

### Configurable Operation System

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Custom (closest: Rule Engine / Specification Pattern)
- **Problem solved:** Business rules (promotion conditions/actions, shipping eligibility/rate calculations, collection filters, payment handling, entity duplication) must be admin-configurable at runtime with typed arguments and auto-generated UI.
- **Implementation:** `ConfigurableOperationDef<T extends ConfigArgs>` base class provides `code`, typed `args` definitions (with compile-time type mapping from config arg types to TypeScript types), localized `description`, and `InjectableStrategy` lifecycle. Each operation type has specific abstract methods (e.g., `test()` for conditions, `execute()` for actions). UI metadata (`ui` property on args) co-locates backend validation with frontend rendering instructions.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Base Class | `packages/core/src/common/configurable-operation.ts` | Type-safe argument system with UI metadata |
  | Promotion Conditions | `packages/core/src/config/promotion/conditions/` | 5 built-in conditions |
  | Promotion Actions | `packages/core/src/config/promotion/actions/` | 7 built-in actions |
  | Shipping Calculators | `packages/core/src/config/shipping-method/` | Rate calculation strategies |
  | Collection Filters | `packages/core/src/config/catalog/` | Collection membership rules |
- **Usage frequency:** common (9 operation types)
- **Components:** `@vendure/core`

---

## Enterprise Integration Patterns

### Message Channel (Point-to-Point)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Message Channel (Hohpe/Woolf)
- **Problem solved:** Long-running operations (search indexing, email sending, data import) must be offloaded from the request-response cycle.
- **Implementation:** `JobQueue<Data>` is a named message channel. `add()` places typed `Job<Data>` messages onto the queue. `JobQueueStrategy` is the pluggable channel adapter (SQL polling, BullMQ/Redis, GCP Pub/Sub).
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Channel | `packages/core/src/job-queue/job-queue.ts` | Named queue with typed messages |
  | Message | `packages/core/src/job-queue/job.ts` | Job envelope with data, retries, queue name |
  | Channel Adapter | `packages/core/src/job-queue/polling-job-queue-strategy.ts` | SQL polling implementation |
  | Redis Adapter | `packages/job-queue-plugin/src/bullmq/bullmq-job-queue-strategy.ts` | BullMQ implementation |
  | Pub/Sub Adapter | `packages/job-queue-plugin/src/pub-sub/pub-sub-job-queue-strategy.ts` | GCP implementation |
- **External systems involved:** Redis (BullMQ), Google Cloud Pub/Sub, SQL database (polling strategy)
- **Usage frequency:** common (search indexing, email, collection reindexing)
- **Components:** `@vendure/core`, `@vendure/job-queue-plugin`, `@vendure/email-plugin`, `@vendure/elasticsearch-plugin`

### Publish-Subscribe Channel (EventBus)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Publish-Subscribe Channel (Hohpe/Woolf)
- **Problem solved:** Core domain operations must trigger side effects (search indexing, email, cache invalidation) without coupling to those consumers.
- **Implementation:** RxJS `Subject<VendureEvent>` with `ofType()` and `filter()` subscription methods. Non-blocking subscribers receive events after transaction commit (via `awaitActiveTransactions()`). Events from rolled-back transactions are dropped.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Event Bus | `packages/core/src/event-bus/event-bus.ts` | RxJS Subject-based pub-sub |
  | Events | `packages/core/src/event-bus/events/` | 60+ typed event classes |
  | Search Subscriber | `packages/core/src/plugin/default-search-plugin/default-search-plugin.ts` | Subscribes to 8 event types |
  | Email Subscriber | `packages/email-plugin/src/plugin.ts` | EmailEventListener DSL |
  | Cache Subscriber | `packages/stellate-plugin/src/stellate-plugin.ts` | CDN purge rules |
- **External systems involved:** None (in-process)
- **Usage frequency:** ubiquitous (60+ event types, multiple subscribers)
- **Components:** `@vendure/core`, all plugins that subscribe to events

### Aggregator (Job Buffer)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Aggregator (Hohpe/Woolf)
- **Problem solved:** Rapid-fire entity changes (e.g., bulk product updates) produce many redundant search index update jobs that should be batched and deduplicated.
- **Implementation:** `JobBuffer<Data>` interface with `collect(job): boolean` (selective interception) and `reduce(jobs): Job[]` (batch aggregation). `SearchIndexJobBuffer` collects search index jobs, deduplicates variant IDs via `unique()`, and deduplicates product jobs via `Set<ID>`.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Buffer Interface | `packages/core/src/job-queue/job-buffer/job-buffer.ts` | `collect()` + `reduce()` contract |
  | Search Buffer | `packages/core/src/plugin/default-search-plugin/search-job-buffer/search-index-job-buffer.ts` | Variant ID aggregation + product deduplication |
  | Collection Buffer | `packages/core/src/plugin/default-search-plugin/search-job-buffer/collection-job-buffer.ts` | Collection update batching |
  | Buffer Service | `packages/core/src/job-queue/job-buffer/job-buffer.service.ts` | Manages active buffers and flush lifecycle |
- **External systems involved:** None (in-process interceptor on job queue)
- **Usage frequency:** occasional (search plugin when `bufferUpdates: true`)
- **Components:** `@vendure/core`

### Anti-Corruption Layer (Payment Webhooks)

- **Scope:** per-component: `@vendure/payments-plugin`
- **Well-known name:** Anti-Corruption Layer (DDD / EIP)
- **Problem solved:** External payment providers (Stripe, Mollie) use different domain models, identifiers, and state machines that must not leak into core Vendure domain concepts.
- **Implementation:** Payment webhook controllers translate between external domain models and Vendure's domain. Stripe webhooks extract Vendure metadata (`channelToken`, `orderCode`, `orderId`) from Stripe PaymentIntent metadata, construct a Vendure `RequestContext`, verify signatures, and route events to Vendure service calls.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Stripe ACL | `packages/payments-plugin/src/stripe/stripe.controller.ts` | Translates Stripe events to Vendure operations |
  | Mollie ACL | `packages/payments-plugin/src/mollie/mollie.controller.ts` | Translates Mollie webhooks to Vendure operations |
  | Stripe Service | `packages/payments-plugin/src/stripe/stripe.service.ts` | Wraps Stripe SDK behind Vendure-compatible interface |
  | Mollie Service | `packages/payments-plugin/src/mollie/mollie.service.ts` | Wraps Mollie SDK behind Vendure-compatible interface |
- **External systems involved:** Stripe, Mollie, Braintree
- **Usage frequency:** common (3 payment providers)
- **Components:** `@vendure/payments-plugin`

### Competing Consumers

- **Scope:** per-component: `@vendure/core`, `@vendure/job-queue-plugin`
- **Well-known name:** Competing Consumers (Hohpe/Woolf)
- **Problem solved:** Background jobs must be processed concurrently by multiple workers for throughput.
- **Implementation:** `PollingJobQueueStrategy` supports configurable `concurrency` (default 1). `BullMQJobQueueStrategy` supports concurrency 3 by default. Multiple worker processes can consume from the same named queue via pessimistic locking (SQL) or native consumer groups (BullMQ).
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | SQL Poller | `packages/core/src/job-queue/polling-job-queue-strategy.ts` | Poll-based with pessimistic locking |
  | BullMQ Consumer | `packages/job-queue-plugin/src/bullmq/bullmq-job-queue-strategy.ts` | Push-based via Redis |
- **External systems involved:** Redis (BullMQ)
- **Usage frequency:** common
- **Components:** `@vendure/core`, `@vendure/job-queue-plugin`

### Polling Consumer

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Polling Consumer (Hohpe/Woolf)
- **Problem solved:** The default job queue backend (SQL) cannot push notifications; consumers must poll for new work.
- **Implementation:** `ActiveQueue` class uses RxJS `interval()` to poll at configurable intervals. `race()` operator races job processing against a shutdown signal for graceful termination.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Active Queue | `packages/core/src/job-queue/polling-job-queue-strategy.ts` | RxJS-based poll loop with graceful shutdown |
- **External systems involved:** SQL database
- **Usage frequency:** common (default job queue implementation)
- **Components:** `@vendure/core`

### Content-Based Router

- **Scope:** cross-component
- **Well-known name:** Content-Based Router (Hohpe/Woolf)
- **Problem solved:** Events, webhooks, and jobs carry type discriminators that determine which processing logic to apply.
- **Implementation:** Event subscriptions check `event.type` (created/updated/deleted) to route to different operations. Stripe webhook controller routes by `event.type` (payment_intent.succeeded vs payment_intent.payment_failed). Job buffer `collect()` routes by `job.data.type` and `job.queueName`.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Event Routing | `packages/core/src/plugin/default-search-plugin/default-search-plugin.ts` | Routes by `event.type` |
  | Webhook Routing | `packages/payments-plugin/src/stripe/stripe.controller.ts` | Routes by Stripe `event.type` |
  | Buffer Routing | `packages/core/src/plugin/default-search-plugin/search-job-buffer/search-index-job-buffer.ts` | Routes by `job.data.type` |
- **External systems involved:** Stripe webhooks
- **Usage frequency:** common
- **Components:** `@vendure/core`, `@vendure/payments-plugin`

### Transactional Client with Retry

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Transactional Client (Hohpe/Woolf) + Retry (Resilience)
- **Problem solved:** Database operations must be atomic, and transient failures (deadlocks, lock contention) must be retried automatically.
- **Implementation:** `TransactionWrapper.executeInTransaction()` wraps work in a database transaction with commit-on-success, rollback-on-failure. Deadlock retry (5 attempts) for MySQL `ER_LOCK_DEADLOCK` and PostgreSQL `deadlock_detected`. SQLite transaction start retry (25 attempts with linear backoff at `attempts * 20ms`).
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Transaction Wrapper | `packages/core/src/connection/transaction-wrapper.ts` | Transaction lifecycle with deadlock retry |
  | Transaction Decorator | `packages/core/src/api/decorators/transaction.decorator.ts` | Declarative transaction annotation |
  | Job Retry | `packages/core/src/job-queue/polling-job-queue-strategy.ts` | Configurable `BackoffStrategy` for job retries |
- **External systems involved:** Database (MySQL, PostgreSQL, SQLite)
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`

### Content Enricher

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Content Enricher (Hohpe/Woolf)
- **Problem solved:** During CSV product import, asset paths are references that need to be resolved into actual binary content from URLs or local files.
- **Implementation:** `DefaultAssetImportStrategy` enriches import data by fetching binary content. `getStreamFromPath()` normalises HTTP/HTTPS URLs (via `node-fetch` with retry) and local filesystem paths into uniform `Readable` streams.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Asset Import Strategy | `packages/core/src/config/asset-import-strategy/default-asset-import-strategy.ts` | URL/file path → Readable stream |
- **External systems involved:** External HTTP servers (asset URLs), local filesystem
- **Usage frequency:** occasional (during data import)
- **Components:** `@vendure/core`

### Event-Driven Consumer (CQRS Query-Side Update)

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Event-Driven Consumer (Hohpe/Woolf) + CQRS Read Model
- **Problem solved:** The product search index (denormalised read model) must stay synchronised with the canonical entity data (write model).
- **Implementation:** `DefaultSearchPlugin` subscribes to 8 event types. Each event dispatches a typed job to the `update-search-index` job queue. `IndexerController` processes jobs, updating the `SearchIndexItem` read model table. Optional `JobBuffer` for deduplication.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Event Subscriber | `packages/core/src/plugin/default-search-plugin/default-search-plugin.ts` | Subscribes to 8 event types |
  | Index Controller | `packages/core/src/plugin/default-search-plugin/indexer/search-index.service.ts` | Processes index update jobs |
  | Read Model | `packages/core/src/plugin/default-search-plugin/search-index-item.entity.ts` | Denormalised search entity |
- **External systems involved:** None (or Elasticsearch via `@vendure/elasticsearch-plugin`)
- **Usage frequency:** common
- **Components:** `@vendure/core`, `@vendure/elasticsearch-plugin`

### Guaranteed Delivery

- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Guaranteed Delivery (Hohpe/Woolf)
- **Problem solved:** Background jobs must not be lost on failure; they must be retried.
- **Implementation:** Jobs specify a `retries` count. `PollingJobQueueStrategy` re-enqueues failed jobs with `BackoffStrategy` (configurable delay function). `BullMQJobQueueStrategy` uses Redis-backed job persistence. The `SqlJobQueueStrategy` uses pessimistic locking to prevent duplicate consumption.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Job Options | `packages/core/src/job-queue/types.ts` | `retries` configuration |
  | Backoff Strategy | `packages/core/src/job-queue/polling-job-queue-strategy.ts` | Pluggable delay function |
- **External systems involved:** SQL database, Redis
- **Usage frequency:** common
- **Components:** `@vendure/core`, `@vendure/job-queue-plugin`

---

## Design Patterns

### Strategy (GoF) -- Infrastructure Strategies

- **Category:** Behavioural
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Strategy
- **Problem solved:** Behaviours like authentication, caching, storage, ID generation, and money handling must be swappable without code changes.
- **Implementation:** 50+ strategy interfaces extending `InjectableStrategy`. Each has a default implementation and can be replaced via `VendureConfig`. Selected at configuration time, not runtime.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Strategy Interface | `packages/core/src/common/types/injectable-strategy.ts` | 10 | Base lifecycle contract |
  | Auth Strategy | `packages/core/src/config/auth/authentication-strategy.ts` | -- | Pluggable authentication |
  | Cache Strategy | `packages/core/src/config/system/cache-strategy.ts` | -- | Pluggable caching backend |
  | Storage Strategy | `packages/core/src/config/asset-storage-strategy/asset-storage-strategy.ts` | -- | Pluggable file storage |
  | ID Strategy | `packages/core/src/config/entity/entity-id-strategy.ts` | -- | Pluggable ID generation |
  | Money Strategy | `packages/core/src/config/entity/money-strategy.ts` | -- | Pluggable money handling |
- **Instances found:** 50+
- **Components:** `@vendure/core`, all plugins providing strategy implementations

### Observer (GoF) -- EventBus

- **Category:** Behavioural
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Observer
- **Problem solved:** Multiple independent consumers must react to domain events without coupling to the publisher.
- **Implementation:** RxJS `Subject<VendureEvent>` with dual subscription modes: non-blocking (`ofType()` returning `Observable`) and blocking (`registerBlockingEventHandler()` with topological ordering).
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Subject | `packages/core/src/event-bus/event-bus.ts` | 101 | `Subject<VendureEvent>` |
  | Publisher | All 36+ services | -- | `eventBus.publish(new SomeEvent(...))` |
  | Non-blocking Subscriber | `packages/core/src/plugin/default-search-plugin/default-search-plugin.ts` | -- | `eventBus.ofType(EventType).subscribe()` |
  | Blocking Subscriber | Various | -- | `eventBus.registerBlockingEventHandler()` |
- **Instances found:** 60+ event types, dozens of subscribers
- **Components:** `@vendure/core`, all plugins

### Proxy (GoF) -- Instrumentation

- **Category:** Structural
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Proxy
- **Problem solved:** All service method calls must be instrumentable for distributed tracing without modifying each method.
- **Implementation:** `@Instrument()` class decorator wraps instances in JavaScript `Proxy`. The proxy's `get` trap intercepts all method calls and delegates to `InstrumentationStrategy.wrapMethod()` for span creation. Conditionally activated via `VENDURE_ENABLE_INSTRUMENTATION` env var.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Proxy Decorator | `packages/core/src/common/instrument-decorator.ts` | 76 | JS `Proxy` wrapping instance methods |
  | Strategy | `packages/core/src/config/system/instrumentation-strategy.ts` | -- | `wrapMethod()` hook |
  | OTel Implementation | `packages/telemetry-plugin/src/config/otel-instrumentation-strategy.ts` | -- | OpenTelemetry span creation |
- **Instances found:** 36+ services decorated with `@Instrument()`
- **Components:** `@vendure/core`, `@vendure/telemetry-plugin`

### Factory Method (GoF) -- Module and Environment Creation

- **Category:** Creational
- **Scope:** cross-component
- **Well-known name:** Factory Method
- **Problem solved:** Complex object construction (NestJS modules, test environments) requires encapsulated creation logic.
- **Implementation:** `configureGraphQLModule()` is a two-level factory: outer produces `DynamicModule`, inner produces `ApolloDriverConfig`. `createTestEnvironment()` produces a fully wired `TestEnvironment` with server and clients. `PluginModule.forRoot()` is a dynamic module factory.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | GraphQL Module Factory | `packages/core/src/api/config/configure-graphql-module.ts` | 35 | Two-level factory |
  | Test Environment Factory | `packages/testing/src/create-test-environment.ts` | 60 | Object Mother for tests |
  | Plugin Module Factory | `packages/core/src/plugin/plugin.module.ts` | 14 | `forRoot()` dynamic module |
- **Instances found:** 5+
- **Components:** `@vendure/core`, `@vendure/testing`

### Decorator (GoF) -- VendurePlugin and GraphQL Document Extension

- **Category:** Structural
- **Scope:** cross-component
- **Well-known name:** Decorator
- **Problem solved:** Classes and GraphQL documents must be augmented with additional behaviour/fields without modification.
- **Implementation:** `@VendurePlugin()` wraps NestJS `@Module()` with Vendure-specific metadata. `useExtendedListQuery()` and `useExtendedDetailQuery()` dynamically decorate GraphQL `DocumentNode` ASTs with additional fields from extensions at runtime.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Class Decorator | `packages/core/src/plugin/vendure-plugin.ts` | 164 | Augments `@Module()` with plugin metadata |
  | Document Decorator | `packages/dashboard/src/lib/hooks/use-extended-list-query.ts` | 27 | `Array.reduce()` pipeline extending GraphQL AST |
  | Instrument Decorator | `packages/core/src/common/instrument-decorator.ts` | 60 | Wraps class with Proxy-based instrumentation |
- **Instances found:** Every plugin (class decorator), every list/detail page (document decorator)
- **Components:** `@vendure/core`, `@vendure/dashboard`

### Singleton (GoF) -- GlobalRegistry

- **Category:** Creational
- **Scope:** per-component: `@vendure/dashboard`
- **Well-known name:** Singleton
- **Problem solved:** The dashboard extension registry must be a single instance across Vite code-split bundles.
- **Implementation:** Double-safety: static instance field in constructor + `globalThis` pinning. `GlobalRegistry` constructor checks for existing instance, and the module-level initialization stores the instance on `globalThis`.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Singleton Registry | `packages/dashboard/src/lib/framework/registry/global-registry.ts` | 12, 47-48 | Static instance + globalThis |
- **Instances found:** 1 (but critical for the entire extension system)
- **Components:** `@vendure/dashboard`

### Layer Supertype (Fowler PEAA)

- **Category:** Structural
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Layer Supertype
- **Problem solved:** All entities need consistent identity (id), auditing (createdAt, updatedAt), and partial hydration support.
- **Implementation:** `VendureEntity` abstract class provides `id`, `createdAt`, `updatedAt`, and a partial-hydration constructor. All 40+ entity classes extend it. `VendureEvent` abstract class with `createdAt` timestamp is the layer supertype for all 60+ event types.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Entity Supertype | `packages/core/src/entity/base/base.entity.ts` | 13 | 40+ entity subclasses |
  | Event Supertype | `packages/core/src/event-bus/vendure-event.ts` | 7 | 60+ event subclasses |
  | Entity Event Supertype | `packages/core/src/event-bus/vendure-entity-event.ts` | 12 | Generic CRUD event template |
- **Instances found:** 100+ (40+ entities + 60+ events)
- **Components:** `@vendure/core`

### Prototype (GoF) -- Entity Cloning in Translation

- **Category:** Creational
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Prototype
- **Problem solved:** Translatable entities must be cloned with resolved locale strings without mutating the original entity.
- **Implementation:** `translateEntity()` creates a shallow clone via `Object.create(Object.getPrototypeOf(entity), Object.getOwnPropertyDescriptors(entity))`, preserving the prototype chain and computed properties, then copies translated strings onto the clone.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Prototype Cloning | `packages/core/src/service/helpers/utils/translate-entity.ts` | 71-73 | Shallow clone with prototype preservation |
- **Instances found:** Used for every translatable entity query (11 entity types)
- **Components:** `@vendure/core`

### Discriminated Union + Type Guard (TypeScript idiom)

- **Category:** Behavioural
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Discriminated Union (TypeScript pattern)
- **Problem solved:** GraphQL mutations return union types combining success entities with typed error objects. TypeScript must narrow these at compile time.
- **Implementation:** `ErrorResultUnion<T, E>` combines entity type with error types. `isGraphQlErrorResult()` type guard checks for `errorCode`, `message`, and `__typename` properties. `JustErrorResults<T>` conditional type extracts only error members.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Union Type | `packages/core/src/common/error/error-result.ts` | 44 | `ErrorResultUnion<T, E>` |
  | Type Guard | `packages/core/src/common/error/error-result.ts` | 71-85 | `isGraphQlErrorResult()` |
  | Generated Errors | `packages/core/src/common/error/generated-graphql-admin-errors.ts` | -- | 30+ error result classes |
- **Instances found:** Used in every mutation return type
- **Components:** `@vendure/core`

### Adapter / Bridge (GoF) -- Injector

- **Category:** Structural
- **Scope:** per-component: `@vendure/core`
- **Well-known name:** Adapter
- **Problem solved:** Strategy objects exist outside the NestJS DI container but need access to DI-managed services.
- **Implementation:** `Injector` wraps NestJS `ModuleRef` behind a simplified `get<T>()` / `resolve<T>()` facade. Passed to strategies via `init(injector)` lifecycle hook. Uses `{ strict: false }` to resolve across module boundaries.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Adapter | `packages/core/src/common/injector.ts` | 15 | Wraps ModuleRef for strategy use |
- **Instances found:** Used by all 50+ strategies during `init()`
- **Components:** `@vendure/core`

### Composite (GoF) -- Dynamic Form Input

- **Category:** Structural
- **Scope:** per-component: `@vendure/admin-ui`
- **Well-known name:** Composite
- **Problem solved:** A single form input component must handle both scalar values and list values (arrays), dynamically instantiating the appropriate child component(s).
- **Implementation:** `DynamicFormInputComponent` manages either a single `ViewContainerRef` (scalar) or a `QueryList<ViewContainerRef>` (list). It resolves the concrete component type from `ComponentRegistryService` by ID and instantiates it at runtime.
- **Key participants:**
  | Role | Example File(s) | Line(s) | Description |
  |------|-----------------|---------|-------------|
  | Composite Host | `packages/admin-ui/src/lib/core/src/shared/dynamic-form-inputs/dynamic-form-input/dynamic-form-input.component.ts` | 64 | Handles single and list modes |
  | Component Registry | `packages/admin-ui/src/lib/core/src/providers/component-registry/component-registry.service.ts` | -- | Maps IDs to Angular component types |
- **Instances found:** Used for all custom fields and configurable operation arguments in Angular admin
- **Components:** `@vendure/admin-ui`

---

## Custom Patterns

### Translatable Entity Pattern

- **Scope:** per-component: `@vendure/core`
- **Problem solved:** E-commerce entities need multi-language content (product names, descriptions, etc.) stored efficiently and retrieved by locale with fallback chains.
- **Convention:** Each translatable entity has a companion `*Translation` entity (e.g., `Product` + `ProductTranslation`). The parent implements `Translatable` interface with `translations: Translation<Self>[]`. Translatable string fields are typed as `LocaleString` (a branded `string` type). At query time, `translateEntity()` clones the entity and resolves `LocaleString` fields from the matching translation row.
- **Implementation:** `LocaleString` branded type prevents accidental assignment of plain strings. `TranslatableSaver` handles CRUD with translation diffing (add/update/remove). `translateDeep()` handles nested relations up to 2 levels. `translateTree()` handles recursive tree structures.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Type System | `packages/core/src/common/types/locale-types.ts` | `LocaleString`, `Translatable`, `Translation<T>`, `Translated<T>` |
  | Entity Cloning | `packages/core/src/service/helpers/utils/translate-entity.ts` | `translateEntity()`, `translateDeep()`, `translateTree()` |
  | Persistence | `packages/core/src/service/helpers/translatable-saver/translatable-saver.ts` | CRUD with translation diffing |
  | Translation Differ | `packages/core/src/service/helpers/translatable-saver/translation-differ.ts` | Computes add/update/remove diffs |
- **Closest well-known pattern:** Entity Attribute Value (EAV) variant / Multilingual Content pattern
- **Usage frequency:** common (11 entity types)
- **Components:** `@vendure/core`

### Entity Trait Interfaces (ChannelAware, SoftDeletable, Orderable, Taggable)

- **Scope:** per-component: `@vendure/core`
- **Problem solved:** Entities need optional cross-cutting capabilities (multi-tenancy, soft deletion, ordering, tagging) without deep inheritance hierarchies.
- **Convention:** Four marker interfaces define optional traits. Infrastructure code (ListQueryBuilder, TransactionalConnection, etc.) checks for these interfaces at runtime and applies appropriate behaviour: `ChannelAware` → automatic channel filtering in queries; `SoftDeletable` → `deletedAt` exclusion in queries; `Orderable` → position-based sorting; `Taggable` → tag association support.
- **Implementation:** TypeScript interface-based (no mixin implementations). Behaviour is centralised in infrastructure code, not in the entities themselves.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Trait Interfaces | `packages/core/src/common/types/common-types.ts` | `ChannelAware`, `SoftDeletable`, `Orderable`, `Taggable` |
  | Channel Filtering | `packages/core/src/connection/transactional-connection.ts` | `findOneInChannel()`, `findByIdsInChannel()` |
  | List Builder | `packages/core/src/service/helpers/list-query-builder/list-query-builder.ts` | Auto-filters by channel if entity is `ChannelAware` |
- **Closest well-known pattern:** Mixin / Trait pattern (without implementation inheritance)
- **Usage frequency:** ubiquitous (most entities implement at least one trait)
- **Components:** `@vendure/core`

### Calculated Property Pattern

- **Scope:** per-component: `@vendure/core`
- **Problem solved:** Some entity properties are derived from relations or computed at load time (e.g., order discounts, tax summaries) and must be visible in GraphQL responses and sortable/filterable in list queries.
- **Convention:** The `@Calculated()` method decorator registers a getter as a "calculated property" by attaching metadata to the entity prototype. `CalculatedPropertySubscriber` (a TypeORM `EntitySubscriberInterface`) listens for `afterLoad` and `afterInsert` events and relocates the getters from prototype to instance (making them enumerable for serialization). Optional `query` instructions allow SQL-level computation for sort/filter support.
- **Implementation:** Decorator stores metadata in `CALCULATED_PROPERTIES` array on prototype. Subscriber reads metadata and relocates getters via `Object.defineProperties()`.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Decorator | `packages/core/src/common/calculated-decorator.ts` | Registers getter as calculated property |
  | Subscriber | `packages/core/src/entity/subscribers.ts` | Relocates getters from prototype to instance |
  | Usage (Order) | `packages/core/src/entity/order/order.entity.ts` | `discounts`, `totalQuantity`, `subTotal`, `taxSummary` |
- **Closest well-known pattern:** Computed Property / Derived Attribute pattern
- **Usage frequency:** common (7 entity files use `@Calculated`)
- **Components:** `@vendure/core`

### VendureEntityEvent (Generic Domain Event)

- **Scope:** per-component: `@vendure/core`
- **Problem solved:** CRUD operations on entities need standardised event shapes that carry the entity, operation type, request context, and optional input.
- **Convention:** `VendureEntityEvent<Entity, Input>` is a generic abstract class enforcing a standard CRUD event shape with `entity`, `type` ('created' | 'updated' | 'deleted'), `ctx`, and optional `input`. Concrete implementations only specify type parameters.
- **Implementation:** `protected constructor` forces subclasses to pass all required fields. The `type` field is a string literal union acting as an event discriminator.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Base Class | `packages/core/src/event-bus/vendure-entity-event.ts` | Generic CRUD event template |
  | ProductEvent | `packages/core/src/event-bus/events/product-event.ts` | `VendureEntityEvent<Product, CreateProductInput \| UpdateProductInput>` |
  | OrderEvent | `packages/core/src/event-bus/events/order-event.ts` | Order-specific event |
- **Closest well-known pattern:** Domain Event (DDD)
- **Usage frequency:** common (20+ entity event types)
- **Components:** `@vendure/core`

### Dual Error Model (I18nError + ErrorResult)

- **Scope:** per-component: `@vendure/core`
- **Problem solved:** "True errors" (internal server failures) need different handling than "expected business rule violations" (insufficient stock, invalid state transitions). Both need i18n support.
- **Convention:** Two parallel error systems: (1) `I18nError` exception hierarchy (`InternalServerError`, `UserInputError`, `ForbiddenError`, etc.) for errors caught by NestJS `ExceptionLoggerFilter`, and (2) `ErrorResult` GraphQL union types for business rule violations returned as data in the response. `isGraphQlErrorResult()` type guard distinguishes them.
- **Implementation:** Mutations return union types like `Order | OrderModificationError`. I18n errors use i18next message keys. Error results have `errorCode` and `message` fields.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Exception Hierarchy | `packages/core/src/common/error/errors.ts` | `I18nError` subclasses |
  | Error Results | `packages/core/src/common/error/error-result.ts` | `ErrorResultUnion`, `isGraphQlErrorResult()` |
  | Exception Filter | `packages/core/src/api/middleware/exception-logger.filter.ts` | Catches and formats I18nErrors |
  | Generated Errors | `packages/core/src/common/error/generated-graphql-admin-errors.ts` | 30+ error result classes |
- **Closest well-known pattern:** Two-track error handling (Railway-oriented programming variant)
- **Usage frequency:** ubiquitous
- **Components:** `@vendure/core`

### Full-Stack Custom Field Pipeline

- **Scope:** cross-component
- **Problem solved:** Users must be able to add custom database columns to any entity at configuration time, with automatic schema generation, type generation, validation, and UI rendering across the entire stack.
- **Convention:** Custom fields are declared in `VendureConfig.customFields` per entity type. At bootstrap, declarations are converted to TypeORM column decorators and GraphQL schema extensions. Frontend frameworks automatically inject custom field selections into queries and generate form controls from server-provided metadata.
- **Implementation:** Backend: `registerCustomFieldsForEntity()` dynamically applies `@Column()`, `@ManyToOne()`, `@ManyToMany()`, `@Index()` decorators. `CustomFieldsValidationSubscriber` validates at runtime. Frontend (Angular): `addCustomFields()` modifies GraphQL AST. Frontend (React): `useGeneratedForm()` + `form-schema-tools.ts`.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Configuration | `packages/core/src/config/vendure-config.ts` | `customFields` property |
  | Backend Registration | `packages/core/src/entity/register-custom-entity-fields.ts` | Dynamic TypeORM decorator application |
  | Schema Generation | `packages/core/src/api/config/generate-resolvers.ts` | GraphQL type generation for custom fields |
  | Angular Injection | `packages/admin-ui/src/lib/core/src/data/utils/add-custom-fields.ts` | AST modification |
  | React Form Engine | `packages/dashboard/src/lib/framework/form-engine/form-schema-tools.ts` | Zod schema generation from custom field config |
- **Closest well-known pattern:** Metadata-driven architecture / Entity Attribute Value (EAV)
- **Usage frequency:** ubiquitous (used on every entity that supports custom fields)
- **Components:** `@vendure/core`, `@vendure/common`, `@vendure/admin-ui`, `@vendure/dashboard`

### Static Init Plugin Configuration Pattern

- **Scope:** cross-component
- **Problem solved:** All plugins need a consistent, type-safe way to accept configuration options and register them in the NestJS DI container.
- **Convention:** Every plugin class has a `static init(options: PluginOptions): Type<PluginClass>` factory method that stores options in a module-scoped variable and returns the class (or a `DynamicModule`). The options are then made available via a custom `InjectionToken` provider.
- **Implementation:** Used universally across all 10 official plugins and test plugins. The pattern ensures options are validated at bootstrap time and available via DI throughout the plugin's module.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Email Plugin | `packages/email-plugin/src/plugin.ts` | `static init(options: EmailPluginOptions)` |
  | Stripe Plugin | `packages/payments-plugin/src/stripe/stripe.plugin.ts` | `static init(options: StripePluginOptions)` |
  | Asset Server | `packages/asset-server-plugin/src/plugin.ts` | `static init(options: AssetServerOptions)` |
  | All 10+ plugins | `packages/*-plugin/src/` | Same `static init()` pattern |
- **Closest well-known pattern:** Factory Method + Module Configuration Injection
- **Usage frequency:** ubiquitous (all plugins)
- **Components:** All plugin packages

### Architectural Fitness Functions

- **Scope:** cross-component
- **Problem solved:** Cross-package architectural constraints (import boundaries, type definition completeness, version consistency) must be enforced automatically.
- **Convention:** Custom TypeScript scripts run during the version/release process to validate architectural rules: `check-imports` validates allowed cross-package dependencies, `check-core-type-defs` ensures public type definitions are complete, `check-angular-versions` ensures Angular version consistency.
- **Implementation:** TypeScript scripts in `scripts/` directory, run via `npm run version` in the root `package.json`.
- **Key participants:**
  | Role | Example File(s) | Description |
  |------|-----------------|-------------|
  | Import Checker | `scripts/check-imports.ts` | Validates cross-package import boundaries |
  | Type Def Checker | `scripts/check-core-type-defs.ts` | Ensures public API type completeness |
  | Angular Version Checker | `scripts/check-angular-versions.ts` | Enforces consistent Angular versions |
- **Closest well-known pattern:** Architectural Fitness Function (Evolutionary Architecture)
- **Usage frequency:** common (runs on every release)
- **Components:** Root → all packages

---

## Cross-Component Patterns

Patterns that are shared or enforced across multiple components.

| Pattern | Category | Components | Enforcement Mechanism |
| ------- | -------- | ---------- | --------------------- |
| Shared Kernel (`@vendure/common`) | Architectural | All packages | npm dependency + TypeScript compilation |
| Protocol Constants | Architectural | core, admin-ui, dashboard, testing, create | `shared-constants.ts` import |
| Schema-First Type Distribution | Architectural | common → core, admin-ui, dashboard | GraphQL codegen pipeline |
| Entity Layer Supertype (`VendureEntity`) | Design | core, all plugins | TypeORM inheritance requirement |
| Full-Stack Custom Field Pipeline | Custom | core, common, admin-ui, dashboard | Config → schema → codegen → auto UI |
| Static Init Plugin Protocol | Custom | All 10+ plugin packages | Convention (no enforcement tool) |
| Architectural Fitness Functions | Custom | Root → all packages | Build scripts at version time |
| Isomorphic Utilities | Architectural | core, admin-ui, dashboard | `shared-utils.ts` import |

---

## Pattern Density by Component

| Component | Arch. Styles | Arch. Patterns | Integration Patterns | Design Patterns | Custom Patterns | Total |
| --------- | ------------ | -------------- | -------------------- | --------------- | --------------- | ----- |
| `@vendure/core` | 5 | 8 | 11 | 10 | 7 | 41 |
| `@vendure/dashboard` | 1 | 4 | 0 | 2 | 1 | 8 |
| `@vendure/admin-ui` | 1 | 3 | 0 | 1 | 0 | 5 |
| `@vendure/payments-plugin` | 0 | 1 | 2 | 0 | 0 | 3 |
| `@vendure/job-queue-plugin` | 0 | 0 | 2 | 0 | 0 | 2 |
| `@vendure/email-plugin` | 0 | 0 | 1 | 0 | 0 | 1 |
| `@vendure/elasticsearch-plugin` | 0 | 0 | 1 | 0 | 0 | 1 |
| `@vendure/testing` | 0 | 0 | 0 | 1 | 0 | 1 |
| `@vendure/common` | 0 | 0 | 0 | 0 | 0 | 0 |
| Cross-component | 1 | 1 | 1 | 2 | 4 | 9 |

> Note: `@vendure/common` hosts types/utilities used by patterns in other packages but does not itself implement any pattern beyond serving as the Shared Kernel.

---

## Anti-Patterns and Inconsistencies (Optional)

Observed deviations, incomplete pattern applications, or conflicting pattern usage.

| Observation | Location | Impact | Suggestion |
| ----------- | -------- | ------ | ---------- |
| Dual frontend coexistence (Angular + React) | `packages/admin-ui/`, `packages/dashboard/` | Maintenance burden of two parallel admin UIs with overlapping functionality. Both implement the same extension API concepts differently. | Expected during migration. The Angular UI is being replaced by the React Dashboard. |
| `e2e-common/` outside `packages/` | Root level | Breaks the convention that all shared code lives in `packages/`. Not managed by Lerna workspace. | Could be moved into `packages/` for consistency, though it functions correctly as-is. |
| Dashboard plugin embedded in dashboard package | `packages/dashboard/plugin/` | Unlike `admin-ui-plugin` which is a separate package, the dashboard's NestJS plugin lives inside the dashboard package. Inconsistent packaging strategy. | Likely intentional to keep the new dashboard self-contained during development. |
| Layer-based vs feature-based organisation inconsistency | Backend (`core`) vs frontends | Backend uses layer-based organisation (api/service/entity), Angular admin uses feature-based (catalog/orders/customers), React dashboard uses hybrid. Makes cross-feature navigation in the backend harder. | The layer-based backend approach is a deliberate architectural choice with documented trade-offs. |
| No optimistic locking | `packages/core/src/entity/` | No `@VersionColumn()` usage anywhere. Concurrent admin edits could silently overwrite each other. The system relies solely on pessimistic locking for concurrency control. | Optimistic locking via version columns would provide safer concurrent editing in admin UIs. |

---

## Notes

- All patterns were identified from direct code evidence. No pattern was inferred without at least one concrete file reference.
- The "usage frequency" assessment is based on the number of distinct usages found during exploration: ubiquitous (used in virtually every module), common (used in many but not all modules), occasional (used in specific scenarios), rare (1-2 uses).
- The distinction between "architectural patterns" and "design patterns" follows the convention that architectural patterns operate at the module/subsystem level while design patterns operate at the class/function level. Some patterns (like Strategy) span both levels.
- Custom patterns include the closest well-known pattern type for reference, even when the Vendure implementation differs significantly from the canonical form.
