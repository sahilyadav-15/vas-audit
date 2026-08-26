# Vastriqo Phase 1 — Conceptual Commerce Architecture

Date: 2026-08-25; closure reconciliation recorded 2026-08-26 (Asia/Kolkata)
Status: **Conceptual architecture complete; all Phase 1 business decisions resolved; final readiness governed by `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`**
Implementation status: **No application code, repository, database, migration, or external data was created or changed**

## 1. Purpose and Outcome

This document completes the source-audit-to-conceptual-architecture step for the independent Vastriqo commerce platform. It extends:

- `PHASE-1-FUNCTIONAL-DEPENDENCY-AUDIT.md`;
- `PHASE-1-REQUIREMENTS-AND-FEATURE-DISPOSITION.md`; and
- `PRODUCT-CATALOG-DESIGN.md`.

`PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` subsequently reconciles the unresolved items under the rule that demonstrated storefront capabilities are replacement requirements. It is authoritative where the earlier “needs business input” lists or readiness conclusion differ from the final A–E classification.

The specialized product document remains the detailed catalog evidence record. This document resolves the wider platform boundaries and connects product/catalog to collections, inventory, identity, wishlist, cart, checkout, orders, payment/shipping/tax, admin, migration, API, and database responsibilities.

This is not an MVP design. The target is a complete replacement and major upgrade of the commerce capabilities currently delegated to Shopify and the BMS bridge. Incremental delivery is still required to make the replacement safe, but the target model is not intentionally disposable or limited to a demo.

The central result is:

```text
vastriqo storefront -----------+
                               |
vastriqo-admin ----------------+----> vastriqo-api ----> Vastriqo database
                               |             |
migration/reconciliation ------+             +----> approved replaceable providers
                                             payment / shipping / tax / email / media
```

In the final state:

- `vastriqo-api` owns core commerce behavior and is a separate application/repository;
- `vastriqo-admin` is a separate application/repository and uses only approved `vastriqo-api` admin contracts;
- the Vastriqo database is independent and is not the BMS database;
- Shopify and BMS are migration sources or temporary cutover dependencies only;
- no Shopify/BMS identifier is a Vastriqo domain primary key;
- no core runtime read or write requires Shopify, `bms-api`, or the BMS database; and
- CCA production, finance, invoice, seller, staff, and IoT concepts remain outside Vastriqo.

## 2. Evidence and Decision Rules

### 2.1 Evidence inspected

The repository contains three independent Git repositories and no root workspace/package configuration:

| Application | Established stack | Relevant evidence |
| --- | --- | --- |
| `vastriqo` | Next.js 16 App Router, React 19, strict TypeScript, npm, Tailwind 4, styled-components, Ant Design 6 | Current customer journeys, BFF/route handlers, Shopify catalog/cart, BMS proxies, localStorage auth/cart pointers. |
| `bms-api` | Node, Express 5, strict TypeScript, npm, Knex 3, MySQL, route/controller/service conventions | Shopify customer/order/address integration, catalog mirror, collections, wishlist, inquiry/newsletter persistence, migrations, operational conventions and debt. |
| `bms-admin` | React 19, React Router 7 SPA, Vite 6, strict TypeScript, npm, React Query 5, Axios, Ant Design 5, Tailwind 4 | Existing administration interaction patterns, BMS/CCA module boundary, Shopify mirror/collection screens, weak route authorization and configuration coupling. |

The source trace followed active storefront call sites through Next route handlers into BMS routes/controllers/services and relevant Knex migrations. No conclusion in this document assumes that a queried Shopify field, an unused BMS table, or static marketing copy is a business requirement.

This was a local static analysis. Live Shopify, deployed environment configuration, production BMS rows, payment configuration, shipping configuration, and tax configuration were not queried. Items that require those facts are explicitly validation or business gates.

### 2.2 Classification meanings

Each domain below uses these exact requirement classifications:

- **MUST SUPPORT** — required to preserve current customer behavior or operate independent core commerce safely.
- **SHOULD SUPPORT** — part of a durable independent platform, but not required to reproduce a currently visible behavior at first cutover.
- **DEFER** — valid capability that must not block the current architecture or first replacement sequence.
- **RETIRE** — current data, coupling, behavior, or assumed capability that must not be reproduced.
- **NEEDS BUSINESS INPUT** — source and engineering evidence cannot select the business policy.

“Fix” below means the replacement must correct the deficiency; it does not authorize a change to the current applications now.

## 3. Target Boundaries and Ownership

### 3.1 Deployment and repository boundary

`vastriqo`, `vastriqo-api`, `vastriqo-admin`, `bms-api`, and `bms-admin` are peer applications. A target application must not be nested inside any other application. There is no approved shared runtime package or shared database.

The initial backend should be a **modular monolith**, not a set of microservices. Catalog, inventory, customer, cart, checkout, order, payment, fulfillment, support, admin identity, and import are explicit modules inside one deployable API and one transactional database. Module boundaries, application services, and integration adapters must be separable without introducing distributed consistency before it is required.

### 3.2 Data ownership

| Owner | Owns | Must not own |
| --- | --- | --- |
| `vastriqo-api` and Vastriqo DB | Products, variants, attributes, collections, prices, stock, reservations, new native customers, credentials/sessions, addresses, wishlists, carts, checkouts, orders, payment state, fulfillment/shipment state, future native inquiries/newsletter where implemented, admin identities/authorization, audit, and mappings for included migration records. | Shopify/BMS-shaped mirrors, test-era customer/order/auxiliary history, CCA operations, browser presentation state. |
| `vastriqo` storefront | Customer presentation, route composition, browser interaction, accessibility, SEO rendering, analytics after approval, and a narrow BFF where needed for secure sessions. | Authoritative price, inventory, eligibility, order creation, payment truth, or domain identifiers from Shopify. |
| `vastriqo-admin` | Administration presentation, workflows, client-side form state, and calls to authorized admin APIs. | Direct database access, direct provider credentials, commerce rules duplicated in the browser, or BMS/Shopify runtime calls. |
| External providers | Bounded payment, shipping, tax, messaging, and media services that are explicitly selected. | Vastriqo domain ownership or the only copy of accepted commerce outcomes. |
| Migration tooling | Read-only extraction, staging, transformation, import, validation, reconciliation, and source mappings. | Customer-facing runtime authority or writes to Shopify/BMS. |

### 3.3 API surfaces

The API has four logical surfaces even if one framework/deployment hosts them:

1. **Public storefront reads** — published catalog, collection, search, availability projections, and public submission endpoints.
2. **Customer commerce** — customer/profile/address, wishlist, cart, checkout, and authorized order history.
3. **Admin operations** — authenticated and authorized catalog, inventory, customer support, order/payment/fulfillment, audit, and import operations.
4. **Integration/import** — provider callbacks/webhooks and controlled migration/reconciliation commands. These are not public storefront routes.

Authentication, authorization, rate limits, response/error rules, and audit requirements differ by surface. A route path prefix alone is not an authorization model.

## 4. Domain Analysis and Requirement Classification

## 4.1 Product / Catalog

### Current trace and actual storefront use

- `vastriqo/lib/shopify/index.ts` supplies a rich Storefront GraphQL list capped at 100 products and product detail by handle/legacy Shopify ID.
- `/products`, `/collections`, and `/search` search, filter, and sort the fetched subset in the browser.
- Product detail consumes handle, title, descriptions, selected options, variants, selected price, availability flags, ordered images/media, and selected metafield/metaobject values.
- Home, Men, Women, Kids, and related-product cards consume a thinner BMS projection created by Shopify Admin sync.
- Variant ID is the purchase identity; product ID/handle also flow into analytics, wishlist snapshots, related lookup, SEO, and legacy links.
- The exact consumed field inventory and product conceptual model are recorded in `PRODUCT-CATALOG-DESIGN.md` §§5–13.

### Deficiencies and treatment

- The first-100 list, browser-only discovery, and no-op “Best Sellers” option are incomplete behavior: **fix**, not preserve.
- Missing queried `quantityAvailable` makes UI availability permissive; Shopify rejects invalid purchase later: **fix** with server-authoritative sellability.
- Canonical product SEO ignores queried overrides and sitemap uses a missing BMS route: preserve generated SEO behavior, **fix** sitemap, and do not import overrides without approval.
- The BMS product “Edit” control is read-only and the mirror is truncated/stale-prone: **replace**, not promote to native admin/schema.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Vastriqo product and variant identities; unique canonical handle; title; searchable plain content and safe rich content; ordered options/values and valid variant combinations; current variant price and ISO currency; ordered independent media with featured behavior and alt text; required specifications listed in the product design; explicit publication/visibility; server-authoritative sellability; complete paginated list/search/filter/sort; product detail; stable card projection; sitemap projection. |
| **SHOULD SUPPORT** | Handle-alias/redirect records once the redirect policy is approved; deliberate merchandising timestamps; typed reusable attribute vocabularies where values are controlled; source comparison/admin import preview. |
| **DEFER** | Compare-at/promotional pricing, variant-specific media, generic SEO overrides, arbitrary tags, brand/vendor presentation, and extra metafields unless approved by business/data validation. |
| **RETIRE** | Shopify GIDs as primary/runtime identities; Shopify GraphQL shape; BMS `shopify_products` as a product model; raw `meta` JSON; first-100 discovery; browser-authoritative filtering/totals; existence-of-variant availability fallback; Shopify collection/category relationships with no current consumer; generic metafield editor. |
| **NEEDS BUSINESS INPUT** | Catalog/audience/color/size vocabulary; SKU/barcode/weight/tax/shipping fields; brand/vendor and tag semantics; compare-at requirement; single/multi-currency business scope; publication minimums; handle redirect retention; “new arrival” and default ranking rules; attribute governance; production-only custom fields; variant media; media publication rule. |

## 4.2 Collections and Merchandising

### Current trace and actual storefront use

- BMS `shopify_categories` plus ordered `shopify_category_products` drive four configured slugs: home featured products and Men/Women/Kids.
- Related products are selected from a shared active BMS category/collection and ordered through membership order.
- Collection records use title, description, slug, active/inactive state, collection order, membership, and membership sort order.
- `/collections` is not a collection landing; it repeats the all-products grid.
- BMS collection editing can pass an image override, but persistence writes the shared `shopify_products.image_url`, so the apparent placement edit affects every BMS card for the product.

### Deficiencies and treatment

- Active collection reads do not independently enforce product publication, allowing stale cards: **fix**.
- Related-product ordering lacks a fully documented tie-breaker: **fix** with deterministic native ordering.
- The image-override UI and stored meaning disagree: do not preserve until business intent and live usage are validated.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Native Collection identity, title, unique slug, optional description, visibility/active state, deterministic collection order, unique ordered product membership, home featured placement, Men/Women/Kids destinations, and related products based initially on shared visible collection membership. All reads must also enforce product publication/sellability display rules. |
| **SHOULD SUPPORT** | Admin membership bulk assignment/reorder; preview of public results; deterministic fallback/tie-breaking; collection-aware sitemap/navigation projection if the final information architecture includes it. |
| **DEFER** | Automated collections, rule builders, campaign scheduling, personalization, best-seller algorithms, and a generalized placement engine. |
| **RETIRE** | Shopify collection membership as the target; `shopify_categories` naming/IDs/schema; product relationships through external IDs; stale/unpublished product leakage; BMS pages/menus as an assumed collection/content source. |
| **NEEDS BUSINESS INPUT** | Whether `/collections` becomes a true landing; audience taxonomy versus curated collections; whether a card image override is product-wide, membership-specific, or retired; any scheduled collection publication requirement. |

**Resolved decision:** collections and ordered membership are in the conceptual architecture now. They are not left as a later BMS dependency because they are already required by multiple current storefront surfaces.

## 4.3 Inventory

### Current trace and actual storefront use

- Shopify owns inventory and downstream purchase enforcement.
- BMS sync stores variant inventory quantities and product total inventory inside a partial mirror/JSON snapshot; it has no stock location, reservation, adjustment, or inventory ledger domain.
- Storefront cards/details do not receive reliable quantity and can mark a variant usable merely because it has an ID.
- No source evidence establishes single versus multiple locations, backorders, reservation timing, or fulfillment deduction policy.

### Deficiencies and treatment

- UI availability is not authoritative: **fix** at every catalog, cart, checkout, and order boundary.
- A last-synced aggregate cannot be the stock source of truth: **retire as authority**, retain only for comparison.
- The replacement cannot accept an order without concurrency-safe stock behavior: reservation/release/consume is a required capability even though policy details remain open.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Authoritative stock per variant and stock scope; available-to-sell computation; concurrency-safe reservation, release, expiry, and consumption; controlled adjustments with reason/actor; movement history; order/fulfillment linkage; cutover snapshot plus delta reconciliation; server-authoritative sellability. |
| **SHOULD SUPPORT** | Low-stock/exception queries, inventory reconciliation reports, stock-count corrections, and a logical location model that supports one or more operational locations without changing variant identity. |
| **DEFER** | Transfers, purchase orders, supplier receiving, lot/batch/serial tracking, demand forecasting, and warehouse optimization unless an operations requirement is approved. |
| **RETIRE** | BMS `totalInventory`/variant JSON as authority; Shopify availability enums as the target model; browser-side or “variant ID exists” sellability; untracked manual stock edits. |
| **NEEDS BUSINESS INPUT** | Single versus multi-location operation; backorder/oversell policy; reserve point and expiry; release rules after payment failure; stock deduction at order, payment, or fulfillment; who may adjust stock and approved reason catalogue. |

## 4.4 Customer and Authentication

### Current trace and actual storefront use

- Login and signup both start Shopify Customer Account OAuth/PKCE. BMS exchanges the code, reads the Shopify Admin customer, upserts `customers`, stores a Shopify access token, and issues a 24-hour `type: CUSTOMER` JWT.
- Browser state (`front_token`, `front_user`) is in localStorage. `/api/auth/me` still reaches BMS and live Shopify.
- Profile reads/writes use the BMS customer row, while checkout details dual-write Shopify then BMS. Addresses are live Shopify Admin CRUD/default. Orders are live Shopify Admin reads.
- Logout calls a BMS route not found in the audited router, suppresses the error, and clears only browser state.
- An older BMS `users` email/password front path and an alternate cookie-based Shopify path have no active verified role.
- The approved target direction is independent email/password customer authentication; Google login is deferred. Admin identity is separate.

### Deficiencies and treatment

- Split customer truth and live dependency: **replace** with one native profile/address owner.
- JavaScript-readable bearer tokens and ineffective server revocation: **replace** with server-owned, revocable session lifecycle.
- Current “signup” is not native registration, and recovery/verification are absent: **implement as independent identity capabilities**.
- Logging OAuth verifier/token/customer data is unsafe legacy behavior: **retire**.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Email/password registration and login; unique normalized customer email; securely hashed credentials; current-customer read; revocable sessions with expiry/rotation policy; logout; password recovery; verification lifecycle capability; one native customer profile; address create/read/update/delete and one default address; privacy-safe account state; separate admin identity. |
| **SHOULD SUPPORT** | Session/device listing and all-session revocation; account security/audit events; normalized phone and address validation through explicit policy. |
| **DEFER** | Google/social login, CCA/BMS SSO, passwordless login, loyalty/wallet/referral identity, and customer organization accounts. |
| **RETIRE** | Shopify Customer Account as final identity; Shopify customer tokens; BMS `customer_tokens`; Shopify/BMS IDs in native sessions; localStorage as the authoritative session; legacy BMS `front/auth` path; duplicate customer profile truth; login-time referral-code side effect. |
| **ALREADY DECIDED / DEFERRED POLICY** | Existing Shopify/BMS test customers, credentials and addresses do not migrate or link; new accounts start clean. Verification gates, wider country support, guest checkout and native-data retention/deletion details are later implementation policies behind the established identity boundary. |

## 4.5 Wishlist

### Current trace and actual storefront use

- BMS `wishlists` stores one row per owner plus Shopify product ID and handle/title/image snapshots. The active path uses `customer_id`; the older path can use BMS `user_id`.
- Authenticated customers can list, save, toggle, and remove products.
- When a signed-out user clicks wishlist, the storefront stores one pending product in localStorage, starts login, and applies it after callback.
- Snapshot fields can become stale and product links rely on Shopify handle/ID.

### Deficiencies and treatment

- Dual owner types and Shopify identity coupling: **replace**.
- Product snapshots must not be catalog authority: display from current Product; retain only a narrowly justified event/history snapshot if required.
- A missing/unpublished/deleted product needs deterministic handling instead of a broken link.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | One active WishlistItem per Customer/Product; list/add/remove/toggle; native IDs; authenticated authorization; pending-login add behavior; current product card projection; deterministic unavailable/deleted-product handling. |
| **SHOULD SUPPORT** | Idempotent add/remove, pagination for large lists, and privacy-safe customer export/deletion handling for new native data. |
| **DEFER** | Multiple named wishlists, sharing, notes, alerts, collaborative lists, and anonymous persistent wishlists beyond the current one-item login handoff. |
| **RETIRE** | `user_id`/`customer_id` polymorphism; Shopify product identity as relation; stale title/image/handle as display authority; admin visibility without an approved support purpose. |
| **ALREADY DECIDED / DEFERRED POLICY** | Existing BMS wishlist rows do not migrate. Unavailable-product presentation and any merchandising/analytics use can be refined later without changing the clean-start decision. |

## 4.6 Cart

### Current trace and actual storefront use

- `CartProvider` stores a Shopify cart ID in localStorage and calls Shopify cart create/read/add/update/remove functions.
- Cart reads up to 100 lines and shows variant title, unit price/currency, product title/handle/image, quantity, subtotal/total, and item count.
- A cart can be built anonymously. Customer identity is attached only when checkout begins, and checkout currently requires login.
- “Buy Now” creates a separate single-line Shopify cart and does not mutate the visible cart.
- There is no implemented login merge, multi-device cart, explicit expiry policy, or native pricing/inventory revalidation.

### Deficiencies and treatment

- Client-held external cart ID is not ownership/security: **replace** with a native cart access/ownership model.
- Current update/remove helpers do not consistently surface Shopify user errors: **fix** with deterministic API errors.
- Cart money and sellability must be revalidated by the server, not trusted from browser projections.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Native Cart and CartItem identities; anonymous cart continuity; optional customer association; add/read/change quantity/remove; positive quantity rules; current product/variant projection; server-computed subtotal; price/publication/inventory validation; explicit active/expired/converted lifecycle; ownership/access protection; direct-purchase cart/checkout path. |
| **SHOULD SUPPORT** | Mutation idempotency, optimistic concurrency/versioning, customer cart continuity, item-level validation messages, and cleanup/observability for abandoned carts. |
| **DEFER** | Saved carts, multiple carts per customer, cross-device anonymous cart recovery, gift messages, and promotions until approved. |
| **RETIRE** | Shopify cart IDs and checkout URLs; `localStorage.cartId` as authority; Shopify line/merchandise schema; the unused Next `/api/cart` route as a target contract; client-authoritative totals. |
| **NEEDS BUSINESS INPUT** | Guest checkout; anonymous/customer cart expiry; login merge/replace policy; direct “Buy Now” relationship to an existing cart; per-line or cart quantity limits. |

## 4.7 Checkout

### Current trace and actual storefront use

- The storefront requires a BMS customer JWT, validates first name, last name, and normalized 10-digit phone, retrieves the stored Shopify customer token, attaches it to the Shopify cart with country `IN`, and redirects to Shopify hosted checkout.
- Shipping, tax, payment, fraud handling, confirmation, and final order creation are hidden inside Shopify.
- Both cart checkout and Buy Now use the same customer-detail gate but create different Shopify carts.
- The visible Cart/Address/Payment progress labels do not represent native application checkout steps.

### Deficiencies and treatment

- A redirect to Shopify is not an independent checkout: **replace** with a durable native checkout orchestration boundary.
- Updating the customer profile merely to prepare checkout mixes account data and a purchase snapshot: **fix** by allowing checkout contact/address snapshots while deliberate profile changes remain separate.
- Provider callbacks/retries must not duplicate charges, reservations, or orders.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Start checkout from a valid cart or direct-purchase path; collect/confirm contact and shipping address; revalidate lines, prices, publication and inventory; create/refresh shipping and tax quotes; select a shipping option and payment method through provider-neutral boundaries; reserve stock; persist checkout state; accept asynchronous result; create exactly one order; return recoverable validation/failure state; idempotency across client retry and provider callback. |
| **SHOULD SUPPORT** | Quote expiry, resumable checkout, explicit terms/consent evidence, risk review hooks, operational failure recovery, and trace correlation across checkout/payment/order. |
| **DEFER** | Gift wrapping, gift messages, stored payment methods, complex promotions, international checkout, and multi-shipment checkout unless approved. |
| **RETIRE** | Shopify hosted checkout URL as final architecture; Shopify buyer token attachment; country hard-coded in integration code as a permanent model; customer-profile dual write; UI steps that do not reflect real state. |
| **NEEDS BUSINESS INPUT** | Guest versus authenticated checkout; mandatory contact fields; supported countries; shipping services/rates/free-shipping rules; tax/GST behavior; payment methods/provider/COD; discounts; fraud checks; terms/privacy consent; cart and quote expiry. |

## 4.8 Orders

### Current trace and actual storefront use

- BMS queries Shopify Admin for reverse chronological customer orders; storefront requests only the latest 10.
- Current cards use order ID/name/date, financial and fulfillment status, total/subtotal/shipping/tax, first transaction gateway, line title/variant/quantity/price/image, and shipping address.
- The BMS query also fetches billing address, discounts, tracking/fulfillment details, and broader transaction data that the UI ignores.
- “Track Order” only scrolls to the same card. “Cancel Order” redirects to contact; it performs no cancellation, and the destination does not consume the passed subject.
- BMS `orders`/`order_items` are CCA production records, not Vastriqo retail orders.

### Deficiencies and treatment

- Latest-10-only history is incomplete: **fix** with authorized pagination and detail.
- Fake tracking and ineffective cancellation are broken UI claims: **do not preserve**; either implement approved workflows or label/route honestly.
- Order, payment, and fulfillment states must not be collapsed into Shopify strings.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Exactly-once native order creation; customer and order number identity; immutable line title/variant/options/media/quantity/unit-price snapshots; shipping/contact address snapshots; exact subtotal, shipping, tax, discount-if-approved, and total; currency; separate order/payment/fulfillment state dimensions; payment references; inventory reservation/consumption linkage; customer list/detail with authorization and pagination; admin search/detail. |
| **SHOULD SUPPORT** | Fulfillment records sufficient to complete accepted orders; shipment/tracking record capability; order timeline/audit; support notes separated from customer-visible data. |
| **DEFER** | Detailed advanced split fulfillment and customer self-service cancellation/refund/return/exchange/tracking workflow design until the relevant order/post-purchase phase. These are required future capabilities, not retired behavior. |
| **RETIRE** | Shopify order IDs/status enum names as native semantics; BMS CCA order schema; first gateway as the complete payment history; latest-10 cap; fake track/cancel behaviors. |
| **ALREADY DECIDED / DEFERRED POLICY** | Current test orders do not migrate or receive a dedicated archive. Order numbering/lifecycles are engineering design; detailed post-purchase and invoice/GST policies are approved in their relevant implementation phases. |

## 4.9 Payment, Shipping, and Tax

### Current trace and actual storefront use

- Shopify owns all live payment, shipping-rate, tax, checkout, order-confirmation, and much fulfillment behavior.
- Repository source does not establish the configured gateway, enabled payment methods, real COD eligibility, shipping carriers/services, rate table, free-shipping rule, tax registration/rules, invoice format, fraud controls, or notification provider.
- FAQ and shipping/returns pages make operational claims, but code does not implement or verify those claims.

### Required capability boundaries

Provider neutrality means the Vastriqo domain records its own requests, decisions, accepted outcomes, provider references, and reconciliation state. It does not mean building a generic abstraction for every possible provider before one is selected.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Payment attempt/transaction state, exact amount/currency, idempotent initiation, verified callbacks/webhooks, duplicate/out-of-order event handling, failure recovery, provider-reference reconciliation, and no sensitive card storage; shipping address, option/service, quoted charge, quote expiry, order snapshot and fulfillment handoff; compliant tax/GST calculation boundary, retained result/breakdown, and order/invoice evidence. |
| **SHOULD SUPPORT** | Replaceable adapters behind stable application services; manual exception queues; scheduled reconciliation; health/latency/error observability; signed webhook verification and replay protection; test/sandbox mode separation. |
| **DEFER** | Multiple gateways/carriers/tax engines, advanced promotion engine, stored instruments, international duties, split tenders, wallet/coins, and automated return shipping until approved. |
| **RETIRE** | Shopify gateway/status names as the domain; static marketing copy as system truth; payment/shipping/tax logic in storefront components; provider secrets in admin-editable generic settings; raw provider payload as the only record of outcome. |
| **NEEDS BUSINESS INPUT** | Provider(s), payment methods and COD; authorization/capture/refund behavior; shipping zones/services/rates/free shipping/carrier versus manual fulfillment; tax/GST responsibility, registration, place-of-supply and invoice requirements; discounts; fraud controls; notification events/channels; reconciliation ownership/frequency. |

## 4.10 Admin Requirements

### Current trace and actual use

- `bms-admin` is a separate React SPA with useful list/form/query-hook patterns but is a combined CCA/BMS application.
- Route protection checks only for token presence; all navigation is effectively available after login. JWT and user data are stored in localStorage; the BMS API URL is hard-coded.
- Shopify product administration is sync/filter/read-only inspection. Collection CRUD, assignment, removal, and reorder are real current merchandising operations.
- Current CCA products, orders, invoices, transactions, users, seller details, and IoT records are unrelated to Vastriqo retail commerce.
- Inquiries have an admin status workflow. No native Shopify retail-order admin, customer admin, inventory operation, or payment/fulfillment operation exists.

### Deficiencies and treatment

- Native admin writes cannot launch before independent admin authentication/authorization and audit are in place.
- BMS UI patterns can inform interaction design, but hard-coded API configuration, localStorage auth, role-free routing, inconsistent types, and CCA domain modules must not be copied.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Separate `vastriqo-admin`; separate admin accounts/sessions from customers; multiple admins; authorization checks in API; product/variant/option/attribute/media/publication management; collection membership/order; inventory view/adjustment/history; customer lookup with controlled PII; order search/detail and approved operations; payment/fulfillment exception visibility; inquiry handling if retained; import preview/run/reconciliation access; attributable audit events; pagination/filtering/validation/conflict feedback. |
| **SHOULD SUPPORT** | Role/permission model designed for later expansion, bulk catalog operations, operational queues, saved filters, accessible responsive forms, safe impersonation-free support tooling, and explicit environment/build metadata. |
| **DEFER** | Dashboard until metrics/actions are defined; CMS/navigation management; wishlist analytics; newsletter campaign tooling; generic settings; wallet/referrals; CCA/BMS SSO. |
| **RETIRE** | CCA users/products/orders/invoices/transactions/seller/IoT modules; Shopify sync as a permanent operation; Shopify Admin links as required workflow; BMS storefront pages/menus as assumed CMS; token-presence-only protection; hard-coded API base URL; direct database/provider access from admin. |
| **NEEDS BUSINESS INPUT** | Initial role/permission catalogue; which support users can view/edit customer/address data; approved order/payment/refund/fulfillment operations; inquiry/newsletter scope; dashboard metrics; content/CMS scope; bulk workflows and approval requirements; audit retention. |

## 4.11 Migration Mapping

### Current source roles

| Source | Appropriate role | Not appropriate as |
| --- | --- | --- |
| Shopify | Preferred complete source for product, variant, option, media, price, publication input and inventory cutover snapshot. | Runtime owner after cutover; target schema; source for customer/address/test-order migration; source through capped storefront queries. |
| BMS | Source for active custom collection definition/membership/order and secondary reconciliation evidence for included product/merchandising data. | Source for customer, wishlist, inquiry, newsletter or test-order migration; Product/inventory/customer/order authority; Vastriqo database; source for CCA retail history. |
| `vastriqo` source | Source for existing editorial content, routes, display behavior, analytics events, and navigation. | Commerce database or proof that marketing claims are implemented. |
| Vastriqo target | New native identities, normalized domain records, accepted business truth, import provenance and reconciliation results. | A raw archive of every Shopify/BMS field. |

### Deficiencies and treatment

- Current Shopify/BMS queries have per-page/nested limits and cannot establish complete migration counts.
- BMS sync is one-way and has no proven delete/unpublish reconciliation.
- Re-running an import by title, handle, email alone, or BMS primary key can create duplicates or overwrite native edits.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | Read-only extraction; complete pagination/export; immutable ImportRun identity and source snapshot/watermark; deterministic normalization; external source mappings with unique source key; idempotent upsert; relationship resolution only through mappings; dry run/preview; validation and issue reporting; counts/checksums/sample comparison; safe rerun; cutover delta; no writes to Shopify/BMS; explicit no-migrate classifications. |
| **SHOULD SUPPORT** | Per-record source hash/status; resume after failure; import-owned-field/conflict policy; reversible target batch before public cutover; media copy verification; historical source archive with bounded retention; admin-readable reconciliation. |
| **DEFER** | Active cart migration, continuous post-cutover sync and a generic ETL platform. |
| **RETIRE** | Tokens/secrets/sessions; raw BMS product JSON as domain data; Shopify/BMS IDs as PK/FK; current test-era customers, addresses, wishlists, orders, inquiries and newsletter rows; CCA products/orders/users/invoices/finance/seller/IoT; disconnected BMS pages/menus; wallet/referral login side effects; stale/orphan relationships; unapproved personal data. |
| **ALREADY DECIDED / VALIDATE LATER** | Product/catalog, media, publication, inventory and required collection data are the migration priority. Product field/taxonomy mappings, image overrides, inventory cutover timing and tombstone behavior are engineering/data-validation checkpoints, not open historical-data scope. |

The detailed deterministic migration design is in §10.

## 4.12 API and Database Design

### Current conventions and actual constraints

- BMS demonstrates that the team can operate Node/TypeScript, a relational database, explicit migrations, route/controller/service layering, transactions, validation, pagination helpers, Docker, and npm.
- All current applications use strict TypeScript and npm lockfiles. Current deployments use Node containers.
- BMS also demonstrates what not to reproduce: global database access, transaction commit inside response helpers, inconsistent envelopes/status behavior, inline queries in controllers, conditional compatibility migrations, hard-coded/default credentials, permissive CORS, JavaScript-readable bearer tokens, high body limits, sensitive logging, and no automated test runner.
- `vastriqo` already uses server components/route handlers as a potential BFF boundary, but that does not make Next.js route handlers the commerce source of truth.

| Classification | Requirements |
| --- | --- |
| **MUST SUPPORT** | One independently deployed modular API; one dedicated transactional relational database; strict TypeScript; explicit versioned migrations; domain/application/integration separation; transactions owned by use cases; consistent validation/error contract; public/customer/admin/integration authorization boundaries; exact money handling; timestamps in UTC; opaque native IDs; idempotency; optimistic/concurrency controls where required; API contract documentation; automated tests; environment-only secrets/config; audit and observability. |
| **SHOULD SUPPORT** | OpenAPI as the HTTP contract, compatibility/version policy, structured logging with correlation IDs, health/readiness checks, background job/outbox capability for reliable async effects, rate limiting, privacy operations, backups/restore tests, and generated/shared API client types without sharing server runtime code. |
| **DEFER** | Microservices, event streaming platform, GraphQL, separate search engine, cache/Redis, data warehouse, and a generalized workflow engine until measured need exists. |
| **RETIRE** | Shared BMS database/code/auth/secrets; copying BMS tables; Shopify-shaped public payloads; mixed response shapes; route paths as authorization; hard-coded URLs; permissive CORS; secrets/default credentials in source; logging tokens/SQL bindings/PII; tests as optional manual work. |
| **NEEDS BUSINESS INPUT** | None at the conceptual infrastructure boundary. Business policies affect domain contracts, not whether the API/database must be independent, transactional, secure, testable, and observable. |

Framework, database product/version, and exact package versions are engineering selections that must be recorded before repository initialization; §11 states what evidence is sufficient to decide now and what still requires an ADR.

## 5. Conceptual Domain Model

```text
Product 1 --- * ProductVariant * --- * ProductOptionValue * --- 1 ProductOption
   |                  |
   |                  +--- * VariantPrice
   |                  +--- * InventoryPosition --- 1 InventoryLocation
   |                  +--- * InventoryReservation --- 1 Checkout/Order
   |
   +--- * ProductMedia --- 1 MediaAsset
   +--- * ProductAttributeValue --- 1 AttributeDefinition
   +--- * CollectionProduct * --- 1 Collection
   +--- * WishlistItem * --- 1 Customer
   +--- * CartItem * --- 1 Cart

Customer 1 --- * Address
   |       +--- * CustomerCredential / CustomerSession / AccountToken
   |       +--- * WishlistItem
   |       +--- * Cart
   `-------+--- * Order

Cart 1 --- * CartItem
  |
  `--- 0..* Checkout 1 --- * PaymentAttempt --- * PaymentTransaction
                    |
                    `--- 0..1 Order 1 --- * OrderItem
                                      +--- * OrderAddress
                                      +--- * Fulfillment --- * FulfillmentItem
                                      `--- * Shipment --- * TrackingEvent

AdminUser * --- * Role * --- * Permission
AdminUser 1 --- * AuditEvent

ImportRun 1 --- * ImportRecord / ImportIssue
ExternalSourceMapping * --- 1 approved target entity
```

Relationships marked with provider- or policy-dependent entities express required capability, not a finalized physical table layout.

## 6. Entity and Aggregate Catalogue

### 6.1 Catalog and merchandising

| Concept | Required fields/concepts | Ownership and invariants |
| --- | --- | --- |
| Product | Native ID, canonical handle, title, plain/searchable description, safe rich description, classification/audience references when approved, merchandising/publication timestamps, publication state, audit timestamps. | Aggregate root. Handle unique among canonical products. Public queries never expose non-public products. Shopify timestamps do not silently become native business timestamps. |
| ProductHandleAlias | Previous handle, Product, redirect status/time. | Conditional on redirect policy. Alias cannot collide with a canonical handle or point to multiple products. |
| ProductVariant | Native ID, Product, optional approved SKU, title/derived label, publication state, ordered option selections, sellability inputs. | Belongs to exactly one Product. Option combination is unique within Product. A variant cannot select an option/value outside its Product. |
| ProductOption / ProductOptionValue | Product, name/code, display order; value label/code/display order. | Selectable dimensions only. Options/values cannot be removed while used without a deliberate variant update. |
| VariantPrice | Variant, exact amount, ISO currency, effective/active concept. | No floating-point money. At most one current price per Variant/currency. Public minimum price derives from eligible variants. Multi-currency cardinality follows business decision. |
| MediaAsset / ProductMedia | Independent object/URL reference, media type, integrity/status, alt text, role, Product, order, optional Variant relation if approved. | Media must be durable outside Shopify/CCA. Ordering is deterministic; featured behavior is unique or derived by explicit rule. |
| AttributeDefinition / ProductAttributeValue | Business code, label, data kind, cardinality/vocabulary rule, order; Product value(s). | Shopify namespace/key is mapping metadata only. Required specification meanings are preserved. Invalid type/cardinality cannot publish. |
| Collection | Native ID, unique slug, title, description, visible/active state, order, timestamps. | Native merchandising aggregate. A public Collection cannot expose an ineligible Product. |
| CollectionProduct | Collection, Product, membership order, optional approved placement metadata. | Unique Collection/Product pair; deterministic order with stable tie-breaker. No external product ID relationship. |

### 6.2 Inventory

| Concept | Required fields/concepts | Ownership and invariants |
| --- | --- | --- |
| InventoryLocation | Native ID, name/code, operational/active state. | One logical location is sufficient if single-location is approved; the model does not require multiple physical warehouses. |
| InventoryPosition | Variant, Location, on-hand/committed or equivalent exact quantities, version. | Unique Variant/Location. Available-to-sell is computed through the approved policy and cannot be mutated by the storefront. Concurrency control is mandatory. |
| InventoryMovement | Variant, Location, signed quantity, reason, source type/ID, actor, occurred time, idempotency/import reference. | Append-only audit evidence for stock-changing operations. Balance changes and movement record commit atomically. |
| InventoryReservation | Variant, Location, Checkout/Order reference, quantity, state, expiry, timestamps. | Held quantity cannot be negative; state transitions are monotonic and idempotent; release/consume occurs at most once; expired holds cannot remain available as active. |

### 6.3 Customer and identity

| Concept | Required fields/concepts | Ownership and invariants |
| --- | --- | --- |
| Customer | Native ID, normalized email, display/name fields, approved phone fields, account state, verification markers, audit timestamps. | Unique normalized email. Profile is the single native truth. PII reads/writes are authorized and audited as appropriate. Starts with newly registered native customers; test users are not imported. |
| CustomerCredential | Customer, password hash metadata, created/changed time, disabled/compromised marker. | Never store/recover plaintext password. Imported Shopify/BMS tokens are not credentials. Credential changes revoke sessions according to security policy. |
| CustomerSession | Customer, opaque session reference/hash, expiry, revocation, rotation/family context, last-use metadata. | Server-revocable and not exposed through logs. Browser credential should be inaccessible to JavaScript where architecture permits. |
| AccountToken | Customer, purpose (verification/recovery/activation), hashed token, expiry, consumption time. | Single-use, expiring, purpose-bound; raw token is not persisted. Exact channel/policy comes later. |
| Address | Customer, recipient, company optional, address lines, city, region/province, postal code, country code, phone, default marker. | Belongs to one Customer. At most one default. Order uses a snapshot, not a mutable Address FK for history. |
| WishlistItem | Customer, Product, created time. | Unique Customer/Product active relationship. Product is authoritative for current presentation. |

### 6.4 Cart, checkout, and order

| Concept | Required fields/concepts | Ownership and invariants |
| --- | --- | --- |
| Cart | Native ID, anonymous access reference or Customer owner, state, currency context, version, created/updated/expiry time. | Only an authorized owner/access token can mutate. Converted/expired cart is not mutable. Anonymous access secret is separate from public ID. |
| CartItem | Cart, Variant, quantity, created/updated time, optional last-observed price for change messaging. | Positive quantity. Variant must be valid for Product. Current authoritative price/stock is re-read; observed price is not order authority. Whether same-variant lines merge is a contract decision in Phase 2. |
| Checkout | Native ID, Cart, Customer optional per guest policy, contact, address snapshot, currency, line/price validation version, shipping/tax quote references, reservation state, lifecycle, expiry, idempotency context. | A Checkout cannot produce more than one accepted Order. Stale quotes/stock require revalidation. Completion is atomic/idempotent across order, stock and payment outcome rules. |
| ShippingQuote/Selection | Checkout, service code/label, exact charge/currency, estimated delivery text/date if provided, provider/source reference, expiry. | Selected option must belong to a current quote and is snapshotted on Order. Provider payload is not the domain contract. |
| TaxQuote/TaxLine | Checkout/order line or total scope, jurisdiction/type/rate where required, exact amount/currency, source reference, expiry/calculated time. | Tax totals reconcile to retained lines/rounding policy. Legal invoice fields follow approved policy. |
| Order | Native ID, public order number, Customer/guest identity snapshot, lifecycle dimensions, currency, exact subtotal/shipping/tax/discount/total, accepted time, creation provenance. | Exactly one per completed native Checkout/idempotency key. Money reconciles. Accepted snapshot is not changed when Product/Address changes. Current test orders are not imported. |
| OrderItem | Order, Product/Variant source refs where still present, title, option/variant label, optional SKU, media reference/snapshot, quantity, exact unit/subtotal/discount/tax/total. | Immutable commercial snapshot apart from explicit correction records; sums reconcile to Order. Product deletion cannot erase history. |
| OrderAddress | Order, kind, all approved postal/contact fields. | Immutable shipping/billing snapshot; no dependency on mutable customer Address. |
| PaymentAttempt | Checkout/Order, provider adapter, method, exact requested amount/currency, state, idempotency key, provider reference, timestamps/failure code. | Duplicate initiation/callback cannot duplicate charge/order. Payment state is distinct from order/fulfillment state. |
| PaymentTransaction | Attempt, kind (authorization/capture/refund/etc. as approved), exact amount/currency, provider reference, status, processed time, reconciliation state. | Append-only provider outcome/history; verified events only; no sensitive instrument data. |
| Fulfillment / FulfillmentItem | Order, operational state, timestamps; OrderItem and quantity. | Fulfilled quantities cannot exceed fulfillable quantities. Partial behavior remains decision-gated. |
| Shipment / TrackingEvent | Fulfillment, carrier/service, tracking reference/URL, shipment state, event/time/location when available. | Customer visibility is policy-controlled. Provider data is normalized; raw evidence may be retained separately with bounded access. |

### 6.5 Administration, support, integration, and migration

| Concept | Required fields/concepts | Ownership and invariants |
| --- | --- | --- |
| AdminUser / AdminCredential / AdminSession | Separate admin identity, normalized email, state, secure credential/session lifecycle. | Never a Customer or BMS user identity. All admin API actions authenticate independently. |
| Role / Permission / AdminRole | Named role, stable permissions, assignments. | API enforces permission; UI hiding is supplementary. Initial role catalogue needs approval, but data design must not assume one all-powerful shared login. |
| AuditEvent | Actor, action, target type/ID, time, request/correlation ID, before/after summary or safe change set, result. | Append-only, sensitive values redacted, attributable to human/system/import/provider. |
| Inquiry | Native ID, Customer optional, submitted contact snapshot, requirement/message, status, timestamps. | Public submitter identity is not forced into a BMS user FK. Attachment is conditional. |
| NewsletterSubscription | Normalized email, state, consent/source evidence, subscribed/unsubscribed timestamps. | Unique normalized email; unsubscribe/suppression is explicit. Current BMS subscriber rows do not migrate; future records are native if the capability is implemented. |
| ExternalSourceMapping | Source system, source entity type, source ID, target type/ID, source version/hash, first/last import run. | Unique source tuple. Never a domain PK/FK exposed as native identity. Retention is bounded by reconciliation/archive need. |
| ImportRun | Source/snapshot/watermark, configuration version, mode, state, started/completed time, counts/checksums. | Immutable run identity and input manifest. A rerun with the same input/config yields the same intended target result. |
| ImportRecord / ImportIssue | Run, source key/hash, target mapping, action/result, warning/error details. | Supports resume, diagnosis, reconciliation and proof that failures were not silently skipped. No secrets/raw sensitive values in user-facing reports. |
| ProviderEvent | Provider, external event ID, type, received/verified/processed time, safe payload reference/hash, result. | Unique provider/event ID; verify before processing; safe replay and out-of-order handling. |

## 7. Lifecycle and Cross-Domain Invariants

Exact enum names belong to detailed design. The following distinct lifecycle concepts are already required:

| Domain | Required lifecycle dimensions | Required invariants |
| --- | --- | --- |
| Product | Draft/non-public versus public/unpublished/retired concepts. | Public reads enforce state plus required data and variant eligibility. Collection membership never bypasses it. |
| Variant | Operational/public eligibility plus inventory-derived sellability. | Product public state alone cannot make an invalid/no-price variant purchasable. |
| Inventory reservation | Held, released, consumed, expired concepts. | One terminal outcome; retries do not double-release/consume. |
| Customer/session | Account active/restricted/deleted as approved; session active/expired/revoked. | Deactivated account cannot create new authenticated commerce actions. Logout/revocation is server-enforced. |
| Cart | Active, expired, converted/closed. | Only active carts mutate; conversion links to at most one accepted order path. |
| Checkout | In progress, awaiting external outcome, failed/recoverable, completed/expired. | At most one Order; expired quote/reservation cannot be accepted without refresh. |
| Order | Commercial acceptance/cancellation dimension. | Order state is not inferred solely from payment or fulfillment. |
| Payment | Initiated/pending/success/failure and approved refund/capture concepts. | Exact amounts and provider history reconcile; callbacks are idempotent. |
| Fulfillment/shipment | Unfulfilled/in-progress/fulfilled and shipment/delivery concepts. | Fulfilled quantities reconcile to OrderItems; tracking does not imply payment state. |
| Import | Planned/dry-run/running/completed-with-or-without-issues/failed. | Partial failure is reported; a successful status requires all required validations. |

Platform-wide invariants:

1. Vastriqo native IDs are used for all runtime relations and APIs.
2. Money is exact, always carries currency, and never uses binary floating point.
3. Stored timestamps are UTC; business display uses explicit locale/time zone.
4. Order/customer-support history survives later product/media/address changes through deliberate snapshots.
5. Public data is deny-by-default with respect to publication and authorization.
6. Admin UI cannot bypass API authorization or domain validation.
7. Price, stock, tax, shipping, and payment outcomes are server-validated at checkout/order boundaries.
8. Cross-domain write use cases define their transaction and idempotency boundary; HTTP response helpers do not commit database transactions.
9. External calls are not placed inside a database transaction as if they were atomic. Persisted attempts/events plus idempotency/reconciliation bridge the boundary.
10. Sensitive credentials, payment instruments, access tokens, recovery tokens, and secrets are not logged or copied into audit/import output.

## 8. Query, Write, and Responsibility Requirements

### 8.1 Public storefront reads

The public/customer API must provide projections for:

- complete published product list with cursor/page contract, deterministic order, approved search fields, filters/facets, min price, media, options summary, and sellability;
- product detail by canonical handle, including redirect/alias handling if approved;
- visible collection list/detail and ordered products; home/gender placements and related products;
- current cart with item validation messages and exact subtotal;
- checkout state/required next action without exposing provider secrets;
- authenticated profile/address/wishlist/order list/detail; and
- published-product sitemap input.

Storefront responsibilities are presentation, input collection, accessible interaction, and secure session handoff. It must not calculate authoritative availability, shipping, tax, payment success, or order acceptance.

### 8.2 Customer/public writes

Required command responsibilities are:

- register/login/logout/recover/verify according to approved identity policy;
- update permitted profile fields and address CRUD/default;
- wishlist add/remove;
- cart create/add/change/remove and direct-purchase initialization;
- checkout start/update/quote/select/pay/confirm through an explicit orchestration contract;
- public/optional-auth inquiry and newsletter subscription where retained.

Every write validates authorization, input, current aggregate state, and idempotency/concurrency where retries can cause duplicate business effects.

### 8.3 Admin reads/writes

Admin API responsibilities include:

- catalog draft/list/detail, validation, create/update, options/variants/prices/attributes/media/publication;
- collections CRUD, membership assignment/removal/reorder and public preview;
- inventory position, movement/reservation inspection and authorized adjustment;
- customer search/detail and only approved support actions;
- order search/detail/timeline and approved lifecycle/payment/fulfillment operations;
- provider exception/reconciliation queues;
- inquiry status/support operations and approved newsletter operations;
- admin identity/role management according to permission; and
- import preview/run/status/issues/reconciliation with elevated permission.

The admin may display external mapping/provenance for diagnosis but never edits Shopify IDs as native identity.

### 8.4 Migration/import responsibilities

Migration is allowed to write target records only through dedicated import application services or an isolated importer using the same invariants. It must:

- mark origin and run;
- preserve deterministic relationships and order;
- avoid customer-facing publication until validation gates pass;
- protect native edits from silent overwrite;
- expose discrepancies instead of guessing; and
- never call source mutation APIs.

## 9. API and Persistence Architecture

### 9.1 Module boundaries

Recommended modules inside the modular monolith:

```text
identity/customer        admin-identity/authorization
catalog                  merchandising/collections
inventory                wishlist
cart                     checkout
orders                   payments
fulfillment/shipping     tax
support/engagement       media
imports/reconciliation   audit/observability
```

This is ownership, not necessarily one folder/table per name. Checkout coordinates catalog, inventory, shipping, tax, payment, and order application services; it does not own their permanent data.

### 9.2 Layering

Useful BMS knowledge to retain:

- explicit routes/controllers or transport adapters;
- request validation;
- application/service layer;
- relational query/migration layer;
- transactions for related writes;
- shared pagination/error utilities; and
- containerized reproducible builds.

Required improvements:

- controllers/transport do not contain direct SQL/domain rules;
- application use cases own transaction boundaries;
- repositories/query services are module-scoped, not a global mutable connection pattern;
- provider adapters are behind ports owned by the relevant application module;
- one documented success/error/pagination contract is enforced;
- secure configuration has no production defaults;
- logs are structured and redact credentials/PII;
- CORS is explicit by environment; browser session design includes CSRF protection where cookies are used;
- automated unit, integration, migration, contract, and high-risk workflow tests exist from repository initialization; and
- background work uses a durable job/outbox approach where loss would affect order/payment/inventory correctness.

### 9.3 Relational database rationale

A transactional relational database is the established and appropriate class of storage because products/variants, ordered membership, unique identities, stock/reservations, customer ownership, exactly-once checkout/order creation, money reconciliation, and audit require constraints and multi-record transactions. This decision does not copy the BMS schema.

The exact database engine and version are not inferable from business source. MySQL is operationally familiar through BMS; another relational engine may be selected only through a short ADR covering deployment support, migration/tooling, transaction/isolation behavior, search needs, backup/restore, and team competence. The conceptual model does not depend on MySQL enum/JSON behavior.

### 9.4 Contract principles

- JSON over HTTP is the initial API style; publish an OpenAPI contract.
- Stable resource reads and explicit business commands are both allowed; do not force payment/refund/publish actions into generic CRUD.
- Public catalog pagination must be stable and complete. Admin list pagination must include deterministic sort and filter metadata.
- Errors have a stable machine code, human-safe message, field details where applicable, correlation ID, and appropriate HTTP status.
- Idempotency keys are mandatory for checkout/order/payment and any retryable external-effect command.
- API responses use Vastriqo IDs. External IDs appear only in authorized import/reconciliation views.
- Compatibility/version policy is defined before storefront cutover; Shopify-shaped compatibility payloads are not the permanent contract.

## 10. Deterministic Migration and Reconciliation Design

### 10.1 Source-to-target mapping

| Data class | Preferred source | BMS role | Target | Temporary mapping | Records not migrated |
| --- | --- | --- | --- | --- | --- |
| Products/variants/options/prices | Complete paginated Shopify Admin export/query | Compare title/handle/price/meta and identify curation joins | Product, Variant, Option/Value, VariantPrice | Shopify product/variant IDs -> native IDs | Raw GraphQL shape, unused fields, BMS mirror PK/meta JSON. |
| Media | Shopify product media plus approved local source assets | Compare retained image override | MediaAsset/ProductMedia | Shopify media/source URL/hash -> target asset | Unreferenced/unapproved source files and raw metaobject media IDs. |
| Attributes | Approved Shopify fields/metafields/metaobjects | Secondary parsed comparison | AttributeDefinition/ProductAttributeValue | Source namespace/key/reference -> approved business code/value | Unknown/unapproved metafields and opaque JSON. |
| Collections | BMS custom categories | Primary source | Collection | BMS category ID -> native Collection | Shopify collections unless separately approved; inactive/stale records according to policy. |
| Membership/order | BMS custom category products | Primary source after product mapping | CollectionProduct | BMS membership/source product ID -> native relationships | Orphan memberships and unapproved image overrides. |
| Inventory | Authoritative Shopify/operations snapshot at cutover plus delta | BMS only discrepancy evidence | Location/Position/Movement initialization | Shopify variant ID -> native Variant; source snapshot reference | BMS aggregate as authority; historical guessed movements. |
| Customers/addresses | Excluded | None | New native Customer/Address records only | None | All current Shopify/BMS test customers, profiles, credentials, passwords, account links and addresses. |
| Wishlists | Excluded | None | New native WishlistItem records only | None | All current BMS wishlist relationships and snapshots. |
| Historical orders | Excluded | None | New native Order records only | None | All current Shopify/BMS test orders, transactions and fulfillments; no dedicated archive. |
| Active carts | Shopify cart service | No durable source | Normally none; expire/bridge during cutover | Short-lived bridge mapping only if approved | Cart/customer tokens and expired carts. |
| Inquiries | Excluded | None | New native Inquiry records only if/when the capability is implemented | None | All current BMS support/inquiry rows and attachments. |
| Newsletter | Excluded | None | New native NewsletterSubscription records only if/when the capability is implemented | None | All current BMS subscriber/consent rows. |
| Existing legal/financial information | Existing source only when safely reusable for an included requirement | Comparison evidence only | Relevant future tax/order evidence boundary | Bounded mapping only if a concrete included record is carried forward | No broad historical legal/financial dataset or invented migration requirement. |

### 10.2 Deterministic run protocol

1. **Declare input manifest.** Record source, extraction method, schema/config version, full snapshot timestamp or incremental watermark, and expected page/range boundaries.
2. **Extract read-only.** Use complete pagination/export. Never use the storefront first-100 list or nested truncated BMS mirror as completeness proof.
3. **Stage outside the core model.** Preserve necessary source evidence in a protected, time-bounded import workspace. Do not expose raw source documents as Product/Customer/Order data.
4. **Normalize deterministically.** Apply a versioned mapping configuration. Normalize identifiers, handles, option values, money and controlled vocabularies consistently for included records. Stable tie-breakers resolve equal source order.
5. **Validate before write.** Report missing parents, duplicate source keys, handle/SKU-if-approved collisions, invalid option combinations, money/currency errors, missing media, orphan memberships and unsupported values for included records; assert excluded record classes remain unwritten.
6. **Upsert by external mapping.** The unique key is `(source system, source entity type, source ID)`, never mutable title/handle/email alone. Create or locate the native record, then persist/update the mapping in the same target transaction.
7. **Resolve relationships after identities.** Product mappings precede collection membership and included inventory/media relationships. No customer/address/wishlist/historical-order relationship pass is created for the excluded test-era data.
8. **Protect target ownership.** An import updates only approved import-owned fields. If a native admin changed a field after the source version imported, raise a conflict instead of silently overwriting.
9. **Reconcile.** Compare counts, source keys, hashes and domain totals; sample/render representative records; report unchanged/created/updated/skipped/warned/failed/orphaned records.
10. **Prove rerun safety.** Running the same manifest/config twice must create no duplicates and produce no business-data changes on the second successful run.
11. **Apply delta at cutover.** Freeze or watermark source changes, run the final delta, reconcile inventory and orders, and record exit criteria. Shopify/BMS remain unmodified.
12. **Handle disappearance safely.** Missing/unpublished/deleted source records become reconciliation/tombstone findings. Import does not delete native records unless an explicit approved policy and review gate authorizes it.

### 10.3 Required reconciliation outputs

- source/target counts by included entity/publication mode plus zero-target assertions for excluded classes;
- every source key mapped once or intentionally excluded with reason;
- product/variant/option/media/attribute/price/currency differences;
- collection definition, membership and ordering differences;
- inventory quantity, reservation, and available-to-sell cutover totals;
- proof that no current customer, address, wishlist, historical order, inquiry or newsletter source record created a target record or external mapping;
- media copy existence, checksum/content-length where available, and target delivery check;
- import failures/warnings and their disposition;
- proof that Shopify/BMS received no mutations; and
- proof of idempotent rerun.

## 11. Repository and Framework Convention Decisions

### 11.1 Resolved from engineering evidence

1. **Separate peer repositories:** the fixed names are `vastriqo-api` and `vastriqo-admin`; neither is nested inside `vastriqo`, `bms-api`, `bms-admin`, or the other target app.
2. **Modular monolith first:** the current scale and transactional domain favor one API/database with strong internal modules. No repository evidence justifies microservices.
3. **Node and strict TypeScript:** this is common to all three current applications and minimizes ecosystem/context switching. The exact supported Node LTS is pinned at initialization.
4. **npm:** all three current repositories have `package-lock.json`; use npm unless an explicit workspace-wide tooling decision changes it before initialization.
5. **Relational transactional persistence:** required by domain invariants. Use explicit versioned migrations and constraints.
6. **HTTP/JSON plus OpenAPI:** compatible with the current storefront/admin architecture while correcting undocumented/inconsistent BMS contracts.
7. **Independent browser session security:** no JavaScript-readable long-lived bearer token as the intended final design. Prefer secure, HttpOnly, appropriately scoped cookies and a BFF/CSRF design where needed.
8. **Admin React/TypeScript compatibility:** React Query-style server state, centralized API client, strict types, and a consistent component system are useful patterns. The current BMS admin implementation is a visual/workflow reference, not a codebase to fork.
9. **Tests are foundational:** a new repository must not repeat the current no-test-runner condition.

### 11.2 Requires an initialization ADR, not business invention

Before scaffolding, engineering must record:

- backend framework and exact versions;
- relational database engine/version and query/migration library;
- native identifier type/generation;
- exact money persistence and rounding convention;
- session/BFF/CORS/CSRF topology for storefront and admin domains;
- background job/outbox implementation;
- object storage/media delivery choice;
- deployment/runtime, environments, secrets, CI/CD, backup/restore and observability baseline; and
- admin framework/build/component-library versions.

Source evidence supports the decision criteria but cannot prove the available infrastructure, team preference, hosting constraint, or provider contract. Choosing those silently would be invention.

## 12. Data That Must Not Be Copied

The target must not copy or depend on:

- Shopify GraphQL wrappers, GIDs as native IDs, enum names, metafield namespaces, metaobject IDs, checkout URLs, cart IDs, customer tokens, access/refresh tokens, webhook secrets, or raw provider credentials;
- BMS primary keys as native identities, raw `shopify_products.meta`, Shopify admin token records, customer token hashes/access tokens, local JWT/session records, permissive/legacy role values, or compatibility-only columns;
- BMS CCA `users`, products, production orders/items, categories, transactions, invoices/items, seller details, fabrics, IoT, or finance data;
- BMS storefront pages/menus merely because tables/screens exist;
- wallet/referral records or automatic referral-code behavior without approval;
- stale/orphan collection memberships, inferred publication, guessed inventory history, duplicate customer identities, or unapproved deleted records;
- static FAQ/policy claims as payment/shipping/returns requirements;
- full provider payloads as the only payment/shipping/tax truth or sensitive payment instrument data;
- source secrets, logs, analytics identifiers, consent assumptions, or personal data with no approved retention purpose; and
- unused queried Shopify fields solely because Shopify supplies them.

## 13. Quality, Security, and Operational Acceptance

Phase 2 detailed design and later implementation must include:

- unit tests for domain invariants and lifecycle transitions;
- database/integration tests for constraints, transactions, idempotency, reservation concurrency and migration reruns;
- API contract tests for public/customer/admin boundaries and consistent errors/pagination;
- end-to-end tests for registration/login/recovery, product discovery/detail, wishlist, cart, checkout success/failure/retry, order history, admin catalog/stock/order operations, and provider callbacks;
- security tests/review for sessions, authorization, CSRF/CORS, password/recovery, rate limiting, PII exposure, webhook verification and secret handling;
- load/concurrency tests around catalog queries, cart mutations, stock reservation and payment callbacks;
- structured logs, correlation IDs, metrics and alerting for checkout/payment/order/inventory/import failures;
- health/readiness checks, backup/restore proof, migration rollback/recovery procedures and deployment rollback;
- privacy/retention/delete/export behavior for customer, wishlist, inquiry, newsletter, logs and import staging; and
- cutover acceptance comparing the current storefront behavior against representative native data, while intentionally fixing the documented deficiencies.

## 14. Final Resolved Decisions

1. The final core commerce path is `vastriqo`/`vastriqo-admin` -> `vastriqo-api` -> dedicated Vastriqo relational database; Shopify/BMS are not final runtime dependencies.
2. `vastriqo-api` and `vastriqo-admin` are independent peer applications/repositories with their fixed names. Nothing is created yet.
3. The backend starts as a strict-TypeScript modular monolith with explicit modules, one transactional relational database, HTTP/JSON OpenAPI contracts, and versioned migrations.
4. Vastriqo-native opaque IDs own every runtime relation. External IDs live only in bounded source mappings/reconciliation/archive support.
5. Collections and ordered membership are part of the target architecture now because they drive home, Men/Women/Kids, and related products.
6. Product publication and variant sellability are explicit server rules. Collection membership and UI helpers cannot bypass them.
7. Inventory needs authoritative positions, movements, and concurrency-safe reservation/release/consume capability; a BMS/Shopify snapshot is migration evidence only.
8. Customer email/password identity, credentials, sessions, recovery, verification capability, profile, and addresses are native. Admin identity is separate.
9. Wishlist is a native Customer/Product relationship; snapshots and dual BMS owner types are retired.
10. Anonymous cart continuity remains supported. Checkout and order creation revalidate price, publication, stock, shipping and tax on the server. Buy Now reuses the same guarantees.
11. Checkout is a persisted orchestration lifecycle and creates at most one Order. Idempotency is mandatory across retry, payment callback, stock and order creation.
12. Order, payment and fulfillment states are separate. Orders/items/addresses/money are durable purchase snapshots.
13. Payment, shipping and tax remain provider-neutral domain boundaries until providers/policies are approved; Vastriqo persists normalized outcomes and reconciliation state.
14. Admin writes require independent admin authentication, API authorization, and audit. CCA modules and permanent Shopify sync are not part of `vastriqo-admin`.
15. Migration is versioned, deterministic, read-only toward Shopify/BMS, mapping-driven, dry-runnable, idempotent, conflict-aware and reconciled. Active carts do not migrate by default.
16. Source-controlled storefront content remains source-controlled initially. Disconnected BMS CMS data is retired unless a separately approved CMS requirement replaces it.
17. Current broken behavior—first-100 discovery, permissive availability, fake tracking, ineffective cancellation fallback, missing sitemap/logout assumptions, and stale BMS publication—is not preserved.
18. Current test-era customers, credentials, addresses, wishlists, historical orders, inquiries and newsletter records do not migrate and receive no dedicated archive or linking flow. Native customer/address/wishlist/order and any future inquiry/newsletter capabilities start with new Vastriqo records.
19. Cancellation, refund, return, exchange and customer-visible tracking are required future capabilities, but their detailed policies/workflows are designed progressively in the relevant post-purchase phase rather than inferred from broken current behavior.

## 15. Earlier Candidate Business Decisions — Resolved or Bounded by Closure

This historical list shows what the conceptual-architecture pass initially grouped as business-dependent. The final closure register now records **no unresolved Phase 1 business decision**. Items below are either resolved, engineering/data-validation work, or deliberately assigned to the relevant later implementation phase.

The original decision groups were:

1. **Catalog policy:** approved classification/audience/color/size/attribute vocabulary; SKU/barcode/weight/brand/tag/compare-at/multi-currency/variant-media/SEO requirements; publication minimums; handle and merchandising-date/ranking policy.
2. **Collections:** `/collections` information architecture and whether the current image override is product-wide, collection-specific, or retired.
3. **Inventory:** locations, oversell/backorder, reservation/deduction/release timing, adjustment roles/reasons, and cutover ownership timing.
4. **Customer/guest:** existing customer/address migration is resolved as **do not migrate**; the native system starts clean. Guest checkout, verification/phone policy, wider country support and native account privacy/retention details remain later implementation policy.
5. **Checkout commercial rules:** payment methods/provider/COD, shipping zones/services/rates/free shipping, tax/GST and invoice rules, discounts, fraud checks, quote/cart expiry, and required checkout consent.
6. **Order operations:** lifecycle and order numbering are detailed engineering design. Cancellation/refund/return/exchange and customer-visible tracking are required future capabilities whose workflows are designed progressively in the relevant post-purchase phase.
7. **Historical/auxiliary data:** current test orders, customer addresses, BMS wishlists, inquiries/attachments and newsletter records are all **do not migrate**. Active carts still expire/use a bounded bridge by default.
8. **Admin operating model:** initial roles/permissions, PII/address access, allowed order/payment/fulfillment actions, support/newsletter workflows, dashboards, bulk approval, and audit retention.
9. **Content and external boundary:** CMS need, permanent CCA/BMS integration if any, analytics/consent, and whether wallet/referrals ever become a Vastriqo feature.

## 16. Recommended Phase 2 Scope

Phase 2 should not build application code. The final closure register §23.6 is authoritative for its outputs. It should convert this conceptual model into detailed, reviewable contracts while scheduling production validation and later implementation-policy checkpoints before the affected contract is frozen:

1. Schedule the remaining production inspections and later legal/post-purchase/provider decisions from the closure register; they do not block Phase 2 start.
2. Record initialization ADRs from §11.2, including framework, database, identifiers, money, security/session and deployment baseline.
3. Produce logical and physical database design with tables, keys, indexes, constraints, deletion/retention strategy, transaction/isolation behavior and lifecycle enums.
4. Produce versioned OpenAPI contracts for public/customer/admin/integration surfaces, including errors, pagination, idempotency and authorization permissions.
5. Produce detailed identity/session/recovery/verification and admin RBAC threat model.
6. Produce inventory/cart/checkout/order/payment/fulfillment state and concurrency design, including failure/retry/reconciliation sequences.
7. Produce provider selection criteria/contracts for payment, shipping, tax, email and media, without embedding a provider into domain semantics.
8. Produce field-level Shopify/BMS mapping specifications, source inventories, dry-run/reconciliation format, cutover delta and rollback plan.
9. Produce admin information architecture/workflows against the approved API operations.
10. Define test, security, observability, performance, backup/restore, deployment and cutover acceptance plans.

After those designs are reviewed, Phase 3 may initialize/build `vastriqo-api`; Phase 4 may initialize/build `vastriqo-admin` against approved contracts. Repository scaffolding must not begin merely to “get started” before the prerequisites below exist.

## 17. Exact Prerequisites for Initializing `vastriqo-api`

All of the following must be recorded and approved:

1. Repository host/owner/access, peer directory location, default branch/protection, license/ownership, and fixed repository name `vastriqo-api`.
2. Approved business decisions affecting catalog, inventory, guest/customer, checkout, order, migration, and admin write scope—including the clean-start/no-history-import decisions—or explicit later-phase deferrals with safe domain boundaries.
3. Production source inventory for included migration/integration scope: product/variant/media counts and limits, configured metafields, active collections/overrides, inventory shape, checkout/provider configuration and relevant data-quality findings. Excluded test customer/order/auxiliary volumes are not prerequisites.
4. Backend runtime/framework and exact supported versions; npm version/lockfile policy; strict TypeScript/build/lint/format conventions.
5. Relational database engine/version/hosting, migration/query library, connection/pool policy, backup/restore expectation, and environment provisioning.
6. Native identifier, UTC timestamp, exact money/currency/rounding, text/collation, soft-delete/retention and audit conventions.
7. Module/layer/dependency rules; transaction/repository conventions; background job/outbox approach; provider adapter boundaries.
8. OpenAPI/versioning, success/error, pagination/filter/sort, idempotency, concurrency, compatibility and generated-client conventions.
9. Customer/admin session topology, credential/password/recovery baseline, CORS/CSRF/cookie policy, authorization model, rate limits, encryption and PII/secrets/log-redaction requirements.
10. API/deployment domains, local/dev/test/staging/prod configuration, secret manager, CI/CD, migration execution/rollback and release strategy.
11. Automated test stack and required unit/integration/contract/e2e/security/concurrency gates; health/readiness, structured logging, metrics, tracing/correlation and alert ownership.
12. Media storage/delivery decision sufficient for product-media import, plus read-only Shopify/BMS migration credential and network-access plan.
13. Approved initial implementation sequence and acceptance criteria, beginning with foundation/identity/admin security/catalog—not an unbounded scaffold with copied BMS modules.

## 18. Exact Prerequisites for Initializing `vastriqo-admin`

All of the following must be recorded and approved:

1. Repository host/owner/access, peer directory location, default branch/protection, and fixed repository name `vastriqo-admin`.
2. Approved admin information architecture and first implementation sequence covering catalog, collections, inventory, customers, orders, imports, support, and provider exceptions as applicable.
3. Approved AdminUser/session and permission contract in `vastriqo-api`; no admin write UI before server authorization exists.
4. Approved OpenAPI admin contracts, error/validation/concurrency behavior, generated type/client approach, and non-hard-coded environment API configuration.
5. Runtime/framework/build tool and exact supported versions; npm/lockfile policy; strict TypeScript/lint/format/test conventions.
6. Component library/design tokens, accessibility/browser support, responsive layout, form/table/filter/confirm/error patterns, and Vastriqo branding/assets.
7. Session/BFF/CORS/CSRF/logout behavior that avoids the BMS localStorage-token pattern; permission-aware route/navigation behavior.
8. PII display/redaction, audit visibility, destructive-action confirmation, re-authentication for sensitive operations, and import/payment/inventory privilege rules.
9. Local/dev/test/staging/prod configuration, admin domain/deployment, CI/CD, source maps/error monitoring, release/rollback and e2e test environment.
10. Explicit exclusion list for CCA/BMS modules, BMS API calls, Shopify sync/Admin links, BMS credentials and shared runtime code.

## 19. Phase 1 to Phase 2 Readiness

**Engineering conclusion:** the repository audit and conceptual architecture are complete. The system boundaries, domain ownership, required concepts, relationships, invariants, read/write responsibilities, provider boundaries, migration method, no-copy rules and application prerequisites are conceptually settled.

**Final gate conclusion:** `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` establishes that **Phase 1 is complete, every Phase 1 business decision is resolved, and Phase 2 detailed design may begin**. Remaining production-data validation and later implementation-phase policy decisions gate only their affected contract/cutover checkpoints; they do not block Phase 2 start.

This does not authorize repository initialization, application code, database migrations or changes to the existing applications.

## 20. Evidence Reference Index

### Storefront

- `vastriqo/lib/shopify/index.ts`
- `vastriqo/lib/shopify/customer-auth-client.ts`
- `vastriqo/lib/front-auth.ts`
- `vastriqo/lib/custom-auth-api.ts`
- `vastriqo/lib/api/custom-auth-route.ts`
- `vastriqo/lib/shopify-api-collection-products.ts`
- `vastriqo/lib/product-metafield-options.ts`
- `vastriqo/lib/internal-products.ts`
- `vastriqo/components/collections/sections/collections-grid-section/products-client.tsx`
- `vastriqo/components/collections/details-page/index.tsx`
- `vastriqo/components/collections/category-collection-page/index.tsx`
- `vastriqo/components/home/home-page/index.tsx`
- `vastriqo/components/cart/cart-context.tsx`
- `vastriqo/components/cart/cart-page.tsx`
- `vastriqo/components/account/account-pages/profile-page.tsx`
- `vastriqo/components/account/account-pages/address-page.tsx`
- `vastriqo/components/account/account-pages/orders-page.tsx`
- `vastriqo/components/account/account-pages/wishlist-page.tsx`
- `vastriqo/components/account/account-pages/inquiries-page.tsx`
- `vastriqo/app/api/auth/*`
- `vastriqo/app/api/cart/checkout/route.ts`
- `vastriqo/app/api/wishlist/*`
- `vastriqo/app/api/contact/submit/route.ts`
- `vastriqo/app/api/newsletter/subscribe/route.ts`
- `vastriqo/app/sitemap.ts`

### BMS API and data structures

- `bms-api/src/api/admin/modules/shopify-apis/{route,controller,service,customer-auth,checkout}.ts`
- `bms-api/src/api/admin/middlewares/shopify-customer-auth.ts`
- `bms-api/src/shared-services/{customer,customerToken,shopifyCollection,inquiry,newsletter}.ts`
- `bms-api/src/api/front/modules/{wishlist,inquiry,newsletter}/*`
- `bms-api/src/db-migrations/20260519121000_create_customers.ts`
- `bms-api/src/db-migrations/20260521120000_create_customer_tokens.ts`
- `bms-api/src/db-migrations/20260515100000_front_wishlist.ts`
- `bms-api/src/db-migrations/20260523110000_customer_wishlists.ts`
- `bms-api/src/db-migrations/20260527130000_shopify_product_collections.ts`
- product/collection follow-up migrations through `20260618110000_drop_shopify_category_products_product_fk.ts`
- `bms-api/src/db-migrations/20260514090000_inquiries.ts`
- `bms-api/src/db-migrations/20260619120000_newsletter_subscribers.ts`
- CCA product/order/invoice/transaction/user migrations, inspected only to enforce exclusion.

### Current admin and conventions

- `bms-admin/app/root.tsx`
- `bms-admin/app/components/ProtectedRoute.tsx`
- `bms-admin/app/lib/{auth,axios}.ts`
- `bms-admin/app/api-hooks/*`
- `bms-admin/app/components/shopify/{ProductsTab,CategoriesTab,CategoryDetail,CategoryProductsManager}.tsx`
- `bms-admin/app/components/inquiry/List.tsx`
- all three `package.json`, `package-lock.json`, `tsconfig.json`, and Dockerfiles
- `bms-api/src/config/*`, `src/loaders/express.ts`, `src/lib/api-response.ts`, and `src/lib/pagination.ts`
