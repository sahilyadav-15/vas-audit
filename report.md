# Vastriqo Migration — Progress Report

## Overall Status

Phase 1 — Target Architecture & Requirements is complete, and all Phase 1 business decisions are approved. Phase 2A — Technical Foundation & Product Schema Design is complete and awaiting review. The Product/Catalog technical foundation is implementation-ready subject to the stated production validations, but neither target repository is authorized for initialization. No application implementation has started.

## Current Architecture

`vastriqo` is a Next.js storefront that uses Shopify directly for catalog, product, cart, and checkout behavior, and uses `bms-api` for customer bridging, addresses, orders, wishlists, custom collections, inquiries, and newsletter features. `bms-api` and the BMS database also serve CCA operations. There is no dedicated Vastriqo API, database, or admin application yet.

## Target Architecture

`vastriqo.in` and the independent `vastriqo-admin` application at `admin.vastriqo.in` will use `vastriqo-api`, backed by a dedicated Vastriqo database. The final core commerce path will not depend on Shopify, `bms-api`, or the BMS database. CCA/BMS will remain independently deployable.

## Phase Status

### Phase 0 — Current-System Audit
Status: Complete

### Phase 1 — Target Architecture & Requirements
Status: Complete

### Phase 2 — Detailed Data & API Design
Status: In Progress — Phase 2A design complete; Phase 2B is next

### Phase 3 — Build Vastriqo API & Database Foundation
Status: Not Started

### Phase 4 — Build Vastriqo Admin
Status: Not Started

### Phase 5 — Migrate Storefront
Status: Not Started

### Phase 6 — Data Migration
Status: Not Started

### Phase 7 — External Commerce Integrations
Status: Not Started

### Phase 8 — Validation, Parallel Operation & Cutover
Status: Not Started

### Phase 9 — Shopify Removal
Status: Not Started

## Completed Work

- Audited `bms-admin`, `bms-api`, `vastriqo`, their data ownership, integrations, Shopify dependencies, and relevant technical debt.
- Documented the target direction, system boundaries, migration phases, and definition of success.
- Confirmed that Shopify currently owns purchase-critical catalog, customer, address, cart, checkout/payment, order, and fulfillment capabilities.
- Identified BMS-owned Vastriqo-adjacent behavior that must be evaluated for migration, including collections, wishlists, inquiries, newsletter, wallet/referrals, and storefront configuration.
- Completed `PHASE-1-REQUIREMENTS-AND-FEATURE-DISPOSITION.md`, classifying audited behavior and dependency exit requirements.
- Completed `PRODUCT-CATALOG-DESIGN.md`, including the product/variant/media/attribute conceptual slice and source mapping.
- Completed `PHASE-1-CONCEPTUAL-ARCHITECTURE.md` across catalog, merchandising, inventory, identity, wishlist, cart, checkout, orders, provider boundaries, admin, migration, API, and database design.
- Completed `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`, reconciling all earlier unresolved items and closing the Phase 1 gate.
- Completed `PHASE-2A-TECHNICAL-FOUNDATION-AND-PRODUCT-SCHEMA.md`, selecting the API/admin/database foundations and defining the implementation-ready physical Product/Catalog, minimum Inventory, merchandising projection, handle and import-control schema.
- Resolved the target as a strict-TypeScript modular monolith with a dedicated transactional relational database, native Vastriqo identities, separate public/customer/admin/integration boundaries, deterministic imports, and no final Shopify/BMS core dependency.
- Created this living progress report for implementation status and next-step tracking.

## Current Decisions

- `vastriqo-api` and `vastriqo-admin` will be independent applications with separate repositories, configuration, deployments, and API contracts.
- Vastriqo will use a dedicated database; the BMS database will not become its commerce database.
- The final Vastriqo runtime will not rely on Shopify, `bms-api`, or the BMS database for core commerce.
- CCA-specific products, production orders, users/roles, invoices, seller details, and finance workflows will remain outside the Vastriqo domain.
- Existing BMS engineering conventions may inform the new API, but code, schema, authentication, and runtime dependencies will not be copied automatically.
- `vastriqo-admin` and `bms-admin` may link to each other as separate applications; shared SSO is optional and deferred until explicitly designed.
- The migration will preserve appropriate storefront behavior and remove dependencies progressively, with validation before cutover.
- The new customer system starts clean: current Shopify/BMS test customers, credentials, addresses and wishlists will not migrate or link.
- Current test orders will not migrate and will not receive a dedicated historical archive; native order history starts with new Vastriqo orders.
- Current BMS newsletter subscribers and support/inquiry records will not migrate.
- Cancellation, refund, return, exchange and customer-visible tracking remain required future capabilities, designed progressively in the relevant post-purchase phase rather than copied from broken current behavior.
- Product/Catalog remains the first migration and implementation vertical, including required variants, options, attributes/specifications, media, pricing, publication, inventory/availability and collection/merchandising data.
- `vastriqo-api` will use Node.js 24 LTS, TypeScript 6, NestJS 11 on Express 5, PostgreSQL 18, Kysely, native ESM, OpenAPI, structured logging and real-PostgreSQL integration tests.
- `vastriqo-admin` will use Next.js 16 App Router, React 19, TypeScript 6, Ant Design 6, TanStack Query 5, generated OpenAPI clients, React Hook Form/Zod and a same-origin BFF/session boundary.
- Native money is stored as signed 64-bit minor units plus ISO currency; public/admin decimal values cross the API as strings with explicit currency.
- Native UUIDv7 identifiers, a collision-free Product handle/alias namespace, typed external-source mappings, strict aggregate invariants and location-aware inventory replace Shopify/BMS identity and schema coupling.

## Approved Business Decisions and Remaining Validation

The authoritative register is `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`. No Phase 1 business decision remains unresolved. Existing test-era customers, addresses, wishlists, orders, inquiries and newsletter records are explicitly outside migration scope. No broad legal/financial historical migration is required.

Production validation has **not** been completed. Product/catalog, collection, inventory and PostgreSQL deployment inspections remain scheduled checkpoints. Phase 2A has now resolved the framework/database baseline, identifiers, money representation, Product/Catalog schema, admin foundation, testing/observability conventions and initial session boundary. Hosting/provider choices and detailed legal/tax/privacy/consent and post-purchase workflows remain for their relevant later design checkpoints.

## Implementation Gates

- Phase 2A is ready for review; its technical foundation is sufficient to initialize `vastriqo-api` after explicit approval and authorization.
- The separate `vastriqo-api` and `vastriqo-admin` repositories do not exist; this is intentional. This documentation task does not authorize either repository to be created.
- `vastriqo-admin` initialization should wait for Phase 2B's reviewed Product OpenAPI and admin-session contracts so its initial structure is contract-driven.
- Production validations in the Phase 2A design §21 must complete before affected schema/import constraints are frozen. They are not permission to read or modify production data during this task.

## Next Step

- Review and approve Phase 2A, then execute Phase 2B Product/Catalog contract and delivery design: versioned OpenAPI contracts, command/concurrency sequences, field-level import/reconciliation rules, media contract, admin workflows, session/CSRF/permission threat model, acceptance tests and authorized read-only production validation. Do not initialize either target repository or implement application code without separate authorization.

## Change Log

### 2026-08-24

- Created the migration progress report.
- Recorded Phase 0 as complete and Phase 1 as in progress.
- Summarized current and target architecture, confirmed decisions, implementation-readiness gaps, blockers, and the next step.

### 2026-08-25

- Completed the Phase 1 requirements disposition, product/catalog conceptual design, and full commerce conceptual architecture.
- Recorded exact `vastriqo-api` and `vastriqo-admin` initialization prerequisites.
- Marked Phase 1 engineering work complete with business approval and data-validation gates still pending.

### 2026-08-26

- Reclassified every unresolved item as already decided, engineering decision, production-data validation, genuine business decision or safe deferral.
- Reconciled stale questions and readiness statements across the Phase 1 documents.
- Closed Phase 1 and marked Phase 2 detailed design ready to start without authorizing application/repository initialization.
- Recorded the final clean-start business decisions: no migration of current test customers, credentials, addresses, wishlists, orders, inquiries or newsletter records; no dedicated test-order archive; and no broad legal/financial history migration.
- Confirmed required future post-purchase capabilities remain in scope for progressive later-phase design and that Product/Catalog remains the first implementation vertical.
- Completed Phase 2A technical design without creating repositories, code, migrations, packages, databases or external configuration.
- Selected the API/admin/database/tooling baseline and documented why useful current-repository conventions are retained while BMS coupling and unsafe legacy patterns are rejected.
- Defined the physical Product/Catalog, Collection, minimum Inventory, projection, handle and deterministic import-control schema, including constraints, indexes, transactions, read/write paths and admin ownership.
- Recorded remaining production validations, bounded schema risks, deferred choices, the `vastriqo-api` review/authorization gate and the additional Phase 2B contract prerequisite for `vastriqo-admin`.
