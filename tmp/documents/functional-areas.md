# Vendure Functional Areas Report

## Overview

This report identifies all functional areas in the Vendure e-commerce monorepo, organized by application. Vendure consists of 5 distinct applications/frontends: the **Backend** (`@vendure/core`), the **Angular Admin UI** (legacy), the **React Dashboard** (new), the **CLI tool**, and the **Create scaffolding tool**. Additionally, **10 official plugins** extend the backend with specialized functionality.

---

# APPLICATION 1: Backend (`@vendure/core`)

Location: `packages/core/src/`

The backend is a NestJS-based server with a three-layer architecture (API → Service → Entity). It exposes dual GraphQL APIs (Admin API and Shop API) and contains 16 functional areas plus cross-cutting infrastructure.

---

## 1. Catalog Management

**Description:** Core product information management including products, variants, options, facets (filterable attributes), and hierarchical collections. Supports i18n translations, multi-channel assignment, and custom fields on all entities.

**Features:**

| Feature | Description |
|---|---|
| Product CRUD | Create, read, update, delete products with translatable name/slug/description, assets, facet values, and channel assignment |
| Product Variant Management | Manage product variants with SKU, pricing (multi-currency), tax category, stock tracking, and option value combinations |
| Product Option Groups & Options | Define option groups (e.g., Size, Color) and their values; generate variant combinations automatically |
| Facet Management | Create and manage facets (attributes) and facet values with translatable names, codes, and private/public visibility |
| Collection Management | Build hierarchical product collections with configurable membership filters (facet-based, product-based), filter inheritance, drag-and-drop reordering |
| Product Search Index Rebuild | Trigger background re-indexing of the product search index |
| Entity Duplication | Duplicate products, facets, and collections with configurable deep-copy options |
| Price Calculation Strategies | Pluggable strategies for variant price calculation, selection (by currency/channel), and cross-channel sync on price updates |
| Stock Display Strategy | Configure how stock levels are exposed to the public Shop API |

**Location:** `packages/core/src/entity/product/`, `packages/core/src/entity/product-variant/`, `packages/core/src/entity/facet/`, `packages/core/src/entity/collection/`, `packages/core/src/service/services/product.service.ts`, `packages/core/src/service/services/product-variant.service.ts`, `packages/core/src/service/services/facet.service.ts`, `packages/core/src/service/services/collection.service.ts`, `packages/core/src/config/catalog/`

---

## 2. Order Management

**Description:** Full order lifecycle management covering cart operations, checkout, state machine transitions (AddingItems → ArrangingPayment → PaymentAuthorized → PaymentSettled → Delivered → Cancelled, etc.), order modifications, surcharges, draft orders, and seller-based order splitting.

**Features:**

| Feature | Description |
|---|---|
| Cart Operations | Add/remove/adjust items in active orders via Shop API |
| Order State Machine | Configurable finite state machine with transitions, guards, and lifecycle hooks (`onTransitionStart`, `onTransitionEnd`) |
| Order Placement | Transition from cart to placed order with configurable "placed" detection strategy |
| Draft Orders | Admin-created orders with customer selection, address assignment, and manual pricing |
| Order Modification | Modify existing orders (add/remove lines, adjust quantities, change addresses) with price diff calculation and surcharges |
| Order Merging | Merge guest cart with authenticated customer's existing cart on login |
| Order Splitting | Split orders across sellers/channels for multi-vendor marketplace scenarios |
| Shipping Calculation | Calculate applicable shipping costs based on order contents and shipping methods |
| Active Order Resolution | Pluggable strategy to determine which order is the customer's "active" cart |
| Guest Checkout | Configurable rules for allowing guest (unauthenticated) checkouts |
| Coupon Code Application | Apply and remove promotion coupon codes to orders |
| Order History | Track all order state changes, modifications, notes, and events with timestamped history entries |
| Order Code Generation | Pluggable strategy for generating human-readable order codes |
| Price Change Handling | Strategy for handling price changes after items have been added to cart |
| Order Interceptors | Hooks to modify order behavior at various lifecycle points |
| Order by Code Access | Configurable access rules for the `orderByCode` query |

**Location:** `packages/core/src/entity/order/`, `packages/core/src/entity/order-line/`, `packages/core/src/entity/order-modification/`, `packages/core/src/entity/shipping-line/`, `packages/core/src/entity/surcharge/`, `packages/core/src/service/services/order.service.ts`, `packages/core/src/service/helpers/order-calculator/`, `packages/core/src/service/helpers/order-state-machine/`, `packages/core/src/service/helpers/order-modifier/`, `packages/core/src/config/order/`

---

## 3. Customer Management

**Description:** Customer account management including personal information, addresses, customer groups for segmentation, and customer activity history tracking.

**Features:**

| Feature | Description |
|---|---|
| Customer CRUD | Create, update, delete customer accounts with personal details (name, email, phone) |
| Address Management | Add, update, delete customer addresses with country, province, and default address designation |
| Customer Groups | Create and manage customer groups for segmentation; assign/remove customers to/from groups |
| Customer History | Track all customer-related events (account creation, verification, address changes, order placement, notes) |
| Customer-Order Relationship | View all orders associated with a customer |

**Location:** `packages/core/src/entity/customer/`, `packages/core/src/entity/customer-group/`, `packages/core/src/entity/address/`, `packages/core/src/entity/history-entry/`, `packages/core/src/service/services/customer.service.ts`, `packages/core/src/service/services/customer-group.service.ts`, `packages/core/src/service/services/history.service.ts`

---

## 4. Authentication & Authorization

**Description:** Dual authentication system for Admin API and Shop API with pluggable authentication strategies, role-based permissions, session management, email verification, and password reset flows.

**Features:**

| Feature | Description |
|---|---|
| Admin Login/Logout | Authenticate administrators against the Admin API |
| Customer Login/Logout | Authenticate customers against the Shop API |
| Native Authentication | Built-in username/password authentication with bcrypt hashing |
| External Authentication | Pluggable strategy for OAuth/OIDC providers (Keycloak, Google, etc.) with automatic user creation |
| Role Management | Define roles with granular permissions; assign roles to administrators |
| Permission System | Fine-grained permission model (CRUD per entity type + special permissions); enforced via `@Allow()` decorator on resolvers |
| Session Management | Cookie-based or bearer token sessions with pluggable caching strategy |
| Email Verification | Customer email verification flow with configurable token generation |
| Password Reset | Password reset flow with token-based verification |
| Password Policy | Pluggable password validation strategy for enforcing password complexity rules |
| Auth Guard | Middleware that extracts session tokens, validates permissions, and populates `RequestContext` |

**Location:** `packages/core/src/config/auth/`, `packages/core/src/entity/session/`, `packages/core/src/entity/authentication-method/`, `packages/core/src/service/services/auth.service.ts`, `packages/core/src/service/services/session.service.ts`, `packages/core/src/service/services/administrator.service.ts`, `packages/core/src/service/services/role.service.ts`, `packages/core/src/api/middleware/auth-guard.ts`

---

## 5. Payments & Refunds

**Description:** Payment processing with pluggable payment method handlers, a payment state machine (Created → Authorized → Settled → Cancelled/Error), refund processing with its own state machine, and payment method eligibility checking.

**Features:**

| Feature | Description |
|---|---|
| Payment Method Configuration | Configure available payment methods with handlers, eligibility checkers, and channel assignment |
| Payment Creation | Create payments via handler-specific logic (e.g., Stripe Payment Intents, Mollie Orders) |
| Payment State Machine | Track payment lifecycle through states (Created, Authorized, Settled, Declined, Cancelled, Error) with configurable transitions |
| Payment Settlement | Settle authorized payments (capture funds) |
| Payment Cancellation | Cancel pending or authorized payments |
| Refund Processing | Create refunds against settled payments with a separate state machine (Pending → Settled/Failed) |
| Refund Lines | Track which order lines and quantities are being refunded |
| Eligibility Checking | Determine which payment methods are eligible for a given order based on configurable rules |
| Manual Payments | Add manual payment entries to orders |

**Location:** `packages/core/src/entity/payment/`, `packages/core/src/entity/payment-method/`, `packages/core/src/entity/refund/`, `packages/core/src/service/services/payment.service.ts`, `packages/core/src/service/services/payment-method.service.ts`, `packages/core/src/service/helpers/payment-state-machine/`, `packages/core/src/service/helpers/refund-state-machine/`, `packages/core/src/config/payment/`, `packages/core/src/config/refund/`

---

## 6. Shipping & Fulfillment

**Description:** Shipping method configuration with eligibility checking and rate calculation, fulfillment lifecycle management with its own state machine, and shipping line assignment to order lines.

**Features:**

| Feature | Description |
|---|---|
| Shipping Method Configuration | Configure shipping methods with translatable descriptions, eligibility checkers, rate calculators, and fulfillment handlers |
| Shipping Eligibility Checking | Pluggable checkers that determine if a shipping method applies to a given order |
| Shipping Rate Calculation | Pluggable calculators that compute shipping costs based on order contents |
| Shipping Line Assignment | Strategy for assigning shipping lines to order lines (supports multi-line orders) |
| Fulfillment Creation | Create fulfillments from order lines with handler-specific logic |
| Fulfillment State Machine | Track fulfillment lifecycle (Created → Pending → Shipped → Delivered → Cancelled) with configurable transitions |
| Shipping Method Testing | Admin tool to test shipping eligibility and rate calculation against simulated orders |

**Location:** `packages/core/src/entity/shipping-method/`, `packages/core/src/entity/fulfillment/`, `packages/core/src/service/services/shipping-method.service.ts`, `packages/core/src/service/services/fulfillment.service.ts`, `packages/core/src/service/helpers/fulfillment-state-machine/`, `packages/core/src/config/shipping-method/`, `packages/core/src/config/fulfillment/`

---

## 7. Promotions & Discounts

**Description:** Configurable promotion engine with pluggable conditions and actions, coupon code support, and per-customer usage limits.

**Features:**

| Feature | Description |
|---|---|
| Promotion CRUD | Create, update, delete promotions with translatable names/descriptions, start/end dates, enable/disable toggle |
| Promotion Conditions | Pluggable conditions that determine when a promotion applies (5 built-in: min order amount, customer group, has facet values, contains products, buy-X-get-Y-free) |
| Promotion Actions | Pluggable actions that define the discount applied (7 built-in: order/line percentage/fixed discounts, facet-based discounts, free shipping, buy-X-get-Y-free) |
| Coupon Codes | Optional coupon code requirement with per-customer usage limits |
| Entity Duplication | Duplicate promotions with configurable options |
| Channel Assignment | Assign/remove promotions to/from channels for multi-tenancy |

**Location:** `packages/core/src/entity/promotion/`, `packages/core/src/service/services/promotion.service.ts`, `packages/core/src/config/promotion/`, `packages/core/src/config/promotion/actions/`, `packages/core/src/config/promotion/conditions/`

---

## 8. Tax

**Description:** Tax calculation system with configurable tax zones, tax categories, tax rates, and pluggable tax line calculation strategies supporting both inclusive and exclusive pricing.

**Features:**

| Feature | Description |
|---|---|
| Tax Category Management | Create, update, delete tax categories (e.g., "Standard", "Reduced", "Zero") |
| Tax Rate Management | Configure tax rates as percentages linking tax categories to zones; enable/disable individual rates |
| Tax Zone Strategy | Pluggable strategy to determine which tax zone applies (default: channel default zone; alternative: address-based) |
| Tax Line Calculation | Pluggable strategy for computing tax lines on order items (handles inclusive vs. exclusive pricing) |

**Location:** `packages/core/src/entity/tax-category/`, `packages/core/src/entity/tax-rate/`, `packages/core/src/service/services/tax-category.service.ts`, `packages/core/src/service/services/tax-rate.service.ts`, `packages/core/src/config/tax/`

---

## 9. Stock & Inventory

**Description:** Multi-location stock management with stock levels per variant per location, stock movement tracking, and configurable stock allocation strategies.

**Features:**

| Feature | Description |
|---|---|
| Stock Location Management | Create and manage warehouse/stock locations |
| Stock Level Tracking | Track available stock per product variant per stock location |
| Stock Movements | Record all stock changes as typed movements: Allocation, Sale, Release, Cancellation, Adjustment |
| Stock Allocation Strategy | Pluggable strategy determining when stock is allocated during the order process |
| Stock Location Strategy | Pluggable strategy determining which stock location(s) to use for availability and allocation |

**Location:** `packages/core/src/entity/stock-level/`, `packages/core/src/entity/stock-location/`, `packages/core/src/entity/stock-movement/`, `packages/core/src/service/services/stock-level.service.ts`, `packages/core/src/service/services/stock-location.service.ts`, `packages/core/src/service/services/stock-movement.service.ts`

---

## 10. Assets & Media

**Description:** File/image upload and management with pluggable storage strategies, preview generation, and naming conventions.

**Features:**

| Feature | Description |
|---|---|
| Asset Upload | Upload images and files with automatic preview generation |
| Asset Management | List, update, delete assets; manage tags; set focal point for images |
| Asset-Entity Association | Associate assets with products, variants, and collections with ordering support |
| Storage Strategy | Pluggable storage backends (local filesystem default; S3 via asset-server-plugin) |
| Naming Strategy | Pluggable file naming (default, hashed) |
| Preview Strategy | Pluggable preview image generation |
| Asset Import | Import assets from external URLs or file paths with pluggable strategy |

**Location:** `packages/core/src/entity/asset/`, `packages/core/src/service/services/asset.service.ts`, `packages/core/src/config/asset-naming-strategy/`, `packages/core/src/config/asset-storage-strategy/`, `packages/core/src/config/asset-preview-strategy/`, `packages/core/src/config/asset-import-strategy/`

---

## 11. Search

**Description:** Product search with a pluggable architecture. The built-in `DefaultSearchPlugin` provides full-text search using database-specific SQL strategies (MySQL, PostgreSQL, SQLite). Can be replaced by Elasticsearch via plugin.

**Features:**

| Feature | Description |
|---|---|
| Full-Text Product Search | Search products by name, description, and SKU with database-native full-text search |
| Faceted Search | Filter search results by facet values, collection, price range |
| Search Index Management | Background job-based search index with incremental updates on product/variant/collection changes |
| Search Job Buffering | Buffer and de-duplicate search index update jobs for performance |
| Database-Specific Strategies | MySQL (`MATCH...AGAINST`), PostgreSQL (`to_tsvector`), SQLite (`REGEXP`) implementations |
| Pluggable Architecture | `SearchStrategy` interface allowing complete replacement (e.g., Elasticsearch) |

**Location:** `packages/core/src/plugin/default-search-plugin/`, `packages/core/src/service/services/search.service.ts`

---

## 12. Channels, Sellers & Multi-Tenancy

**Description:** Multi-tenant architecture through Channels. Each channel has its own currency, language, tax settings, and product subset. Sellers represent marketplace vendors. Zones and Countries/Provinces provide geographic configuration.

**Features:**

| Feature | Description |
|---|---|
| Channel Management | Create and configure channels with currency, language, default tax/shipping zones, seller assignment |
| Seller Management | Create and manage marketplace sellers; assign sellers to channels |
| Zone Management | Create geographic zones grouping countries/provinces for tax and shipping rules |
| Country & Province Management | Manage countries with ISO codes and provinces/states within countries |
| Multi-Channel Entity Assignment | Assign/remove products, collections, promotions, shipping/payment methods, and other entities to/from channels |
| Active Channel Resolution | Determine the active channel from request headers (`vendure-token`) |

**Location:** `packages/core/src/entity/channel/`, `packages/core/src/entity/seller/`, `packages/core/src/entity/zone/`, `packages/core/src/entity/region/`, `packages/core/src/service/services/channel.service.ts`, `packages/core/src/service/services/seller.service.ts`, `packages/core/src/service/services/zone.service.ts`, `packages/core/src/service/services/country.service.ts`

---

## 13. Global Settings & Tags

**Description:** System-wide configuration, a scoped key-value settings store, and a tagging system for organizing entities.

**Features:**

| Feature | Description |
|---|---|
| Global Settings | Configure available languages, out-of-stock threshold, track-inventory defaults, and global custom fields |
| Settings Store | Per-user or global key-value settings persistence (used by admin UI for preferences like dashboard widget layout) |
| Tag Management | Create and manage tags for organizing products, variants, assets, collections, customers, and orders |

**Location:** `packages/core/src/entity/global-settings/`, `packages/core/src/entity/tag/`, `packages/core/src/entity/settings-store-entry/`, `packages/core/src/service/services/global-settings.service.ts`, `packages/core/src/service/services/tag.service.ts`, `packages/core/src/service/helpers/settings-store/`

---

## 14. Job Queue & Background Processing

**Description:** Producer/consumer job queue for offloading long-running tasks. Supports pluggable backends (in-memory, SQL, BullMQ/Redis, GCP Pub/Sub) and job buffering.

**Features:**

| Feature | Description |
|---|---|
| Job Creation | Queue background jobs from any service with typed data and configurable retries |
| Job Processing | Worker processes consume and execute jobs from the queue |
| Job Monitoring | Admin API queries for listing, filtering, and inspecting job status/data/results |
| Job Cancellation | Cancel running or pending jobs |
| Pluggable Queue Backend | Swap between in-memory, SQL-based polling, BullMQ (Redis), or GCP Pub/Sub |
| Job Buffering | Intercept and aggregate jobs before queue dispatch for batch processing |
| Old Job Cleanup | Scheduled task for cleaning completed/failed jobs after configurable retention |

**Location:** `packages/core/src/job-queue/`, `packages/core/src/plugin/default-job-queue-plugin/`, `packages/core/src/config/job-queue/`

---

## 15. Scheduled Tasks

**Description:** Cron-like task scheduling for recurring background operations with configurable schedule expressions and distributed locking.

**Features:**

| Feature | Description |
|---|---|
| Task Scheduling | Define recurring tasks with cron expressions via builder API |
| Task Execution | Execute tasks on schedule with injector-based dependency access |
| Distributed Locking | Prevent concurrent execution across server instances via pluggable `SchedulerStrategy` |
| Task Monitoring | Admin API for listing scheduled tasks, viewing execution status, and last/next run times |
| Manual Task Triggering | Trigger scheduled tasks on demand via Admin API |
| Task Enable/Disable | Enable or disable individual scheduled tasks at runtime |
| Built-in Tasks | Session cleanup, old job record cleanup, orphaned settings store entry cleanup |

**Location:** `packages/core/src/scheduler/`, `packages/core/src/plugin/default-scheduler-plugin/`

---

## 16. Data Import & Export

**Description:** CSV-based product import system with asset importing, initial data population, and bulk product catalog loading.

**Features:**

| Feature | Description |
|---|---|
| CSV Product Import | Parse and import products/variants from CSV files with facets, options, and assets |
| Asset Import | Import assets from URLs or file paths during product import |
| Initial Data Population | Seed database with countries, zones, tax rates, shipping methods, payment methods, and roles |
| Fast Import Mode | Optimized bulk insert path for large dataset imports |
| Admin API Import | Upload CSV files via Admin API for product import |

**Location:** `packages/core/src/data-import/`

---

## Backend Cross-Cutting Infrastructure

These are not standalone functional areas but shared infrastructure used across all functional areas:

| Area | Description | Location |
|---|---|---|
| **Event System** | RxJS-based EventBus with 60+ event types; non-blocking and blocking modes | `packages/core/src/event-bus/` |
| **Caching** | `CacheService` with pluggable strategies (in-memory, SQL, Redis) + per-request cache | `packages/core/src/cache/` |
| **Custom Fields** | Dynamic field system allowing users to add custom columns to any supported entity at runtime | `packages/core/src/entity/register-custom-entity-fields.ts` |
| **i18n / Localization** | Translatable entity pattern with `*Translation` entities, locale-aware error messages | `packages/core/src/i18n/` |
| **Database Connection** | `TransactionalConnection` wrapper with transaction interceptor and subscriber | `packages/core/src/connection/` |
| **Health Checks** | `/health` endpoint with pluggable strategies (TypeORM, HTTP) | `packages/core/src/health-check/` |
| **Plugin System** | `@VendurePlugin()` decorator extending NestJS modules with API extensions, entities, and config | `packages/core/src/plugin/` |
| **Strategy Pattern** | 50+ `InjectableStrategy` interfaces for all extensible behaviors | `packages/core/src/config/` |
| **Entity Infrastructure** | Base entity (VendureEntity), ID strategy, money strategy, slug strategy, soft-delete | `packages/core/src/entity/base/` |

---

# APPLICATION 2: Angular Admin UI (Legacy)

Location: `packages/admin-ui/src/`

The legacy Angular admin interface organized into 8 lazy-loaded feature modules. Being replaced by the React Dashboard but still fully operational.

---

## 1. Login

**Description:** Admin user authentication screen with auth guard redirect for already-authenticated users.

**Features:**

| Feature | Description |
|---|---|
| Admin Login | Username/password authentication form with session establishment |
| Login Guard | Redirects already-authenticated users to the dashboard |

**Location:** `packages/admin-ui/src/lib/login/src/`

---

## 2. Dashboard (Home)

**Description:** Landing page after login with configurable widgets showing order metrics and recent activity.

**Features:**

| Feature | Description |
|---|---|
| Order Metrics Chart | Time-series chart of order volume and revenue |
| Order Summary Widget | Summary statistics of order counts by state |
| Latest Orders Widget | List of most recent orders with quick navigation |
| Widget Layout Customization | Drag-and-drop widget arrangement with persistent layout |

**Location:** `packages/admin-ui/src/lib/dashboard/src/`

---

## 3. Catalog

**Description:** Product catalog management — the largest feature module with 28 components covering products, variants, facets, collections, and assets.

**Features:**

| Feature | Description |
|---|---|
| Product List & Detail | Browse, search, create, edit, and delete products with translatable fields |
| Product Variant Management | Manage variants with pricing, stock, tax categories, and option values |
| Product Options Editor | Create/edit option groups and generate variant combinations |
| Facet List & Detail | Manage facets and facet values with visibility control |
| Collection List & Detail | Build hierarchical collections with tree view, configurable filters, and drag-and-drop reordering |
| Asset List & Detail | Browse and manage uploaded files/images with preview and tagging |
| Bulk Actions | Assign/remove to channels, assign facet values, duplicate, delete (17 bulk actions total) |
| Channel Assignment Dialogs | Assign products, facets, collections to channels |

**Location:** `packages/admin-ui/src/lib/catalog/src/`

---

## 4. Orders (Sales)

**Description:** Full order lifecycle management with 30 components covering order listing, details, modifications, fulfillments, payments, and refunds.

**Features:**

| Feature | Description |
|---|---|
| Order List | Searchable, filterable order list with state-based filtering |
| Order Detail | Comprehensive order view with line items, addresses, history, and state transitions |
| Draft Order Creation | Create orders manually with customer, address, and line item selection |
| Order Modification | Edit existing orders (add/remove lines, adjust quantities, change addresses) with preview |
| Fulfillment Management | Create fulfillments, track shipment state, view fulfillment details |
| Payment Management | View payments, add manual payments, track payment state transitions |
| Refund Processing | Create refunds against payments, settle refunds, view refund details |
| Order State Visualization | Interactive order process graph showing all states and transitions |
| Order History | Timestamped timeline of all order events and notes |
| Seller Orders | View aggregate/seller order relationships in multi-vendor scenarios |

**Location:** `packages/admin-ui/src/lib/order/src/`

---

## 5. Customers

**Description:** Customer account and group management with address handling and customer history.

**Features:**

| Feature | Description |
|---|---|
| Customer List & Detail | Browse, search, create, edit customers with status badges |
| Address Management | Add, edit, delete customer addresses with form dialogs |
| Customer Groups | Create, edit, delete customer groups; add/remove members |
| Customer History | Timeline of customer events (account creation, orders, notes) |
| Bulk Actions | Delete customers, delete groups, remove group members |

**Location:** `packages/admin-ui/src/lib/customer/src/`

---

## 6. Marketing

**Description:** Promotional offers and discount rule management.

**Features:**

| Feature | Description |
|---|---|
| Promotion List & Detail | Browse, create, edit promotions with conditions and actions |
| Condition/Action Configuration | Configure pluggable promotion conditions and actions with dynamic form inputs |
| Bulk Actions | Assign/remove to channels, duplicate, delete promotions |

**Location:** `packages/admin-ui/src/lib/marketing/src/`

---

## 7. Settings

**Description:** The most route-rich module (27 routes) covering all administrative configuration across 13 sub-areas.

**Features:**

| Feature | Description |
|---|---|
| Administrator Management | List, create, edit administrators with role assignment |
| Role Management | Define roles with granular permissions per entity type |
| Profile Editor | Edit the current admin user's personal details and password |
| Channel Configuration | Create and configure channels with currency, language, zones, and seller |
| Seller Management | List, create, edit sellers for marketplace support |
| Stock Location Management | Manage warehouse/stock locations with channel assignment |
| Shipping Method Configuration | Configure shipping methods with eligibility checkers and rate calculators; includes shipping test tool |
| Payment Method Configuration | Configure payment methods with handlers and eligibility checkers |
| Tax Category Management | Create and manage tax categories |
| Tax Rate Management | Configure tax rates linking categories to zones |
| Country Management | Manage countries with ISO codes and enable/disable |
| Zone Management | Create zones and manage member countries/regions |
| Global Settings | Configure system-wide settings (languages, stock threshold, custom fields) |

**Location:** `packages/admin-ui/src/lib/settings/src/`

---

## 8. System

**Description:** System monitoring and operational tools for background processes.

**Features:**

| Feature | Description |
|---|---|
| Job Queue Monitoring | List and filter background jobs by state/queue with status badges |
| System Health Check | View server health status from the `/health` endpoint |
| Scheduled Task List | View configured scheduled tasks with execution status |

**Location:** `packages/admin-ui/src/lib/system/src/`

---

## Angular Admin UI Cross-Cutting Infrastructure

| Area | Description | Location |
|---|---|---|
| **Core Module** | App shell, navigation, breadcrumbs, channel switcher, theme switcher, notifications | `packages/admin-ui/src/lib/core/src/` |
| **Data Module** | Apollo GraphQL client with domain-specific data services and auth interceptor | `packages/admin-ui/src/lib/core/src/data/` |
| **Shared Module** | 100+ reusable components, 26 dynamic form inputs, pipes, directives, dialogs | `packages/admin-ui/src/lib/core/src/shared/` |
| **Extension System** | APIs for adding nav items, page tabs, bulk actions, form inputs, detail components, dashboard widgets, action bar items | `packages/admin-ui/src/lib/core/src/extension/` |
| **Route Guards** | `AuthGuard`, `LoginGuard`, `OrderGuard`, `CanDeactivateDetailGuard` | `packages/admin-ui/src/lib/core/src/providers/guard/` |
| **PageService** | Dynamic page tab registration enabling plugins to add tabs to any page | `packages/admin-ui/src/lib/core/src/providers/page/` |

---

# APPLICATION 3: React Dashboard (New)

Location: `packages/dashboard/src/`

The new React-based admin dashboard using TanStack Router (file-based routing), TanStack Query, Radix UI, and Tailwind CSS. Organized into 7 navigation sections with 26 distinct pages.

---

## 1. Insights (Dashboard Home)

**Description:** Main landing page with configurable dashboard widgets in a responsive grid layout.

**Features:**

| Feature | Description |
|---|---|
| Dashboard Widgets | Configurable grid of widgets (order metrics, order summary, latest orders) |
| Date Range Filtering | Filter widget data by date range |
| Widget Layout Persistence | Drag-and-drop widget repositioning with layout saved to user settings |
| Custom Widgets | Extension API for adding custom dashboard widgets |

**Location:** `packages/dashboard/src/app/routes/_authenticated/index.tsx`, `packages/dashboard/src/lib/framework/dashboard-widget/`

---

## 2. Products

**Description:** Product catalog management with variant generation, option group editing, and rich text content.

**Features:**

| Feature | Description |
|---|---|
| Product List | Searchable list (name/slug/SKU) with bulk actions (assign channels, assign facets, duplicate, delete) |
| Product Detail | Edit product with translatable name/slug/description, variant management, option groups, facet values, assets, channel assignment |
| Variant Generation | Create product variants from option group combinations |
| Option Group & Option Detail | Edit product option groups and individual option values |
| Bulk Channel Assignment | Assign/remove products to/from channels in bulk |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_products/`

---

## 3. Product Variants

**Description:** Standalone variant management with multi-currency pricing and multi-location stock tracking.

**Features:**

| Feature | Description |
|---|---|
| Variant List | Searchable variant list (name/SKU) with price and stock display |
| Variant Detail | Edit variant with multi-currency pricing (add/remove currencies), stock levels per location, tax category, facet values |
| Bulk Actions | Assign/remove channels, assign facet values, delete |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_product-variants/`

---

## 4. Facets

**Description:** Facet management with inline facet value display and expandable detail sheets.

**Features:**

| Feature | Description |
|---|---|
| Facet List | Browse facets with visibility badges, inline facet values, bulk actions |
| Facet Detail | Edit facet with translatable name, code, private toggle, embedded facet values table |
| Facet Value Detail | Edit individual facet values with translatable name and code |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_facets/`

---

## 5. Collections

**Description:** Hierarchical collection management with tree view, drag-and-drop, and configurable filters.

**Features:**

| Feature | Description |
|---|---|
| Collection Tree List | Expandable hierarchy with drag-and-drop reordering and circular reference detection |
| Collection Detail | Edit with translatable fields, configurable filters with inheritance, content preview, assets |
| Collection Moving | Move collections between parents with validation |
| Bulk Actions | Assign/remove channels, duplicate, move, delete |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_collections/`

---

## 6. Assets

**Description:** Digital asset management with gallery view, tagging, and focal point editing.

**Features:**

| Feature | Description |
|---|---|
| Asset Gallery | Grid display of assets with selectable items and pagination |
| Asset Detail | Image preview with focal point editor, size selector, tag management, properties display |
| Bulk Delete | Select and delete multiple assets |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_assets/`

---

## 7. Orders

**Description:** Comprehensive order management with state transitions, fulfillments, payments, refunds, and draft order creation.

**Features:**

| Feature | Description |
|---|---|
| Order List | Search by code/customer/transaction ID with state-based faceted filter and draft order creation |
| Order Detail | View order with line items, payments, fulfillments, history, state transitions |
| Draft Order Editor | Create orders with customer selection, address selection, line items, shipping, coupon codes |
| Order Modification | Modify existing orders with line item editing, address changes, surcharges, and preview |
| Seller Order Detail | View seller sub-orders in multi-vendor scenarios |
| Fulfillment Management | View fulfillment details and state |
| Payment Management | View payment details, add manual payments |
| Refund Processing | Create refunds via dialog |
| Order History | Timeline of all order events |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_orders/`

---

## 8. Customers

**Description:** Customer management with addresses, order history, and group assignment.

**Features:**

| Feature | Description |
|---|---|
| Customer List | Searchable list (name/email/phone) with status badges and group display |
| Customer Detail | Edit personal info, manage addresses, view order history, customer history timeline, manage group membership |
| Bulk Delete | Delete selected customers |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_customers/`

---

## 9. Customer Groups

**Description:** Customer group management for segmentation.

**Features:**

| Feature | Description |
|---|---|
| Customer Group List | Browse groups with member count and search |
| Customer Group Detail | Edit group name, manage group members via embedded table |
| Bulk Delete | Delete selected customer groups |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_customer-groups/`

---

## 10. Promotions

**Description:** Marketing promotion management with coupon codes and scheduling.

**Features:**

| Feature | Description |
|---|---|
| Promotion List | Searchable list (name/coupon code) with enabled badge and date columns |
| Promotion Detail | Edit promotion settings |
| Bulk Actions | Assign/remove channels, duplicate, delete |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_promotions/`

---

## 11. Administrators

**Description:** Admin user management.

**Features:**

| Feature | Description |
|---|---|
| Administrator List | Search by name/email with role badges |
| Administrator Detail | Edit admin user details and role assignments |
| Bulk Delete | Delete selected administrators |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_administrators/`

---

## 12. Roles

**Description:** Role-based access control management.

**Features:**

| Feature | Description |
|---|---|
| Role List | Browse roles with expandable permissions and channel badges |
| Role Detail | Edit role permissions and channel assignments |
| Bulk Delete | Delete non-system roles |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_roles/`

---

## 13. Channels

**Description:** Multi-tenancy channel management.

**Features:**

| Feature | Description |
|---|---|
| Channel List | Browse channels with seller, language, currency display |
| Channel Detail | Edit channel configuration |
| Bulk Delete | Delete selected channels |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_channels/`

---

## 14. Sellers

**Description:** Marketplace seller management.

**Features:**

| Feature | Description |
|---|---|
| Seller List | Browse and search sellers |
| Seller Detail | Edit seller details |
| Bulk Delete | Delete selected sellers |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_sellers/`

---

## 15. Stock Locations

**Description:** Warehouse/stock location management.

**Features:**

| Feature | Description |
|---|---|
| Stock Location List | Browse and search stock locations |
| Stock Location Detail | Edit stock location details |
| Bulk Actions | Assign/remove channels, delete |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_stock-locations/`

---

## 16. Shipping Methods

**Description:** Shipping method configuration with testing tool.

**Features:**

| Feature | Description |
|---|---|
| Shipping Method List | Browse methods with fulfillment handler display |
| Shipping Method Detail | Edit shipping method configuration |
| Shipping Method Test Tool | Test eligibility and rate calculation via slide-out sheet |
| Bulk Actions | Assign/remove channels, delete |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_shipping-methods/`

---

## 17. Payment Methods

**Description:** Payment method configuration.

**Features:**

| Feature | Description |
|---|---|
| Payment Method List | Browse methods with enabled/disabled filter |
| Payment Method Detail | Edit payment method configuration |
| Bulk Actions | Assign/remove channels, delete |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_payment-methods/`

---

## 18. Tax Configuration

**Description:** Tax categories and tax rates management.

**Features:**

| Feature | Description |
|---|---|
| Tax Category List & Detail | Manage tax categories with default designation |
| Tax Rate List & Detail | Configure tax rates with percentage, category, zone, and enabled status; faceted filtering |
| Bulk Delete | Delete selected categories/rates |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_tax-categories/`, `packages/dashboard/src/app/routes/_authenticated/_tax-rates/`

---

## 19. Geographic Configuration

**Description:** Country and zone management for tax and shipping rules.

**Features:**

| Feature | Description |
|---|---|
| Country List & Detail | Manage countries with ISO codes |
| Zone List & Detail | Manage zones with member countries via slide-out sheet |
| Bulk Delete | Delete selected countries/zones |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_countries/`, `packages/dashboard/src/app/routes/_authenticated/_zones/`

---

## 20. Global Settings

**Description:** System-wide configuration.

**Features:**

| Feature | Description |
|---|---|
| Global Settings Editor | Configure available languages, global out-of-stock threshold, track inventory defaults, custom fields |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_global-settings/`

---

## 21. Profile

**Description:** Currently logged-in administrator's profile management.

**Features:**

| Feature | Description |
|---|---|
| Profile Editor | Edit personal name, email, and password |
| Authentication Methods Display | View configured authentication methods (native, external) |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_profile/`

---

## 22. System Monitoring

**Description:** System monitoring tools for background processes and server health.

**Features:**

| Feature | Description |
|---|---|
| Job Queue Monitoring | List and filter jobs with auto-refresh, state filtering, payload inspection, job cancellation |
| Health Checks | Server health monitoring with 5-second auto-refresh and per-resource status cards |
| Scheduled Task Management | View, enable/disable, and manually trigger scheduled tasks with result inspection |

**Location:** `packages/dashboard/src/app/routes/_authenticated/_system/`

---

## React Dashboard Cross-Cutting Infrastructure

| Area | Description | Location |
|---|---|---|
| **Layout System** | AppLayout, AppSidebar, NavMain, NavUser, ChannelSwitcher | `packages/dashboard/src/lib/components/layout/` |
| **Framework Layer** | ListPage, useDetailPage, PageLayout engine, NavMenu config, Global Registry | `packages/dashboard/src/lib/framework/` |
| **Extension API** | `defineDashboardExtension()` for routes, nav items, page blocks, widgets, form components, data table columns | `packages/dashboard/src/lib/framework/extension-api/` |
| **Data Components** | DataTable, MoneyInput, RichTextInput, SlugInput, DatetimeInput, RelationInput | `packages/dashboard/src/lib/components/data-input/`, `packages/dashboard/src/lib/components/data-display/` |
| **Providers** | Auth, Channel, ServerConfig, Theme, UserSettings, i18n, Alerts | `packages/dashboard/src/lib/providers/` |
| **GraphQL Client** | awesome-graphql-client with gql.tada for type-safe queries | `packages/dashboard/src/lib/graphql/` |

---

# APPLICATION 4: CLI Tool (`@vendure/cli`)

Location: `packages/cli/src/`

A command-line tool for managing existing Vendure projects, providing code generation and database management capabilities.

---

## 1. Code Generation (`vendure add`)

**Description:** Interactive code scaffolding for extending Vendure projects with new plugins, entities, services, API extensions, and UI components.

**Features:**

| Feature | Description |
|---|---|
| Plugin Scaffolding | Create a new plugin with constants, types, and plugin template files |
| Entity Generation | Add entities to a plugin with support for custom fields, translatable entities, and TypeORM decorators |
| Service Generation | Generate services (basic or entity-typed) with dependency injection and plugin registration |
| API Extension Generation | Add GraphQL queries/mutations with simple or full CRUD resolvers |
| Job Queue Integration | Add job queue support to an existing plugin service |
| Dashboard Extension | Scaffold React dashboard extensions for a plugin |
| Angular UI Extensions | Scaffold legacy Angular UI extensions (deprecated) |
| GraphQL Codegen Setup | Configure GraphQL code generation for a project |
| Interactive Mode | Menu-driven selection when run without flags |

**Location:** `packages/cli/src/commands/add/`

---

## 2. Database Migration Management (`vendure migrate`)

**Description:** Database schema migration tools using TypeORM migrations.

**Features:**

| Feature | Description |
|---|---|
| Generate Migration | Generate a new migration file from schema changes |
| Run Migrations | Execute all pending migrations |
| Revert Migration | Revert the most recent migration |

**Location:** `packages/cli/src/commands/migrate/`

---

## 3. Schema Generation (`vendure schema`)

**Description:** Export GraphQL schema files for Admin or Shop APIs.

**Features:**

| Feature | Description |
|---|---|
| Schema Export | Generate Admin or Shop API schema in SDL or JSON format |
| Configurable Output | Custom output directory and file name |

**Location:** `packages/cli/src/commands/schema/`

---

# APPLICATION 5: Project Scaffolding Tool (`@vendure/create`)

Location: `packages/create/src/`

A project generator invoked via `npm create @vendure` that creates new Vendure projects from templates.

---

## 1. Project Scaffolding

**Description:** Interactive project generator that creates a complete Vendure project with database configuration, email templates, and optionally a Next.js storefront.

**Features:**

| Feature | Description |
|---|---|
| Quick Start Mode | Auto-detects Docker; uses PostgreSQL (with Docker) or SQLite (without Docker); populates sample data |
| Manual Configuration Mode | Interactive prompts for database type, connection details, superadmin credentials, product population |
| CI Mode | Non-interactive SQLite setup with defaults for CI/CD pipelines |
| Database Support | MySQL, MariaDB, PostgreSQL, SQLite with driver-specific configuration |
| Next.js Storefront | Optional monorepo setup with apps/server + apps/storefront |
| Template Generation | Handlebars-based templates for vendure-config, .env, Dockerfile, docker-compose, tsconfig, vite.config |
| Email Template Setup | Copy email templates from `@vendure/email-plugin` |
| Sample Data Population | Optional product catalog seeding |
| Docker Container Management | Auto-start PostgreSQL Docker container in Quick Start mode |

**Location:** `packages/create/src/`

---

# OFFICIAL PLUGINS

Plugins extend the backend with specialized functionality. Each is an independent npm package.

---

## Plugin 1: Asset Server (`@vendure/asset-server-plugin`)

**Description:** Serves asset files with on-the-fly image transformation via Sharp, supporting local filesystem and S3/MinIO storage.

**Features:**

| Feature | Description |
|---|---|
| Asset Serving | Serve uploaded assets via configurable Express route |
| Image Transformation | On-the-fly resize, crop, format conversion (JPEG/PNG/WebP/AVIF), quality adjustment |
| Focal Point Cropping | Crop images around a focal point set in the admin UI |
| Named Transform Presets | Pre-defined transformation presets (tiny, thumb, small, medium, large + custom) |
| Transformation Caching | Cache transformed images for performance; bypass with `?cache=false` |
| S3/MinIO Storage | Store assets in AWS S3 or MinIO with configurable bucket, prefix, and endpoint |
| Local Filesystem Storage | Default storage on the server's local filesystem |
| Transform Limiting | Restrict allowed transformations to presets only (`PresetOnlyStrategy`) |
| Configurable Cache Headers | Set `Cache-Control` headers for served assets |
| Dynamic URL Prefix | Static or per-request asset URL prefix for CDN integration |

**Location:** `packages/asset-server-plugin/src/`

---

## Plugin 2: Email (`@vendure/email-plugin`)

**Description:** Event-driven transactional email system with MJML templates, Handlebars rendering, and Nodemailer transport.

**Features:**

| Feature | Description |
|---|---|
| Event-Driven Email Triggers | Fire emails in response to Vendure events (order confirmation, email verification, password reset, email change) |
| MJML + Handlebars Templates | Responsive email templates with dynamic data injection |
| Multiple Transport Options | SMTP, AWS SES, Sendmail, File (dev), Testing, Noop |
| Dynamic Transport Selection | Choose transport based on channel or request context |
| Custom Event Handlers | Define new email triggers for any Vendure event |
| Template Loading | Pluggable template source (default: file-based; custom loaders supported) |
| Email Attachments | Attach files to outgoing emails |
| Global Template Variables | Static or async variables available to all templates |
| Development Mailbox | Web-based email viewer at configurable route for dev/testing |
| Async Queue Processing | Email sending via background job queue with retries |

**Location:** `packages/email-plugin/src/`

---

## Plugin 3: Elasticsearch (`@vendure/elasticsearch-plugin`)

**Description:** High-performance product search via Elasticsearch 7.x, replacing the default database search.

**Features:**

| Feature | Description |
|---|---|
| Full-Text Product Search | Elasticsearch-powered product search with relevance scoring |
| Real-Time Index Sync | Automatic index updates on product, variant, collection, stock, and tax changes |
| Price Range Aggregation | Aggregated price range data in search results |
| Custom Field Mappings | Map custom fields to Elasticsearch index fields |
| Script Fields | Computed search result values via Elasticsearch scripts |
| Health Check Integration | Elasticsearch health status in the system health check endpoint |
| Buffered Updates | Optional batched index updates for high-throughput scenarios |

**Location:** `packages/elasticsearch-plugin/src/`

---

## Plugin 4: Job Queue (`@vendure/job-queue-plugin`)

**Description:** Production-grade job queue backends for Redis (BullMQ) and Google Cloud Pub/Sub.

**Features:**

| Feature | Description |
|---|---|
| BullMQ/Redis Backend | Push-based job queue using Redis with BullMQ |
| Worker Concurrency | Configurable concurrent job processing (default: 3) |
| Job Priority | Priority-based queue ordering |
| Old Job Cleanup | Configurable retention for completed/failed jobs |
| Redis Health Check | Monitor Redis connection health |
| Redis Job Buffer Storage | Store job buffer data in Redis |
| GCP Pub/Sub Backend | Google Cloud Pub/Sub-based job queue |
| Per-Queue Options | Custom BullMQ options per queue name |

**Location:** `packages/job-queue-plugin/src/`

---

## Plugin 5: Payments (`@vendure/payments-plugin`)

**Description:** Payment provider integrations for Stripe, Mollie, and Braintree.

**Features:**

| Feature | Description |
|---|---|
| **Stripe Integration** | |
| Stripe Payment Intents | Create and manage Payment Intents via Stripe API |
| Stripe Webhooks | Receive `payment_intent.succeeded` and `payment_intent.payment_failed` events |
| Stripe Customer Storage | Optional customer vaulting with `stripeCustomerId` custom field |
| **Mollie Integration** | |
| Mollie Order API | Create and manage Mollie Orders (not Payments) |
| Mollie Webhooks | Receive payment status updates from Mollie |
| Pay-Later Methods | Support for Klarna and similar pay-later methods |
| **Braintree Integration** | |
| Client Token Generation | Generate Braintree client tokens for Drop-in UI |
| Nonce-Based Payments | Synchronous payment processing without webhooks |
| Payment Vaulting | Optional customer vaulting via `braintreeCustomerId` custom field |

**Location:** `packages/payments-plugin/src/`

---

## Plugin 6: GraphiQL (`@vendure/graphiql-plugin`)

**Description:** Embedded GraphiQL IDE for exploring and testing GraphQL APIs.

**Features:**

| Feature | Description |
|---|---|
| Admin API Explorer | GraphiQL IDE at `/graphiql/admin` for the Admin API |
| Shop API Explorer | GraphiQL IDE at `/graphiql/shop` for the Shop API |
| Pre-Populated Queries | Support for `?query=` parameter with pre-filled queries |
| Embedded Mode | Support for iframe embedding via `?embeddedMode=true` |

**Location:** `packages/graphiql-plugin/src/`

---

## Plugin 7: Harden (`@vendure/harden-plugin`)

**Description:** Security hardening for production deployments.

**Features:**

| Feature | Description |
|---|---|
| Query Complexity Analysis | Limit GraphQL query complexity with configurable maximum (default: 1,000) |
| Custom Complexity Factors | Override complexity for specific fields (e.g., deeply nested relations) |
| Schema Sniffing Prevention | Hide field suggestions in validation error messages |
| Production API Mode | Disable introspection, playground, and debug mode |
| Complexity Logging | Log complexity scores for tuning |

**Location:** `packages/harden-plugin/src/`

---

## Plugin 8: Sentry (`@vendure/sentry-plugin`)

**Description:** Sentry error tracking and performance monitoring integration.

**Features:**

| Feature | Description |
|---|---|
| Automatic Error Capture | Capture unhandled exceptions to Sentry |
| Distributed Tracing | Performance tracing via Sentry SDK |
| Profile Sampling | Continuous profiling with configurable sample rate |
| Console Log Capture | Send console logs to Sentry with configurable log levels |
| Test Mutation | Optional `createTestError` mutation for verifying Sentry configuration |

**Location:** `packages/sentry-plugin/src/`

---

## Plugin 9: Stellate (`@vendure/stellate-plugin`)

**Description:** GraphQL edge caching integration with Stellate (formerly GraphCDN).

**Features:**

| Feature | Description |
|---|---|
| Event-Driven Cache Purging | Automatically purge cached data on entity changes |
| Built-In Purge Rules | 8 default rules for products, variants, collections, stock, tax rates, and channel assignments |
| Custom Purge Rules | Create custom purge rules for user-defined entity types |
| Buffered Purging | Debounced batch processing of purge events |
| Dev Mode | Prevent actual API calls in development with optional debug logging |

**Location:** `packages/stellate-plugin/src/`

---

## Plugin 10: Telemetry (`@vendure/telemetry-plugin`)

**Description:** OpenTelemetry instrumentation for distributed tracing and structured logging.

**Features:**

| Feature | Description |
|---|---|
| Distributed Tracing | Instrument Vendure services with OpenTelemetry trace spans |
| Structured Logging | OTLP-based structured log export to Loki and other backends |
| Method Hooks | Pre/post hooks on service methods for custom telemetry (experimental) |
| Configurable Exporters | Support for OTLP trace and log exporters (Jaeger, Loki, etc.) |
| Console Log Level | Separate console log level alongside OTLP logging |

**Location:** `packages/telemetry-plugin/src/`

---

# SUMMARY

## Functional Area Count by Application

| Application | Functional Areas | Total Features |
|---|---|---|
| Backend (`@vendure/core`) | 16 + cross-cutting | ~85 |
| Angular Admin UI (legacy) | 8 + infrastructure | ~40 |
| React Dashboard (new) | 22 + infrastructure | ~65 |
| CLI Tool | 3 | ~15 |
| Create Tool | 1 | ~9 |
| Official Plugins (10) | 10 | ~55 |
| **Total** | **~60** | **~269** |

## Backend ↔ Frontend Feature Mapping

The Angular Admin UI and React Dashboard both cover the same backend functional areas but with different UI implementations:

| Backend Area | Angular Admin UI | React Dashboard |
|---|---|---|
| Catalog Management | Catalog module (products, variants, facets, collections, assets) | Products, Product Variants, Facets, Collections, Assets (separate route groups) |
| Order Management | Orders module | Orders |
| Customer Management | Customers module (customers + groups) | Customers, Customer Groups (separate route groups) |
| Authentication & Authorization | Login module + Settings (admins, roles) | Login page + Administrators, Roles |
| Payments & Refunds | Within Orders + Settings (payment methods) | Within Orders + Payment Methods |
| Shipping & Fulfillment | Within Orders + Settings (shipping methods) | Within Orders + Shipping Methods |
| Promotions & Discounts | Marketing module | Promotions |
| Tax | Settings (tax categories, tax rates) | Tax Categories, Tax Rates |
| Stock & Inventory | Settings (stock locations) | Stock Locations |
| Assets & Media | Within Catalog | Assets |
| Channels & Multi-Tenancy | Settings (channels, sellers, zones, countries) | Channels, Sellers, Zones, Countries |
| Global Settings | Settings | Global Settings |
| Job Queue | System | System (Job Queue) |
| Scheduled Tasks | System | System (Scheduled Tasks) |
| Health Checks | System | System (Healthchecks) |
| Dashboard/Insights | Dashboard module | Insights |
| Profile | Settings (profile) | Profile |
