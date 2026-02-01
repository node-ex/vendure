# Vendure: Types of Entrypoints and Exit Points

## Overview

This report catalogues the **types** of entrypoints and exit points found in the Vendure e-commerce monorepo. It covers backend (NestJS + TypeORM + GraphQL), two frontend apps (Angular admin UI, React dashboard), CLI tools, and all official plugins.

---

# PART 1: ENTRYPOINTS

## 1. API Entrypoints

### 1.1 GraphQL API (Dual Schema -- Admin & Shop)

The primary API surface. Two separate GraphQL APIs served by Apollo Server via `@nestjs/apollo`.

**Code patterns:**
- Schema-first `.graphql` files combined with auto-generated custom field types
- `configureGraphQLModule()` factory creating `ApolloDriver` per API
- NestJS `@Resolver()`, `@Query()`, `@Mutation()`, `@ResolveField()` decorators
- Custom decorators: `@Allow()` (permissions), `@Transaction()`, `@Ctx()` (RequestContext), `@Relations()`
- Plugin extensions via `@VendurePlugin({ adminApiExtensions, shopApiExtensions })`

**Example files:**
- `packages/core/src/api/api.module.ts`
- `packages/core/src/api/config/configure-graphql-module.ts`
- `packages/core/src/api/resolvers/admin/order.resolver.ts`
- `packages/core/src/api/resolvers/shop/shop-order.resolver.ts`
- `packages/core/src/api/api-internal-modules.ts`

### 1.2 NestJS REST Controllers

Standard NestJS REST controllers registered via plugin `controllers` metadata array.

**Code patterns:**
- `@Controller('route')` class decorator with `@Get()`, `@Post()` method decorators
- Registered via `@VendurePlugin({ controllers: [MyController] })`
- Can use `@Allow()` for permission-based guarding

**Example files:**
- `packages/core/src/health-check/health-check.controller.ts`
- `packages/payments-plugin/src/stripe/stripe.controller.ts`
- `packages/payments-plugin/src/mollie/mollie.controller.ts`
- `packages/dev-server/test-plugins/rest-plugin.ts`
- `packages/core/e2e/fixtures/test-plugins/with-rest-controller.ts`

### 1.3 Express Router Middleware

Plugins mount Express `Router` instances as NestJS middleware through `NestModule.configure(consumer: MiddlewareConsumer)`.

**Code patterns:**
- Plugin implements `NestModule`, `configure(consumer)` method
- `consumer.apply(expressRouter).forRoutes('route')`
- Express `Router()` with `router.get()`, `router.use()`, `router.post()`

**Example files:**
- `packages/asset-server-plugin/src/plugin.ts`
- `packages/admin-ui-plugin/src/plugin.ts`
- `packages/dashboard/plugin/dashboard.plugin.ts`
- `packages/graphiql-plugin/src/plugin.ts`
- `packages/email-plugin/src/dev-mailbox.ts`

### 1.4 VendureConfig Middleware

Custom Express/NestJS middleware registered globally through `VendureConfig.apiOptions.middleware[]`.

**Code patterns:**
- `Middleware` interface: `{ handler, route, beforeListen }`
- Applied via `app.use(mid.route, mid.handler)` in `bootstrap()`

**Example files:**
- `packages/core/src/common/types/common-types.ts`
- `packages/core/src/bootstrap.ts`
- `packages/core/src/config/vendure-config.ts`

### 1.5 Proxy Middleware

Reverse proxy via `createProxyHandler()` for plugins running their own internal servers (Vite, Angular dev server).

**Code patterns:**
- `createProxyHandler(options: ProxyOptions): RequestHandler`
- Uses `http-proxy-middleware` under the hood

**Example files:**
- `packages/core/src/plugin/plugin-utils.ts`
- `packages/dashboard/plugin/dashboard.plugin.ts`
- `packages/admin-ui-plugin/src/plugin.ts`

### 1.6 Standalone Worker Health Check Server

Separate plain Express HTTP server (not NestJS) for worker process health monitoring.

**Code patterns:**
- Plain `express()` + `http.createServer()` on a separate port
- Single `app.get(healthRoute, ...)` handler
- Started via `worker.startHealthCheckServer({ port })`

**Example files:**
- `packages/core/src/worker/worker-health.service.ts`
- `packages/core/src/worker/vendure-worker.ts`

---

## 2. Webhook / Callback Entrypoints

### 2.1 Payment Gateway Webhook Controllers

NestJS REST controllers receiving POST requests from external payment gateways.

**Code patterns:**
- `@Controller('payments')` + `@Post('stripe')` or `@Post('mollie/:channelToken/:paymentMethodId')`
- Signature verification (Stripe: `stripe-signature` header) or URL-parameter routing (Mollie)
- Manually constructed `RequestContext` (request from external system)
- Raw body middleware for Stripe signature verification

**Example files:**
- `packages/payments-plugin/src/stripe/stripe.controller.ts`
- `packages/payments-plugin/src/mollie/mollie.controller.ts`
- `packages/payments-plugin/src/stripe/raw-body.middleware.ts`
- `packages/payments-plugin/src/stripe/stripe.plugin.ts`

### 2.2 External Authentication Callback Strategy

Strategy-based pattern where external IdPs (Keycloak/OIDC, Google OAuth) provide tokens validated through the GraphQL `authenticate` mutation.

**Code patterns:**
- Implements `AuthenticationStrategy<Data>` interface
- `defineInputType()` returns GraphQL `DocumentNode`
- `authenticate(ctx, data)` validates token against external provider
- Uses `ExternalAuthenticationService` to find/create users

**Example files:**
- `packages/core/src/config/auth/authentication-strategy.ts`
- `packages/dev-server/test-plugins/keycloak-auth/keycloak-authentication-strategy.ts`
- `packages/dev-server/test-plugins/google-auth/google-authentication-strategy.ts`

---

## 3. Background Job Entrypoints

### 3.1 Job Queue Process Function

Services declare `JobQueue<Data>` instances created via `JobQueueService.createQueue()` with a `process` callback.

**Code patterns:**
- `this.jobQueue = await this.jobQueueService.createQueue({ name, process: async (job) => {...} })`
- Jobs added via `queue.add(data, { retries })`, returns `SubscribableJob`
- Pluggable backends: `PollingJobQueueStrategy` (SQL/in-memory), `BullMQJobQueueStrategy` (Redis), `PubSubJobQueueStrategy` (GCP)

**Example files:**
- `packages/core/src/service/services/session.service.ts`
- `packages/core/src/service/services/collection.service.ts`
- `packages/core/src/plugin/default-search-plugin/indexer/search-index.service.ts`
- `packages/email-plugin/src/plugin.ts`
- `packages/elasticsearch-plugin/src/indexing/elasticsearch-index.service.ts`

### 3.2 Scheduled Tasks (Cron-based)

`ScheduledTask` instances with cron schedules, registered in `VendureConfig.schedulerOptions.tasks`.

**Code patterns:**
- `new ScheduledTask({ id, schedule: cron => cron.every(2).hours(), execute: async ({injector}) => {...} })`
- `SchedulerService` creates `croner` `Cron` instances per task
- `SchedulerStrategy` handles distributed locking (DB-based by default)

**Example files:**
- `packages/core/src/scheduler/scheduled-task.ts`
- `packages/core/src/scheduler/scheduler.service.ts`
- `packages/core/src/scheduler/tasks/clean-sessions-task.ts`
- `packages/core/src/plugin/default-job-queue-plugin/clean-jobs-task.ts`
- `packages/core/src/config/settings-store/clean-orphaned-settings-store-task.ts`

### 3.3 Job Buffer (Interceptor/Aggregation Pattern)

`JobBuffer<Data>` intercepts jobs before they reach the queue, enabling batch aggregation/de-duplication on flush.

**Code patterns:**
- `JobBuffer` interface with `collect(job): boolean` and `reduce(jobs): Job[]`
- Activation: `jobQueueService.addBuffer(buffer)`, flush: `jobQueueService.flush(buffer)`

**Example files:**
- `packages/core/src/job-queue/job-buffer/job-buffer.ts`
- `packages/core/src/plugin/default-search-plugin/search-job-buffer/search-index-job-buffer.ts`
- `packages/core/src/plugin/default-search-plugin/search-job-buffer/collection-job-buffer.ts`

---

## 4. CLI Entrypoints

### 4.1 Published CLI Tool (`@vendure/cli`)

Shebang-annotated Commander.js program with declarative `CliCommandDefinition[]` array.

**Code patterns:**
- `package.json` `"bin"` field maps `vendure` to compiled JS
- Commander.js `program.command().option().action()` chains
- Commands support interactive mode via `@clack/prompts`

**Example files:**
- `packages/cli/src/cli.ts`
- `packages/cli/src/commands/command-declarations.ts`
- `packages/cli/src/shared/cli-command-definition.ts`

### 4.2 Published Scaffolding Tool (`@vendure/create`)

Standalone `create-vendure-app` script using Commander.js + `@clack/prompts` interactive wizard.

**Code patterns:**
- `index.js` shebang wrapper requiring main module
- `program.arguments('<project-directory>').option(...).parse(process.argv)`

**Example files:**
- `packages/create/index.js`
- `packages/create/src/create-vendure-app.ts`

### 4.3 NestJS Server Bootstrap (`bootstrap()`)

Exported `bootstrap(config)` function that creates and starts the NestJS application.

**Code patterns:**
- `export async function bootstrap(userConfig): Promise<INestApplication>`
- `NestFactory.create()` + middleware/cookie/CORS config + `app.listen()`
- Invoked via `ts-node`-registered scripts

**Example files:**
- `packages/core/src/bootstrap.ts`
- `packages/dev-server/index.ts`

### 4.4 NestJS Worker Bootstrap (`bootstrapWorker()`)

Headless NestJS application context for background job processing.

**Code patterns:**
- `export async function bootstrapWorker(userConfig): Promise<VendureWorker>`
- `NestFactory.createApplicationContext()` (no HTTP listener)
- `worker.startJobQueue()`, `worker.startHealthCheckServer()`

**Example files:**
- `packages/core/src/bootstrap.ts`
- `packages/dev-server/index-worker.ts`

### 4.5 ts-node Script Entrypoints

TypeScript files run directly via `node -r ts-node/register`. Used for populate, migration, load-test, and build scripts.

**Code patterns:**
- `node -r ts-node/register <file>.ts` in `package.json` scripts
- `if (require.main === module)` guard for dual-purpose files

**Example files:**
- `packages/dev-server/populate-dev-server.ts`
- `packages/dev-server/migration.ts`
- `packages/dev-server/load-testing/run-load-test.ts`
- `packages/core/build/copy-static.ts`

### 4.6 Programmatic Migration API

Exported functions `runMigrations()`, `generateMigration()`, `revertLastMigration()` for database migrations.

**Code patterns:**
- Functions take `VendureConfig` + optional `MigrationOptions`
- Create raw TypeORM `DataSource` connections
- Detect `VENDURE_RUNNING_IN_CLI` env var for logging behavior

**Example files:**
- `packages/core/src/migrate.ts`
- `packages/dev-server/migration.ts`

### 4.7 E2E Test Bootstrap

`TestServer` class and `createTestEnvironment()` factory wrapping NestJS bootstrap for E2E testing.

**Code patterns:**
- `const { server, adminClient, shopClient } = createTestEnvironment(config)`
- `await server.init({ initialData, productsCsvPath, customerCount })` in `beforeAll()`
- Shared `e2e-common/vitest.config.mts` configuration

**Example files:**
- `packages/testing/src/test-server.ts`
- `packages/testing/src/create-test-environment.ts`
- `e2e-common/vitest.config.mts`

---

## 5. Event / Message Entrypoints

### 5.1 EventBus Non-Blocking Subscription

RxJS-based `EventBus` using `Subject<VendureEvent>`. Subscribers fire after transaction commits.

**Code patterns:**
- `eventBus.ofType(EventType).pipe(...).subscribe(event => {...})`
- Also `eventBus.filter(predicate)` for custom predicate-based filtering

**Example files:**
- `packages/core/src/event-bus/event-bus.ts`
- `packages/core/src/plugin/default-search-plugin/default-search-plugin.ts`
- `packages/email-plugin/src/plugin.ts`
- `packages/stellate-plugin/src/stellate-plugin.ts`

### 5.2 EventBus Blocking Event Handler

Synchronous handlers within the same database transaction as the publisher.

**Code patterns:**
- `eventBus.registerBlockingEventHandler({ event, id, handler, before?, after? })`
- Handlers > 100ms trigger warning log

**Example files:**
- `packages/core/src/event-bus/event-bus.ts`
- `packages/core/src/config/catalog/multi-channel-stock-location-strategy.ts`
- `packages/core/src/service/services/role.service.ts`

### 5.3 Finite State Machine Transition Hooks

Lifecycle callbacks on state transitions for Orders, Payments, Fulfillments, and Refunds.

**Code patterns:**
- `onTransitionStart`, `onTransitionEnd`, `onTransitionError` on process interfaces
- Four process types: `OrderProcess`, `PaymentProcess`, `FulfillmentProcess`, `RefundProcess`

**Example files:**
- `packages/core/src/common/finite-state-machine/finite-state-machine.ts`
- `packages/core/src/config/order/order-process.ts`
- `packages/core/src/config/payment/payment-process.ts`
- `packages/core/src/config/fulfillment/fulfillment-process.ts`

### 5.4 TypeORM Entity Subscribers

TypeORM's native `EntitySubscriberInterface` for database entity lifecycle events.

**Code patterns:**
- `@EventSubscriber()` class with `afterLoad`, `afterInsert`, `beforeInsert`, `beforeUpdate`, `afterTransactionCommit`, `afterTransactionRollback`

**Example files:**
- `packages/core/src/entity/subscribers.ts`
- `packages/core/src/connection/transaction-subscriber.ts`
- `packages/core/src/connection/custom-fields-validation-subscriber.ts`

### 5.5 Email Event Listener DSL

Domain-specific builder pattern layered on EventBus for email triggering.

**Code patterns:**
- `new EmailEventListener('event-name').on(EventType).filter(...).setRecipient(...).setSubject(...)`

**Example files:**
- `packages/email-plugin/src/event-listener.ts`
- `packages/email-plugin/src/handler/event-handler.ts`

---

## 6. Frontend Entrypoints

### 6.1 Angular Root Route Configuration

Top-level route array passed to `RouterModule.forRoot()`.

**Code patterns:**
- `RouterModule.forRoot(routes)` in `AppModule`
- Lazy-loaded feature modules via `loadChildren: () => import(...)`

**Example files:**
- `packages/admin-ui/src/app/app.routes.ts`
- `packages/admin-ui/src/app/app.module.ts`

### 6.2 Angular Lazy-Loaded Feature Module Routes

Feature areas as NgModules loaded lazily with `ROUTES` multi-provider pattern.

**Code patterns:**
- `loadChildren: () => import('@vendure/admin-ui/catalog').then(m => m.CatalogModule)`
- `providers: [{ provide: ROUTES, useFactory: createRoutes, multi: true, deps: [PageService] }]`

**Example files:**
- `packages/admin-ui/src/lib/catalog/src/catalog.module.ts`
- `packages/admin-ui/src/lib/catalog/src/catalog.routes.ts`
- `packages/admin-ui/src/lib/order/src/order.routes.ts`

### 6.3 Angular Dynamic Tab Route Registration (PageService)

Registry-based system where extensions add tabs to existing pages.

**Code patterns:**
- `pageService.registerPageTab({ location, tab, route, component })`
- `children: pageService.getPageTabRoutes('product-list')`

**Example files:**
- `packages/admin-ui/src/lib/core/src/providers/page/page.service.ts`
- `packages/admin-ui/src/lib/core/src/extension/register-page-tab.ts`

### 6.4 Angular Extension Host (iframe + postMessage)

External URLs embedded in iframes within the Angular admin, communicating via `window.postMessage`.

**Code patterns:**
- `hostExternalFrame({ path, extensionUrl, openInNewTab })`
- `window.addEventListener('message', handler)` + `extensionWindow.postMessage(response, origin)`

**Example files:**
- `packages/admin-ui/src/lib/core/src/shared/components/extension-host/host-external-frame.ts`
- `packages/admin-ui/src/lib/core/src/shared/components/extension-host/extension-host.service.ts`

### 6.5 React TanStack File-Based Routes

Convention-based file routing under `src/app/routes/` with TanStack Router.

**Code patterns:**
- `export const Route = createFileRoute('/_authenticated/_products/products')({ component, loader })`
- Auto-generated `routeTree.gen.ts`

**Example files:**
- `packages/dashboard/src/app/routes/_authenticated/_products/products.tsx`
- `packages/dashboard/src/app/routes/_authenticated/_products/products_.$id.tsx`
- `packages/dashboard/src/app/routes/_authenticated/_orders/orders.tsx`

### 6.6 React Dynamic Extension Routes

Dashboard extensions register routes at runtime via `defineDashboardExtension({ routes })`.

**Code patterns:**
- `defineDashboardExtension({ routes: [{ path, component, authenticated }] })`
- `useExtendedRouter` hook dynamically creates TanStack routes from extension registry

**Example files:**
- `packages/dashboard/src/lib/framework/page/use-extended-router.tsx`
- `packages/dashboard/src/lib/framework/extension-api/define-dashboard-extension.ts`

### 6.7 Angular Route Guards (canActivate / canDeactivate)

Injectable guard classes for authentication and unsaved changes protection.

**Code patterns:**
- `canActivate(route): Observable<boolean>` -- `AuthGuard`, `LoginGuard`, `OrderGuard`
- `canDeactivate(component: DeactivateAware): boolean | Observable<boolean>`

**Example files:**
- `packages/admin-ui/src/lib/core/src/providers/guard/auth.guard.ts`
- `packages/admin-ui/src/lib/core/src/shared/providers/routing/can-deactivate-detail-guard.ts`

### 6.8 React `beforeLoad` Route Guard

TanStack Router's `beforeLoad` hook for authentication checks.

**Code patterns:**
- `beforeLoad: ({ context }) => { if (!context.auth.isAuthenticated) throw redirect({ to: '/login' }) }`

**Example files:**
- `packages/dashboard/src/app/routes/_authenticated.tsx`
- `packages/dashboard/src/app/routes/login.tsx`

### 6.9 Angular Route Resolvers (BaseEntityResolver)

Angular route resolvers fetching entity data before navigation completes.

**Code patterns:**
- `extends BaseEntityResolver<T>` with GraphQL data fetching
- `resolve: { entity: SomeResolver }` on routes

**Example files:**
- `packages/admin-ui/src/lib/core/src/common/base-entity-resolver.ts`
- `packages/admin-ui/src/lib/catalog/src/providers/routing/product-variants-resolver.ts`

### 6.10 React Route Loaders (detailPageRouteLoader)

TanStack Router `loader` functions that prefetch entity data via React Query.

**Code patterns:**
- `loader: detailPageRouteLoader({ queryDocument, breadcrumb })`
- Calls `queryClient.ensureQueryData()` for prefetching

**Example files:**
- `packages/dashboard/src/lib/framework/page/detail-page-route-loader.tsx`
- `packages/dashboard/src/app/routes/_authenticated/_products/products_.$id.tsx`

### 6.11 Frontend Timer / Polling Patterns

Periodic data fetching and rate-limiting patterns.

**Code patterns:**
- Angular: `interval(30000)`, `timer(0, pollingDelayMs)`, `debounceTime(250)`, `throttleTime(1000)`
- React: `useQuery({ refetchInterval: 5000 })`, `setTimeout` with exponential backoff

**Example files:**
- `packages/admin-ui/src/lib/core/src/providers/health-check/health-check.service.ts`
- `packages/admin-ui/src/lib/core/src/providers/job-queue/job-queue.service.ts`
- `packages/dashboard/src/lib/hooks/use-job-queue-polling.ts`
- `packages/dashboard/src/lib/providers/alerts-provider.tsx`

### 6.12 Application Bootstrap Patterns

Framework-specific application initialization.

**Code patterns:**
- Angular: `platformBrowserDynamic().bootstrapModule(AppModule)` + `provideAppInitializer()` for extension registration
- React: `ReactDOM.createRoot(element).render(<App />)` + `virtual:dashboard-extensions` Vite module

**Example files:**
- `packages/admin-ui/src/main.ts`
- `packages/dashboard/src/app/main.tsx`
- `packages/dashboard/src/lib/framework/extension-api/use-dashboard-extensions.ts`

---

# PART 2: EXIT POINTS

## 7. HTTP Client Exit Points

### 7.1 `node-fetch` (Direct HTTP Fetch)

General-purpose HTTP client for outgoing requests.

**Code patterns:**
- `import fetch from 'node-fetch'`
- `await fetch(url, { method: 'POST', headers, body, timeout })`

**Example files:**
- `packages/core/src/health-check/http-health-check-strategy.ts`
- `packages/stellate-plugin/src/service/stellate.service.ts`
- `packages/testing/src/simple-graphql-client.ts`

### 7.2 Node.js Built-in `http` / `https`

Low-level HTTP calls without third-party clients.

**Code patterns:**
- `http.get(url, { timeout }, res => {...})` / `https.get(...)`
- `http.request({ hostname, port, method: 'HEAD' }, res => {...})`

**Example files:**
- `packages/core/src/config/asset-import-strategy/default-asset-import-strategy.ts`
- `packages/dashboard/plugin/dashboard.plugin.ts`

### 7.3 Payment Provider SDKs (Stripe, Mollie, Braintree)

Official SDK clients making HTTPS calls to payment APIs.

**Code patterns:**
- Stripe: `VendureStripeClient extends Stripe` with `stripe.paymentIntents.create()`, `stripe.refunds.create()`
- Mollie: `createMollieClient({ apiKey })` with `mollieClient.payments.create()`, `mollieClient.payments.get()`
- Braintree: `new BraintreeGateway({...})` with `gateway.transaction.sale()`, `gateway.transaction.refund()`

**Example files:**
- `packages/payments-plugin/src/stripe/stripe.service.ts`
- `packages/payments-plugin/src/mollie/mollie.service.ts`
- `packages/payments-plugin/src/braintree/braintree.handler.ts`

### 7.4 Elasticsearch Client

Official `@elastic/elasticsearch` client for search and index management.

**Code patterns:**
- `new Client({ node })` with `client.search()`, `client.bulk()`, `client.indices.create()`, `client.indices.delete()`

**Example files:**
- `packages/elasticsearch-plugin/src/elasticsearch.service.ts`
- `packages/elasticsearch-plugin/src/indexing/indexer.controller.ts`
- `packages/elasticsearch-plugin/src/indexing/indexing-utils.ts`

### 7.5 AWS S3 SDK

AWS SDK v3 for S3 object storage, dynamically imported.

**Code patterns:**
- Dynamic import: `this.AWS = await import('@aws-sdk/client-s3')`
- Command pattern: `s3Client.send(new GetObjectCommand({...}))`, `new Upload({client, params}).done()`

**Example files:**
- `packages/asset-server-plugin/src/config/s3-asset-storage-strategy.ts`

### 7.6 Sentry SDK

Error reporting and performance tracing via `@sentry/node`.

**Code patterns:**
- `Sentry.init({ dsn, integrations, tracesSampleRate })`
- `Sentry.captureException(exception)`, `Sentry.captureMessage(message)`

**Example files:**
- `packages/sentry-plugin/instrument.ts`
- `packages/sentry-plugin/src/sentry.service.ts`

### 7.7 OpenTelemetry Exporters

Trace spans and log records exported to OTLP backends (Jaeger, Loki).

**Code patterns:**
- `new OTLPTraceExporter({ url })` + `BatchSpanProcessor`
- `new OTLPLogExporter()` + `BatchLogRecordProcessor`
- `otelLogger.emit({ severityNumber, body, attributes })`

**Example files:**
- `packages/telemetry-plugin/src/instrumentation.ts`
- `packages/telemetry-plugin/src/config/otel-logger.ts`
- `packages/telemetry-plugin/src/config/otel-instrumentation-strategy.ts`

### 7.8 HTTP Proxy Middleware

Proxies incoming requests to plugin-hosted sub-servers.

**Code patterns:**
- `createProxyMiddleware({ target, pathRewrite, logger })`
- Wrapped in `createProxyHandler()` utility

**Example files:**
- `packages/core/src/plugin/plugin-utils.ts`

---

## 8. Database Exit Points

### 8.1 TransactionalConnection (Central Data Access)

Core abstraction replacing direct TypeORM `DataSource` usage. All services inject this.

**Code patterns:**
- `this.connection.getRepository(ctx, Entity)` for transaction-aware access
- `getEntityOrThrow()`, `findOneInChannel()`, `findByIdsInChannel()`, `withTransaction()`

**Example files:**
- `packages/core/src/connection/transactional-connection.ts`
- `packages/core/src/service/services/product.service.ts`
- `packages/core/src/service/services/order.service.ts`

### 8.2 Transaction Decorator and Interceptor

`@Transaction()` decorator wrapping resolver execution in a database transaction.

**Code patterns:**
- `@Transaction()` on resolver methods
- `TransactionInterceptor` creates `QueryRunner`, starts transaction, attaches `EntityManager` to `RequestContext`

**Example files:**
- `packages/core/src/api/decorators/transaction.decorator.ts`
- `packages/core/src/connection/transaction-wrapper.ts`

### 8.3 ListQueryBuilder (Paginated Queries)

Specialized helper for GraphQL `PaginatedList` queries with auto-generated filter/sort/pagination.

**Code patterns:**
- `this.listQueryBuilder.build(Entity, options, { relations, channelId, ctx, customPropertyMap }).getManyAndCount()`

**Example files:**
- `packages/core/src/service/helpers/list-query-builder/list-query-builder.ts`
- `packages/core/src/service/helpers/list-query-builder/parse-filter-params.ts`

### 8.4 Entity Hydrator (Lazy Relation Loading)

Retroactively populates entity relations not loaded in initial query.

**Code patterns:**
- `await this.entityHydrator.hydrate(ctx, entity, { relations: [...], applyProductVariantPrices })`

**Example files:**
- `packages/core/src/service/helpers/entity-hydrator/entity-hydrator.service.ts`

### 8.5 TranslatableSaver (i18n Persistence)

Helper for creating/updating `Translatable` entities with their translation rows.

**Code patterns:**
- `translatableSaver.create({ ctx, input, entityType, translationType, beforeSave })`
- `translatableSaver.update({ ctx, input, entityType, translationType })`

**Example files:**
- `packages/core/src/service/helpers/translatable-saver/translatable-saver.ts`

### 8.6 Database-Specific Search Strategies

Per-database-dialect search using DB-specific SQL functions via TypeORM `SelectQueryBuilder`.

**Code patterns:**
- MySQL: `MATCH...AGAINST`, `GROUP_CONCAT`, `BIT_OR`
- PostgreSQL: `to_tsvector`, `plainto_tsquery`
- SQLite: `REGEXP`
- All use `qb.getRawMany()` with raw column selection

**Example files:**
- `packages/core/src/plugin/default-search-plugin/search-strategy/mysql-search-strategy.ts`
- `packages/core/src/plugin/default-search-plugin/search-strategy/postgres-search-strategy.ts`
- `packages/core/src/plugin/default-search-plugin/search-strategy/sqlite-search-strategy.ts`

### 8.7 SQL-Based Infrastructure Strategies

Job queue, cache, and scheduler default implementations using TypeORM with pessimistic locking.

**Code patterns:**
- `this.rawConnection.getRepository(Entity)` for non-transactional access
- `connection.transaction(async em => {...})` with `setLock('pessimistic_write')`
- Upsert, batch delete, LRU eviction via nested subqueries

**Example files:**
- `packages/core/src/plugin/default-job-queue-plugin/sql-job-queue-strategy.ts`
- `packages/core/src/plugin/default-cache-plugin/sql-cache-strategy.ts`
- `packages/core/src/plugin/default-scheduler-plugin/default-scheduler-strategy.ts`

### 8.8 Migration System

TypeORM migration infrastructure for schema changes.

**Code patterns:**
- `generateMigration()`, `runMigrations()`, `revertLastMigration()`
- Generated files with `queryRunner.query('ALTER TABLE ...')`

**Example files:**
- `packages/core/src/migrate.ts`

### 8.9 Dynamic Custom Field Registration

Programmatic application of TypeORM decorators at bootstrap for custom fields.

**Code patterns:**
- `registerCustomFieldsForEntity(config, 'Product', CustomProductFields)`
- Dynamically applies `@Column()`, `@ManyToOne()`, `@ManyToMany()`, `@Index()` to embedded classes

**Example files:**
- `packages/core/src/entity/register-custom-entity-fields.ts`
- `packages/core/src/entity/custom-entity-fields.ts`

---

## 9. Message Queue Exit Points

### 9.1 EventBus Publish

In-process RxJS `Subject`-based event emission. ~58 event types across ~36 publishing services.

**Code patterns:**
- `await this.eventBus.publish(new SomeEvent(ctx, entity, 'created', input))`
- Events extend `VendureEvent` or `VendureEntityEvent<Entity, Input>`

**Example files:**
- `packages/core/src/event-bus/event-bus.ts`
- `packages/core/src/service/services/product.service.ts`
- `packages/core/src/service/services/order.service.ts`

### 9.2 Job Queue Add (Background Job Dispatch)

Dispatch to pluggable queue backends (SQL, Redis/BullMQ, GCP Pub/Sub).

**Code patterns:**
- `await this.jobQueue.add(data, { retries })`
- Strategy: BullMQ `queue.add()` (Redis LPUSH), Pub/Sub `topic.publish(Buffer)`

**Example files:**
- `packages/core/src/plugin/default-search-plugin/indexer/search-index.service.ts`
- `packages/email-plugin/src/plugin.ts`
- `packages/job-queue-plugin/src/bullmq/bullmq-job-queue-strategy.ts`
- `packages/job-queue-plugin/src/pub-sub/pub-sub-job-queue-strategy.ts`

---

## 10. External Service Exit Points

### 10.1 Email Delivery (Transport-based Sender)

Nodemailer with pluggable transports (SMTP, AWS SES, Sendmail, file, noop).

**Code patterns:**
- `EmailSender` interface with `send(email, options)`
- `NodemailerEmailSender` using `nodemailer.createTransport(transportOptions)`
- `EmailTransportOptions` discriminated union: `type: 'smtp' | 'ses' | 'sendmail' | 'file' | 'none'`

**Example files:**
- `packages/email-plugin/src/sender/email-sender.ts`
- `packages/email-plugin/src/sender/nodemailer-email-sender.ts`
- `packages/email-plugin/src/types.ts`

### 10.2 File/Asset Storage (Strategy)

`AssetStorageStrategy` with local filesystem and S3 implementations.

**Code patterns:**
- Strategy interface: `writeFileFromBuffer()`, `readFileToBuffer()`, `deleteFile()`, `fileExists()`
- S3: Dynamic `import('@aws-sdk/client-s3')`, `s3Client.send(new PutObjectCommand(...))`

**Example files:**
- `packages/core/src/config/asset-storage-strategy/asset-storage-strategy.ts`
- `packages/asset-server-plugin/src/config/s3-asset-storage-strategy.ts`
- `packages/asset-server-plugin/src/config/local-asset-storage-strategy.ts`

### 10.3 CDN Cache Purging (Stellate)

Event-driven HTTP POST calls to Stellate Purging API, triggered by Vendure events.

**Code patterns:**
- `StellateService.purge()` using `node-fetch` POST to `https://admin.stellate.co/{serviceName}`
- `PurgeRule` classes binding events to purge calls
- RxJS buffering/debouncing before purge

**Example files:**
- `packages/stellate-plugin/src/service/stellate.service.ts`
- `packages/stellate-plugin/src/stellate-plugin.ts`
- `packages/stellate-plugin/src/purge-rule.ts`

### 10.4 Caching Backend (Redis)

`CacheStrategy` with Redis implementation using `ioredis`.

**Code patterns:**
- `RedisCacheStrategy` with `client.get()`, `client.set()`, `client.del()`, `multi.exec()`
- Dynamic import: `await import('ioredis')`

**Example files:**
- `packages/core/src/config/system/cache-strategy.ts`
- `packages/core/src/plugin/redis-cache-plugin/redis-cache-strategy.ts`

### 10.5 Error Tracking (Sentry)

`ErrorHandlerStrategy` delegating to Sentry SDK for exception/message capture.

**Code patterns:**
- `SentryErrorHandlerStrategy` with `handleServerError()` / `handleWorkerError()`
- `SentryService` wrapping `Sentry.captureException()`, `Sentry.captureMessage()`

**Example files:**
- `packages/core/src/config/system/error-handler-strategy.ts`
- `packages/sentry-plugin/src/sentry-error-handler-strategy.ts`
- `packages/sentry-plugin/src/sentry.service.ts`

### 10.6 Observability / Telemetry (OpenTelemetry)

SDK preload + `InstrumentationStrategy` wrapping methods in trace spans.

**Code patterns:**
- `@Instrument()` class decorator proxying all methods through `InstrumentationStrategy.wrapMethod()`
- `BatchSpanProcessor(OTLPTraceExporter)`, `BatchLogRecordProcessor(OTLPLogExporter)`

**Example files:**
- `packages/core/src/common/instrument-decorator.ts`
- `packages/telemetry-plugin/src/instrumentation.ts`
- `packages/telemetry-plugin/src/config/otel-instrumentation-strategy.ts`

---

# PART 3: SUMMARY

## Entrypoint Types Summary

| Category | Type | Count |
|---|---|---|
| **API** | GraphQL (dual), REST controllers, Express router middleware, Config middleware, Proxy middleware, Worker health server | 6 |
| **Webhook/Callback** | Payment gateway webhooks, External auth callback strategies | 2 |
| **Background Jobs** | Job queue process, Scheduled tasks (cron), Job buffer | 3 |
| **CLI** | Published CLI, Scaffolding tool, Server bootstrap, Worker bootstrap, ts-node scripts, Migration API, E2E test bootstrap | 7 |
| **Events/Messages** | EventBus non-blocking, EventBus blocking, FSM transition hooks, TypeORM subscribers, Email event listener DSL | 5 |
| **Frontend** | Angular routes (root, lazy, dynamic tabs, extension iframe), React routes (file-based, dynamic extension), Guards (Angular canActivate/canDeactivate, React beforeLoad), Resolvers/loaders, Timers/polling, App bootstrap | 12 |
| **Total** | | **35** |

## Exit Point Types Summary

| Category | Type | Count |
|---|---|---|
| **HTTP Client** | node-fetch, Node.js http/https, Payment SDKs (Stripe/Mollie/Braintree), Elasticsearch, AWS S3, Sentry, OpenTelemetry, HTTP proxy | 8 |
| **Database** | TransactionalConnection, Transaction decorator, ListQueryBuilder, EntityHydrator, TranslatableSaver, DB-specific search, SQL infrastructure strategies, Migrations, Dynamic custom fields | 9 |
| **Message Queue** | EventBus publish, Job queue add | 2 |
| **External Services** | Email (Nodemailer), File storage (S3/local), CDN purging (Stellate), Caching (Redis), Error tracking (Sentry), Telemetry (OpenTelemetry) | 6 |
| **Total** | | **25** |

## Patterns NOT Found

| Pattern | Status |
|---|---|
| WebSocket / Socket.IO | Not present |
| Server-Sent Events (SSE) | Not present |
| GraphQL Subscriptions | Not present |
| gRPC | Not present |
| NestJS Microservices (RPC) | Not present |
| OpenAPI / Swagger | Not present |
| SMS sending | Not present |
| Web Workers | Not present |

## Foundational Design Pattern

Nearly all exit point strategies share the `InjectableStrategy` base interface:

```typescript
interface InjectableStrategy {
    init?: (injector: Injector) => void | Promise<void>;
    destroy?: () => void | Promise<void>;
}
```

This provides lifecycle management, pluggability (swap implementations via `VendureConfig`), and lazy dependency loading (dynamic `import()` in `init()`).
