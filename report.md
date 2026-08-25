# Vastriqo Migration — Progress Report

## Overall Status

Phase 1 — Target Architecture & Requirements has an engineering-complete conceptual architecture. Business approval, production source-data validation, and repository-initialization ADRs remain open, so Phase 1 is not yet approved and neither target application should be initialized.

## Current Architecture

`vastriqo` is a Next.js storefront that uses Shopify directly for catalog, product, cart, and checkout behavior, and uses `bms-api` for customer bridging, addresses, orders, wishlists, custom collections, inquiries, and newsletter features. `bms-api` and the BMS database also serve CCA operations. There is no dedicated Vastriqo API, database, or admin application yet.

## Target Architecture

`vastriqo.in` and the independent `vastriqo-admin` application at `admin.vastriqo.in` will use `vastriqo-api`, backed by a dedicated Vastriqo database. The final core commerce path will not depend on Shopify, `bms-api`, or the BMS database. CCA/BMS will remain independently deployable.

## Phase Status

### Phase 0 — Current-System Audit
Status: Complete

### Phase 1 — Target Architecture & Requirements
Status: Engineering Complete; Approval Pending

### Phase 2 — Detailed Data & API Design
Status: Not Started

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

## Open Decisions

The consolidated register in `PHASE-1-CONCEPTUAL-ARCHITECTURE.md` §15 is authoritative. The remaining business decisions are limited to:

- catalog taxonomy and conditional operational/product fields;
- collection landing and card-image override meaning;
- inventory location, reservation, deduction, adjustment, and oversell policy;
- guest checkout, verification/phone/address policy, cart merge/expiry, and customer migration;
- payment/COD, shipping, tax/GST, discounts, fraud, consent, and quote-expiry policy;
- order numbering/lifecycle, cancellation/refund/return/exchange, fulfillment/tracking, and notifications;
- historical and auxiliary data migration/retention;
- admin roles, permitted operations, PII access, support/CMS/dashboard scope, and audit retention; and
- permanent CCA/BMS integration, analytics/consent, and wallet/referral disposition.

Separate initialization ADRs must select exact framework/database/version, identifier/money/session, deployment/security, testing/observability, and admin-UI conventions. The architecture already fixes npm, strict TypeScript, modular-monolith, relational, OpenAPI, native-identity, and independence principles.

## Current Blockers

- The separate `vastriqo-api` and `vastriqo-admin` Git repositories have not yet been created; this is intentional and no application initialization has started.
- The conceptual architecture is complete, but the genuine business decisions, production source inventories, and initialization ADRs listed in `PHASE-1-CONCEPTUAL-ARCHITECTURE.md` §§15, 17, and 18 are not yet approved.

## Next Step

- Review and approve `PHASE-1-CONCEPTUAL-ARCHITECTURE.md`, resolve its consolidated business decisions, validate production source inventories, and record the initialization ADRs. Phase 2 can then freeze the physical schema, API/security contracts, lifecycle/concurrency design, migration specifications, and admin workflows before either repository is initialized.

## Change Log

### 2026-08-24

- Created the migration progress report.
- Recorded Phase 0 as complete and Phase 1 as in progress.
- Summarized current and target architecture, confirmed decisions, implementation-readiness gaps, blockers, and the next step.

### 2026-08-25

- Completed the Phase 1 requirements disposition, product/catalog conceptual design, and full commerce conceptual architecture.
- Recorded exact `vastriqo-api` and `vastriqo-admin` initialization prerequisites.
- Marked Phase 1 engineering work complete with business approval and data-validation gates still pending.
