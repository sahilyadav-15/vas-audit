# Vastriqo Phase 1 — Closure and Final Decision Register

Date: 2026-08-26 (Asia/Kolkata); final business-decision reconciliation recorded 2026-08-26
Status: **Phase 1 conceptually complete; all Phase 1 business decisions resolved; safe to begin Phase 2 detailed design**
Scope: final requirements/architecture reconciliation only; no application code, repository, physical schema, migration, provider selection, or external write

## 1. Purpose and Governing Rule

This document closes Phase 1 by reconciling and classifying the unresolved items in:

- `PHASE-1-FUNCTIONAL-DEPENDENCY-AUDIT.md`;
- `PHASE-1-REQUIREMENTS-AND-FEATURE-DISPOSITION.md`;
- `PRODUCT-CATALOG-DESIGN.md`;
- `PHASE-1-CONCEPTUAL-ARCHITECTURE.md`;
- `report.md`; and
- `Vastriqo Platform — Project Goal & Phased Direction.md`.

It is the authoritative Phase 1 decision/status overlay. The earlier documents remain authoritative for detailed source evidence and analysis. Their open-question sections are retained as historical discovery records, but they no longer act as the current phase gate where this document reclassifies or resolves an item.

The governing rule is:

> If the current Vastriqo storefront demonstrably requires a capability, it is a replacement requirement unless there is an explicit decision to change or retire it.

Therefore:

```text
CURRENT STOREFRONT BEHAVIOR
        |
        v
REQUIRED VASTRIQO BEHAVIOR
        |
        v
VASTRIQO DOMAIN MODEL
        |
        v
DATABASE CONCEPTS
        |
        v
API RESPONSIBILITIES
```

The reasoning must not run backwards from Shopify fields or BMS tables. Shopify/BMS implementation detail is evidence about current behavior and migration, not the target schema.

## 2. Final Classification Scheme

Every item revisited in this closure has exactly one classification:

| Code | Classification | Meaning |
| --- | --- | --- |
| **A** | **ALREADY DECIDED** | Established by the current storefront, source audit, or explicit project direction. It is not awaiting approval. |
| **B** | **ENGINEERING DECISION** | Architecture/source evidence is sufficient to choose a safe design direction. The decision and rationale are recorded here. |
| **C** | **PRODUCTION DATA VALIDATION** | Actual Shopify/BMS configuration or rows must be inspected. This validates mapping/volume/behavior; it is not converted into a questionnaire. |
| **D** | **BUSINESS DECISION** | Source cannot determine a choice that materially changes customer experience, operations, legal obligations, or historical-data scope. |
| **E** | **SAFE TO DEFER** | Optional or future capability with an explicit boundary. It does not block the domain model, database/API design, or Phase 2 start. |

“Not yet independently implemented” is never, by itself, a **D** classification.

## 3. Reconciliation of Earlier Phase 1 Documents

| Earlier issue or contradiction | Final reconciliation |
| --- | --- |
| `/collections` was repeatedly listed as needing a business choice. | Current behavior is an all-products alias, so preserving that is **A**. A true collection landing is **E**. |
| Collections were once treated as a possible later slice. | BMS collections drive home, Men/Women/Kids and related products; native Collection/Membership is **A** and included now. |
| Catalog taxonomy, brand, tags and SKU were grouped into one large business blocker. | Required discovery behavior is **A**; normalized representation and optional SKU handling are **B**; actual values/coverage are **C**; new customer-facing taxonomy/brand behavior is **E** unless explicitly requested. |
| Handle redirects, publication minimums and “new arrivals” were left open. | Alias-based handle continuity, explicit publication gating and a deliberate merchandising timestamp are **B**. Current placeholder media behavior means missing media does not silently become a Phase 1 business blocker. |
| BMS image override meaning was considered wholly undecidable. | Source proves it is product-wide in current persistence. Import it as product-wide card media only if **C** finds live differences; placement-specific override is **E**. |
| Inventory locations/reservation timing were treated as pre-architecture business blockers. | Location-scoped inventory, reservation before payment/order acceptance, expiry/release and atomic consumption are **B**. Actual locations and Shopify oversell policies are **C**. Backorders beyond validated current behavior are **E**. |
| Guest checkout was treated as undecided because Shopify may have caused the login requirement. | The storefront demonstrably requires login for checkout; authenticated checkout is **A**. Guest checkout is **E**. |
| Checkout phone/country rules were treated as open. | Current checkout requires first name, last name, a normalized 10-digit phone and India buyer context; preserving those is **A**. International commerce/phone verification are **E**. |
| Cart merge and Buy Now semantics were treated as business blockers. | Current behavior has no login merge and Buy Now creates a separate purchase cart. Preserve both as **A**; enhanced merge/cross-device behavior is **E**. |
| Tracking/cancellation/returns content was mixed with implemented behavior. | Fake tracking and ineffective cancellation behavior are not requirements. Honest contact-support cancellation request remains **A**; real self-service tracking/cancellation/returns/refunds/exchanges are required future capabilities whose detailed workflows/policies are **E** until the relevant post-purchase phase. |
| Provider choice was mixed with capability need. | Payment, shipping, tax and fulfillment boundaries are **A**; provider selection is **E** and happens in detailed design/procurement. Actual Shopify configuration is **C** to preserve enabled behavior. |
| Admin scope was broadly held for business approval. | Product, variants/options, media/specifications, publication, collections, inventory, customers, orders, payment/fulfillment operations, inquiries, imports/reconciliation and multiple independent admins are **A**. Complex RBAC/dashboard/CMS/generic settings are **E**. |
| Framework/database/package selection was presented as a Phase 1 blocker. | Strict TypeScript, Node ecosystem, npm, modular monolith, relational persistence and OpenAPI are **B**. Exact framework/database/version ADRs are Phase 2 outputs, not Phase 1 blockers. |
| Production inventories were treated as required before any Phase 2 design. | No production inspection blocks Phase 2 start. Selected checks must precede the relevant schema/import/provider contract freeze or migration dry run (§20). |
| The previous readiness conclusion said Phase 2 could not begin. | Superseded: the foundation is sufficiently bounded. Phase 1 is complete and Phase 2 may begin, subject to the non-implementation boundary in §23. |
| Customer, address, wishlist, inquiry, newsletter and historical-order migration were left as business choices. | The current records are test-era data and are explicitly excluded. New native capabilities remain required where already established, but the new platform starts with new customer accounts, addresses and orders. No account linking, activation, credential migration or dedicated archive is required. |
| Legal/financial migration and post-purchase scope were grouped with Phase 1 blockers. | No broad legal/financial history migration will be created; already available information may be reused only when safe and useful. Detailed legal policies and required future cancellation/refund/return/exchange/tracking workflows are handled in their relevant implementation phases and do not reopen Phase 1. |

## 4. Domain 1 — Product / Catalog

### 4.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Rich list/detail comes from Shopify; cards/detail consume native-seeming product identity/handle/title/descriptions, product type/discovery text, variants/options, current price/currency, ordered media, availability intent and displayed specifications. Search/filter/sort currently run over a capped first-100 list. |
| Future required capability | A complete, paginated, published native catalog; stable cards/detail; variant selection; exact price; ordered media; approved specifications; canonical handles; full-catalog search/filter/sort; sitemap input; server-authoritative publication and sellability. |
| Current external dependencies | Shopify Storefront/Admin GraphQL; BMS partial mirror for selected cards; external image URLs. |
| Vastriqo-owned replacement | Product aggregate, variants/options/selections, price, media, typed attributes, publication/read projections, discovery queries and external mappings. |
| Migration implication | Transform complete Shopify product data and required metafield/metaobject business values; use BMS only for comparison/curation joins; do not copy Shopify/BMS schema. |

### 4.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Products, variants, options/values and selections | **A** | Current detail and purchase flow require them. | Model/contracts required. |
| Product media and displayed specifications | **A** | Current card/detail behavior establishes ordered media, alt/fallback behavior and the required fabric/care/material/fit/etc. meanings. | Model/contracts required. |
| Search/filter/sort over complete catalog | **A** | Preserve behavior while fixing first-100/browser limitation. | Native query design required. |
| Product classification and searchable discovery terms | **B** | Use normalized business classification/search terms rather than raw Shopify tags. Seed from validated current product type/tags/vendor/audience evidence. | Detailed taxonomy representation belongs to Phase 2. |
| SKU representation | **B** | Support nullable SKU on Variant and uniqueness for non-empty approved values; SKU is operational identity, not customer purchase identity. | Include optional concept; do not require every variant. |
| Actual SKU coverage/duplicates/style-number use | **C** | Inspect Shopify variant SKU and BMS `style_no` values before final import validation/constraints. | Does not block Phase 2 start; blocks SKU import rules. |
| Barcode, physical weight and Shopify shipping/tax flags | **E** | Add only when selected fulfillment/shipping/tax rules require them; keep out of initial source mapping by default. | Provider/operations extension point only. |
| Compare-at/promotional pricing | **E** | Not displayed today. The money model can add price types later without importing unused values now. | No block. |
| Currency model | **B** | Every money value stores exact amount plus ISO currency; never hard-code currency into schema or use binary float. | Phase 2 chooses physical exact representation/rounding. |
| Actual currency usage | **C** | Inventory distinct product/variant/cart/order currencies and inconsistent BMS INR fallbacks. | Blocks price import mapping, not Phase 2 start. |
| Handle changes and redirects | **B** | Canonical handles remain unique; old published handles become aliases/redirects so bookmarks/SEO do not require Shopify lookup. | Include alias concept. |
| Publication lifecycle | **B** | Separate non-public/public/retired concepts and enforce public eligibility in every projection. Exact enum names/transitions are Phase 2 details. | Required design dimension. |
| Media required for publication | **B** | Preserve current fallback behavior: media is strongly validated but absence alone does not make the architecture invent a publication prohibition. | Admin validation can warn; rule can tighten later. |
| New-arrival/default ordering | **B** | Preserve current created-time intent using a deliberate merchandising/publication timestamp seeded from source; default order is stable merchandising order. Remove the false “Best Sellers” claim until a real rule exists. | Query/admin design required. |
| Attribute storage/governance | **B** | Model typed AttributeDefinition and ProductAttributeValue supporting scalar/list and controlled/free values. Current displayed meanings are required; vocabulary management is not a generic Shopify metafield editor. | Physical representation in Phase 2. |
| Actual configured metafields/metaobjects and values | **C** | Inspect production identifiers, types, cardinality, references and values, especially `styling_tips`, `about_this_item`, color, size and style number. | Required before catalog schema/import mapping freeze. |
| Brand/vendor presentation, variant-media switching, SEO overrides, arbitrary tags | **E** | Current experience does not require new dedicated UI/behavior. Preserve discovery meaning through search data; add richer behavior only later. | No block. |
| Product/media volumes and source truncation | **C** | Count all products, variants and media through complete source export/query. | Sizing/import validation only. |

## 5. Domain 2 — Collections and Merchandising

### 5.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | BMS active custom categories, unique slugs, ordered membership and four known placements drive home, Men/Women/Kids and shared-collection related products. `/collections` is currently an all-products alias. |
| Future required capability | Native Collection and CollectionMembership, visibility, ordered membership, home/category reads and deterministic related-product rule using Vastriqo Product IDs. |
| Current external dependencies | BMS `shopify_categories`, `shopify_category_products`, mirrored product IDs/images. |
| Vastriqo-owned replacement | Collection aggregate, ordered membership, publication-aware public projections and admin management. |
| Migration implication | Transform active/current BMS collection meaning and order after Shopify Product -> Vastriqo Product mapping. |

### 5.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Native collections, slugs, active state, membership/order | **A** | Required by multiple current surfaces. | Required model/admin/API. |
| Home/Men/Women/Kids and related products | **A** | Preserve current placement behavior; related initially uses shared visible Collection membership. | Required public queries. |
| `/collections` behavior | **A** | Preserve current all-products alias until an explicit later change. | No landing-page domain required now. |
| True collection landing, automated rules, scheduling, personalization, best-seller ranking | **E** | Future merchandising capabilities; no current behavior proves them. | No block. |
| Product publication enforcement | **B** | Collection reads must join/enforce current Product public eligibility; active membership cannot publish a stale product. | Query invariant required. |
| Related-product tie-breaker | **B** | Membership order first, then stable native Product ID/order; exclude current Product and duplicates. | Deterministic contract. |
| Audience representation | **B** | Use explicit audience terms seeded from current Men/Women/Kids collections/validated source values; retire title/tag guessing as authority. | Phase 2 taxonomy relation. |
| Actual collection rows, membership/order, staleness/orphans | **C** | Inspect active/inactive records, four expected slugs, memberships, duplicate orders, missing products and unpublished/deleted products. | Required before collection import mapping/dry run. |
| BMS card-image override persistence meaning | **B** | Source proves current storage is product-wide. Preserve it only as product-wide card media when live data confirms use; do not invent placement-level semantics. | No membership override field required now. |
| Actual image overrides | **C** | Compare `shopify_products.image_url` with Shopify featured media and inventory every difference/current use. | Required before media import mapping. |
| Placement-specific media override | **E** | Add later only if deliberately requested. | Explicit extension boundary. |

## 6. Domain 3 — Inventory

### 6.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Shopify ultimately decides variant availability and prevents/permits purchase; storefront availability is incomplete; BMS keeps only a partial quantity snapshot. |
| Future required capability | Authoritative native stock, available-to-sell, adjustments, movement history, reservation/release/consume and reconciliation across catalog/cart/checkout/order. |
| Current external dependencies | Shopify inventory policy/quantity/location behavior; BMS comparison snapshot. |
| Vastriqo-owned replacement | InventoryLocation, InventoryPosition, InventoryMovement and InventoryReservation concepts owned by Inventory module. |
| Migration implication | Recreate native inventory from an authoritative cutover snapshot/delta and reconcile by Variant mapping; never reconstruct fake historical movements from BMS. |

### 6.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Native stock, sellability and reconciliation | **A** | Required for independent sale acceptance. | Required domain/API/admin. |
| Location-scoped model | **B** | Model stock against InventoryLocation even if only one default location exists. This supports current single-scope use and future real locations without changing Variant identity. | Include conceptual relationship. |
| Actual Shopify locations/stock ownership | **C** | Inspect locations, fulfillment services, tracked/untracked variants and which quantity is authoritative. | Required before inventory schema/import freeze, not Phase 2 start. |
| Oversell/backorder baseline | **B** | Default to deny oversell. If production data proves Shopify `CONTINUE`/backorder behavior for specific variants, import an explicit per-variant policy; otherwise do not invent backorders. | Model policy extensibility. |
| Actual oversell policies | **C** | Inspect per-variant inventory policy and current negative/zero-stock selling behavior. | Required before sellability migration/cutover rules. |
| Reservation timing and expiry | **B** | Reserve atomically when a checkout becomes payment/order-pending; expire/release on timeout or terminal failure; consume once on accepted order. TTL is environment/config policy selected in Phase 2, not a schema question. | Required lifecycle/concurrency design. |
| Stock deduction versus fulfillment | **B** | Order acceptance consumes the reservation/commits stock. Fulfillment records physical progress and must not decrement the same quantity again. | Prevent double deduction. |
| Adjustments | **B** | Every adjustment requires signed quantity, reason, actor/source and append-only movement evidence in the same transaction as balance change. | Required admin/use-case design. |
| Adjustment reason catalogue and authorization depth | **B** | Use stable extensible reason codes plus notes; API authorization is mandatory. Exact roles may be expanded later. | Does not need business questionnaire. |
| Transfers, POs, receiving, lots/serials, forecasting | **E** | No current evidence; separate future inventory operations. | No block. |

## 7. Domain 4 — Customer and Authentication

### 7.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Customer account/login, profile, address CRUD/default, authenticated checkout and order history exist through Shopify/BMS. Checkout requires first/last name, normalized 10-digit phone and India buyer context. |
| Future required capability | Native email/password registration/login, recovery/verification capability, secure revocable sessions/logout, one native profile and customer addresses/default. The customer system starts clean and creates new accounts going forward. |
| Current external dependencies | Shopify Customer Accounts/Admin customer/address; BMS customer/token/JWT bridge; browser localStorage. |
| Vastriqo-owned replacement | Customer, CustomerCredential, CustomerSession, purpose-bound AccountToken and CustomerAddress, with admin identity completely separate. |
| Migration implication | Do not migrate Shopify/BMS customers, addresses, passwords, credentials, sessions or account mappings. New Vastriqo customers register directly and create native addresses going forward. |

### 7.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Customer accounts and email/password | **A** | Explicit project decision and current account behavior. | Required identity contracts. |
| Profile and address CRUD/default | **A** | Current customer behavior. | Required model/API. |
| Authenticated checkout | **A** | Current storefront explicitly requires login. | Checkout Customer relationship required; guest remains optional extension. |
| Checkout name/phone/India baseline | **A** | Preserve first name, last name, normalized 10-digit phone and India context. | Initial validation contract. |
| Session architecture | **B** | Server-revocable sessions; browser credential should use Secure/HttpOnly/SameSite cookie or BFF topology rather than a JavaScript-readable long-lived bearer token. | Phase 2 threat/session design. |
| Password recovery and email verification capability | **B** | Required by native email/password identity. Persist hashed, expiring, single-use purpose tokens; do not gate checkout on a new verification rule by default. | Required identity design. |
| Mandatory verification before purchase / phone verification | **E** | Not enforced by current source. May be added later as an explicit customer-policy change. | No block. |
| Guest checkout and international address support | **E** | Current flow is authenticated and India-oriented. Model can remain extensible without implementing either. | No block. |
| Login cart merge, cross-device sessions and all-device logout UI | **E** | Not current behavior. Session revocation capability still exists server-side. | No block. |
| Google/social login and CCA SSO | **E** | Previously explicitly deferred. | Identity provider extension boundary. |
| Existing customer/profile/address migration | **A** | Do not migrate the current test users or addresses. Do not create linking, activation, deduplication, password or credential migration logic for them. | Excluded from migration design; native identity/address design proceeds for new records. |
| Existing customer source inventory | **A** | No source counts or field-quality inspection is required for migration because the source class is excluded. Read-only inspection remains permissible only if later needed to retire a dependency safely. | No migration checkpoint. |
| Future customer privacy/retention policy | **E** | Define deletion, retention, consent and export behavior for new native records during the relevant identity/legal implementation design. | Required before production rollout, not a Phase 1 or product/catalog blocker. |

## 8. Domain 5 — Wishlist

### 8.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Authenticated customers list/add/toggle/remove saved products; one pending signed-out selection is applied after login. BMS stores Shopify product snapshots and dual owner types. |
| Future required capability | Native Customer/Product saved relationship, pending-login handoff, current Product projection and unavailable-product handling. |
| Current external dependencies | BMS wishlists, Shopify/BMS customer and product IDs. |
| Vastriqo-owned replacement | WishlistItem relation owned by Customer/Wishlist module. |
| Migration implication | Do not migrate current BMS wishlist rows or their snapshots. New wishlist relationships use only new native Customer and Product identities. |

### 8.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Wishlist and pending-login behavior | **A** | Required current feature. | Required API/model. |
| Ownership/uniqueness | **B** | One active Customer/Product relationship; add/remove are idempotent. Eliminate `user_id`/`customer_id` polymorphism. | Constraint/use-case required. |
| Unavailable/deleted Product behavior | **B** | Retain the relation for history but mark unavailable and prevent purchase; do not serve stale snapshot as live Product. Permanent cleanup follows retention rules. | Read projection required. |
| Existing wishlist migration | **A** | Do not migrate the current test-era BMS wishlist rows or snapshots. | Excluded from migration design; no source inventory or mapping is required. |
| Named/shared lists, alerts, wishlist analytics/admin visibility | **E** | No current requirement and privacy impact. | No block. |

## 9. Domain 6 — Cart

### 9.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Anonymous device-local cart supports add/read/update/remove and totals; checkout requires login. Buy Now creates a separate one-line cart. No login merge exists. |
| Future required capability | Native anonymous/customer-capable Cart/CartItem, secure ownership, persistence/expiry, native Variant IDs, exact server subtotal and price/publication/stock validation. |
| Current external dependencies | Shopify Cart API and IDs; browser localStorage pointer. |
| Vastriqo-owned replacement | Cart aggregate and line operations; storefront holds only a protected access/session reference. |
| Migration implication | Active Shopify carts expire or use a short transition bridge; they are not normal historical data migration. |

### 9.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Cart creation/persistence/line operations/totals | **A** | Current storefront requirement. | Required model/API. |
| Anonymous cart plus authenticated checkout | **A** | Preserve current ownership progression. | Cart supports anonymous access; checkout requires Customer. |
| No automatic login merge | **A** | Current behavior has no merge. Do not invent one for cutover. | Simpler initial contract. |
| Buy Now | **A** | Preserve as a separate direct-purchase cart/checkout path using identical validation. | Required command, not separate commerce domain. |
| Cart expiry | **B** | Server owns configurable expiry; expired/converted carts are immutable. Exact duration is Phase 2 operational configuration, not business schema. | Lifecycle field/index/cleanup design. |
| Same-variant lines and quantity rules | **B** | One line per Variant while no line customization exists; add increments quantity; quantity must be positive and constrained by validated stock/policy. | Unique relationship/command semantics. |
| Price handling | **B** | Cart may retain last-observed price for change messaging, but current Product/Variant price is authoritative and revalidated at every mutation/checkout. | Prevent stale client totals. |
| Saved/multiple/cross-device cart and future merge rules | **E** | No current requirement. | Extension boundary. |
| Active-cart migration | **E** | Default to expiry/temporary bridge at cutover, not durable import. | No Phase 2 data-migration block. |

## 10. Domain 7 — Checkout

### 10.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Signed-in customer proceeds from cart/Buy Now, supplies required delivery contact, then Shopify performs address, shipping, tax, payment and order creation. |
| Future required capability | Persisted native checkout orchestration: validate lines/prices/publication/stock, capture contact/address, quote/select shipping/tax/payment, reserve stock and create one order idempotently. |
| Current external dependencies | BMS customer validity/token; Shopify cart/hosted checkout/payment/order; hidden providers/config. |
| Vastriqo-owned replacement | Checkout aggregate/application workflow coordinating Inventory, Shipping, Tax, Payment and Order modules through provider adapters. |
| Migration implication | Checkouts are new native operational records; no Shopify checkout schema import. Active in-flight transition is a cutover concern. |

### 10.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Native checkout capability and authenticated customer | **A** | Current purchase path and project independence establish both. | Required design. |
| Contact/address and India baseline | **A** | Preserve current first/last/phone/India behavior; address book supplies defaults but Order captures snapshot. | Required validation/projection. |
| Shipping/tax/payment capability | **A** | Necessary to reproduce completed purchase even though Shopify hides implementation. | Required module boundaries. |
| Persisted checkout/idempotency | **B** | Persist lifecycle, quotes, reservation and external attempts; one Checkout creates at most one accepted Order. | Required transaction/concurrency design. |
| Quote expiry/revalidation/recovery | **B** | Quotes/reservations expire; retry revalidates price/stock and safely resumes without duplicate order/payment. | Required contract. |
| Actual enabled payment methods/COD/discount/shipping/tax/fraud behavior | **C** | Inspect Shopify production checkout/settings/test orders. Preserve only demonstrably enabled behavior or explicitly changed policy. | Required before provider/integration contract freeze, not Phase 2 start. |
| Provider selection | **E** | Phase 2/procurement decision behind stable boundaries. | No domain-model block. |
| Guest checkout, international checkout, stored payment methods, gift wrapping/messages, complex promotions, multi-shipment | **E** | Not established by current runtime. | No block. |
| Checkout legal consent wording/requirements | **E** | Detailed legal/business approval is intentionally handled in the checkout implementation phase; the architecture retains consent/evidence boundaries without inventing wording now. | Required before final checkout production contract/content, not Phase 1. |

## 11. Domain 8 — Orders

### 11.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Authenticated customers see recent order cards with number/date, financial/fulfillment status, money totals, payment gateway, items and shipping address. Shopify owns order creation and fulfillment. Tracking/cancellation UI is not functional. |
| Future required capability | Native accepted Order/OrderItem/address/money snapshots, separate order/payment/fulfillment dimensions, customer paginated history/detail, admin search/operations and minimum fulfillment/shipment capability. |
| Current external dependencies | Shopify Admin orders/transactions/fulfillment; BMS proxy. BMS CCA orders are unrelated. |
| Vastriqo-owned replacement | Order aggregate plus Payment/Fulfillment/Shipment relations and immutable history. |
| Migration implication | New orders are native. Do not import or build a dedicated archive for current Shopify/BMS test orders; never use BMS CCA order tables. |

### 11.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Native order creation/history/detail/items/money/address | **A** | Current order history and independent checkout establish these concepts. | Required model/API/admin. |
| Payment and fulfillment state dimensions | **A** | Current cards expose both; independent operations need both. | Separate lifecycle concepts. |
| Fulfillment/shipment operational capability | **A** | Accepted physical orders must be completed independently even though current account UI ignores details. | Required admin/domain boundary. |
| Pagination and immutable snapshots | **B** | Replace latest-10 cap with authorized stable pagination; snapshot commercial data so later catalog/address edits do not rewrite history. | Required design. |
| Order number | **B** | Use a unique human-readable public number separate from opaque ID. Exact format/sequence and enumeration protection are Phase 2 engineering details. | Required identifier concept. |
| Lifecycle design | **B** | Keep order, payment and fulfillment states separate; Phase 2 defines minimal transitions needed for placement, failure, cancellation-by-support and fulfillment without cloning Shopify enum names. | Detailed design task. |
| Broken tracking button and cancel subject fallback | **B** | Remove false tracking action; preserve an honest support-contact path and correctly carry order reference. Do not model broken routing as cancellation. | Storefront later fixes after API exists. |
| Actual order statuses, transactions, fulfillments, tracking and notification behavior | **C** | Inspect representative current behavior only when designing the applicable native order/provider workflow; current test orders are not a migration source. | Required before the applicable provider/workflow contract freeze, not for historical mapping. |
| Historical Shopify/BMS order treatment | **A** | Do not migrate current test orders and do not build a dedicated archive for them. | Excluded from migration design; native Order starts with post-cutover orders. |
| Tax invoice/GST document obligations | **E** | Keep extensible evidence/document boundaries and approve detailed legal/finance policy in the relevant implementation phase. Do not invent or broadly migrate historical legal/financial records. | Required before final tax/invoice production contract, not Phase 1. |
| Customer self-service tracking/cancellation/refund/return/exchange and partial fulfillment | **E** | These are required future capabilities, but the broken/incomplete current UI is not their specification. Inspect, obtain policy decisions and design them progressively in the relevant order/post-purchase phase. | No Product/Catalog or Phase 2 foundation block; not retired. |
| Transactional notifications | **C** | Inspect actual Shopify notifications delivered today. If active, the event behavior is a replacement requirement; channel/provider selection remains deferred. | Required before notification integration contract/cutover. |

## 12. Domain 9 — Payment, Shipping and Tax

### 12.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Shopify checkout accepts payment, calculates shipping and tax, creates the order and likely performs configured external effects; repository source does not expose configuration. Order UI displays shipping/tax/payment results. |
| Future required capability | Provider-neutral but Vastriqo-owned payment attempts/transactions, shipping quotes/selections/fulfillment handoff, tax quote/lines/evidence, verified callbacks, retry/idempotency and reconciliation. |
| Current external dependencies | Shopify plus unknown configured gateway, carrier/rate, tax, notification and risk services. |
| Vastriqo-owned replacement | Stable application/domain records and adapters; provider is not domain authority. |
| Migration implication | Recreate integrations/configuration for new native orders. Current test order/payment/fulfillment history is excluded; do not migrate credentials or raw provider payloads as domain data. |

### 12.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Payment, shipping, tax and fulfillment boundaries | **A** | Required to replace completed Shopify commerce. | Required modules/contracts. |
| Exact money/outcome persistence | **B** | Persist requested/accepted amount+currency, normalized method/service/tax result, provider reference, timestamps and reconciliation state; no sensitive card data. | Required design. |
| Webhook verification/replay/idempotency | **B** | Unique provider event/reference, signature verification, safe replay/out-of-order processing and idempotent commands. | Required integration architecture. |
| Reconciliation and exception handling | **B** | Scheduled/on-demand reconciliation with explicit mismatches and admin queue; frequency is operational configuration. | Required design/admin projection. |
| Actual gateway, payment methods, COD and discounts | **C** | Inspect production checkout configuration and representative orders. Static FAQ claims are insufficient. | Required before payment adapter/migration mapping freeze. |
| Actual shipping zones/services/rates/free-shipping/carrier/manual behavior | **C** | Inspect production shipping profiles/settings and orders. | Required before shipping contract/config freeze. |
| Actual tax/GST calculation and invoice output | **C** | Inspect Shopify tax settings, registrations, representative orders and invoices. | Required before tax/invoice contract freeze. |
| Actual fraud and notification services | **C** | Inspect installed/configured services and observed checkout/order events. | Determines replacement scope before cutover. |
| Provider selection, multiple providers, stored instruments, international duties, advanced promotion engine, wallet/split tender | **E** | Provider/procurement/future capabilities behind stable ports. | No Phase 2 start block. |
| Legal tax/privacy/payment retention obligations | **E** | Detailed policies are deliberately assigned to their relevant implementation phase; retain appropriate evidence/extension boundaries without inventing policy now. | Required before the affected production contract, not Phase 1. |
| Refund/return commercial policy | **E** | Cancellation/refund/return/exchange are required future capabilities. Their exact policy/workflow is designed progressively during the relevant post-purchase phase. | No Product/Catalog or foundation block; capability is not retired. |

## 13. Domain 10 — Admin Requirements

### 13.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | BMS admin manages collections and inquiry status and inspects Shopify mirror; Shopify Admin owns products/inventory/retail orders/customer commerce. Project direction explicitly requires separate native admin and multiple accounts. |
| Future required capability | Independent `vastriqo-admin` managing required first-party commerce through authorized `vastriqo-api`; independent admin sessions, multiple accounts, audit and migration/reconciliation tooling. |
| Current external dependencies | Shopify Admin and BMS admin/API/database. |
| Vastriqo-owned replacement | Admin identity and authorized API operations; no direct DB/provider/Shopify/BMS browser calls. |
| Migration implication | Admin itself is recreated. BMS screens are interaction evidence only; CCA modules/data do not migrate. |

### 13.2 Required management scope

| Scope | Status | Boundary |
| --- | --- | --- |
| Products, variants/options, price, specifications, media, publication | **A** | First-party structured editors; no generic JSON/metafield editor. |
| Collections/membership/order and public preview | **A** | Native Product IDs and publication-aware results. |
| Inventory positions, movements, reservations, adjustments | **A** | Server-authorized operations with reason/audit. |
| Customers and necessary order-support profile view | **A** | Minimum PII, redaction and audit; customer self-service remains primary for addresses. |
| Orders, payment exceptions, fulfillment/shipping progress | **A** | Only approved state-changing commands; no generic status editing. |
| Inquiries/support status | **A** | Current BMS admin behavior, corrected customer ownership. |
| Migration preview/run/issues/reconciliation | **A** | Elevated tooling, mapping provenance and no source mutation. |
| Multiple independent admin accounts | **A** | No shared BMS users/secrets. |

### 13.3 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Admin authentication/security | **B** | Separate AdminUser/AdminSession; server authorization; secure cookie/BFF topology; no localStorage token as final design. | Required auth design. |
| Initial authorization structure | **B** | Support multiple accounts and permission checks by operation. A simple initial administrator role is enough; model must not require complex organizational RBAC. | Detailed permissions in Phase 2. |
| Audit | **B** | Append-only actor/action/target/result/correlation evidence with safe change summary for security and commerce writes. | Required platform concept. |
| Customer address editing by staff, wishlist visibility, bulk approval workflows | **E** | Not required by current admin behavior; add only for approved support need/privacy controls. | No block. |
| Dashboard, CMS/navigation, newsletter campaign UI, generic settings, advanced analytics | **E** | No current operational requirement. Source content remains source-controlled; subscription API retains consent/status. | No block. |
| Complex roles/permissions/SSO | **E** | Architecture keeps permission extension; no CCA/BMS shared auth. | No block. |
| Advanced cancellation/refund/return/exchange/tracking operations | **E** | Added with approved order policy, not generic admin status mutation. | No block. |
| CCA users/products/orders/invoices/transactions/seller/IoT/wallet modules | **A** | Explicitly outside Vastriqo and must not appear in `vastriqo-admin`. | Exclusion requirement. |
| Newsletter export/suppression and PII/audit retention | **E** | Existing subscriber data will not migrate. Define policy only if/when the future native capability and admin workflow are implemented. | No core admin or Phase 2 block. |

## 14. Domain 11 — Migration Mapping

### 14.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Shopify owns core source data; BMS owns relevant merchandising and auxiliary records and contains unrelated CCA data. Current queries are capped/incomplete. |
| Future required capability | Read-only extraction, versioned transformation, external mappings, dry-run, idempotent target writes, validation, reconciliation, delta/cutover and no source mutation. |
| Current external dependencies | Shopify exports/APIs, BMS database/API and current source-controlled assets/content. |
| Vastriqo-owned replacement | ImportRun, ImportRecord/Issue, ExternalSourceMapping, protected staging and target module import application services. |
| Migration implication | Use source identity only in mappings/provenance; never as native PK/FK. Protect native edits and report disappearance/conflicts. |

### 14.2 Shopify migration classes

| Data/capability | Class | Treatment |
| --- | --- | --- |
| Required Product/Variant/Option/Selection/Price data | **TRANSFORM** | Normalize into native catalog and mappings. |
| Required media | **MIGRATE** | Copy/rehost selected ordered media with alt/integrity validation. |
| Displayed metafield/metaobject specifications | **TRANSFORM** | Map readable business meanings into native attributes; no namespace/GID coupling. |
| Publication/handle/merchandising timestamps | **TRANSFORM** | Map to explicit native publication/alias/merchandising concepts. |
| Inventory | **RECREATE** | Initialize positions from authoritative cutover snapshot/delta; create initialization movements and reconcile. |
| Customer/profile/address data | **DO NOT MIGRATE** | Current records are test-era data. Start native accounts clean; no linking, activation, password/credential or address import. |
| Historical orders/transactions/fulfillment | **DO NOT MIGRATE** | Current test orders need neither customer-visible history nor a dedicated archive. Native Order starts with post-cutover orders. |
| Active carts/checkouts | **DEFER** | Expire or use a short-lived cutover bridge; not normal import. |
| Shopify IDs needed for included product/catalog mapping or route continuity | **ARCHIVE/RETAIN TEMPORARILY** | Store bounded ExternalSourceMapping only for included records/reconciliation/redirects; never create mappings for excluded customers, addresses or test orders and never use source IDs as native identity. |
| Raw GraphQL shape, unused fields, access/customer tokens, sessions, secrets | **DO NOT MIGRATE** | Not domain data; revoke/delete after dependency exit. |

### 14.3 BMS migration classes

| Data/capability | Class | Treatment |
| --- | --- | --- |
| Active custom collections and membership/order | **TRANSFORM** | Map through native Product IDs; validate staleness/orphans/order. |
| Live product-wide card image overrides | **TRANSFORM** | Only where data validation proves a difference/current use. |
| Product mirror rows/JSON and source/sync metadata | **ARCHIVE/RETAIN TEMPORARILY** | Reconciliation evidence only; never Product schema. |
| Wishlists | **DO NOT MIGRATE** | Current test-era relationships/snapshots are excluded. New native wishlist capability remains required. |
| Inquiries | **DO NOT MIGRATE** | Current test-era support/inquiry records and attachments are excluded. Future native inquiry capability remains independent of this data decision. |
| Newsletter subscribers | **DO NOT MIGRATE** | Current test-era subscriber/consent rows are excluded. Future native subscription capability remains independent of this data decision. |
| Source-controlled storefront content/navigation | **RECREATE** | Keep in `vastriqo` source initially; do not import disconnected BMS CMS. |
| Shopify/BMS external correlation IDs | **ARCHIVE/RETAIN TEMPORARILY** | Bounded mappings only for included product/catalog/collection/inventory import, reconciliation or redirects; none for excluded customer/order/auxiliary records. |
| Customer/access/admin tokens, JWT/session data, Shopify admin tokens | **DO NOT MIGRATE** | Recreate credentials/sessions securely. |
| BMS storefront pages/menus, wallet/referral side effects | **DO NOT MIGRATE** | No active storefront requirement; future features are separate. |
| CCA products/orders/users/invoices/transactions/seller/fabric/IoT/finance data | **DO NOT MIGRATE** | Outside Vastriqo domain. |

### 14.4 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Deterministic/idempotent/read-only migration protocol | **B** | Unique `(source system, type, ID)` mapping; versioned manifest/config/hash; dry run; relationship passes; safe rerun; conflict/tombstone reporting; no source writes. | Detailed specification in Phase 2. |
| Source completeness/counts/quality | **C** | Execute checklist in §20 using complete sources. | Required before mapping freeze/dry run. |
| Product/collection/inventory migration | **A** | Required for independent catalog cutover. | Required Phase 2 mapping design. |
| Customer/address/wishlist historical import | **A** | Do not migrate current test-era records or create account-linking/activation logic. | Excluded from mapping and validation scope. |
| Historical order import/archive | **A** | Do not migrate or create a dedicated archive for current test orders. | Excluded from mapping and validation scope. |
| Inquiry/newsletter historical retention/import | **A** | Do not migrate current test-era records. New native capabilities remain separate requirements. | Excluded from mapping and validation scope. |
| Existing legal/financial information | **A** | No broad historical migration project. Carry forward only already available information that proves safely and straightforwardly reusable for an included target requirement. | Case-by-case mapping evidence only; no dedicated migration stream. |
| Active cart migration | **E** | Default expire/bridge; no durable migration work now. | No block. |
| Continuous Shopify/BMS sync after cutover | **E** | Explicitly outside final architecture. Only bounded cutover delta is allowed. | No block. |
| Source record disappearance | **B** | Produce tombstone discrepancy; never delete native records automatically during import. | Required reconciliation rule. |

## 15. Domain 12 — API and Database Design

### 15.1 Established-to-target trace

| Concern | Final statement |
| --- | --- |
| Current established behavior | Storefront and BMS expose inconsistent direct/proxy contracts; BMS has useful Node/TS/relational/migration/layering experience plus significant security/testing/coupling debt. |
| Future required capability | Independent modular API and dedicated relational database with public/customer/admin/integration/import boundaries, explicit contracts, transactions, security, tests and observability. |
| Current external dependencies | None that must dictate target framework/schema. Existing Node/TypeScript/npm knowledge is useful. |
| Vastriqo-owned replacement | `vastriqo-api` modular monolith and dedicated Vastriqo database; `vastriqo-admin` is a separate client. |
| Migration implication | Import tooling uses native application invariants and bounded external mappings; public API never exposes Shopify/BMS-shaped payloads. |

### 15.2 Final unresolved-item classification

| Item | Class | Resolution / boundary | Phase 2 effect |
| --- | --- | --- | --- |
| Separate API/admin/database and no final Shopify/BMS dependency | **A** | Explicit project direction. | Non-negotiable boundary. |
| Backend style | **B** | Node ecosystem, strict TypeScript, npm and modular monolith. Retain useful layering knowledge without copying BMS code/global state. | Phase 2 selects exact framework/version. |
| Persistence class | **B** | Transactional relational database with explicit migrations, constraints and UTC timestamps. | Phase 2 ADR selects engine/version/query layer. |
| Native identity | **B** | Opaque Vastriqo IDs in every runtime relation/contract; external source IDs only in authorized mappings. | Phase 2 selects physical ID format. |
| Money/time | **B** | Exact amount+ISO currency; no binary floats; UTC persistence and explicit display timezone. | Phase 2 selects storage/rounding details. |
| API style/contracts | **B** | JSON/HTTP with OpenAPI; stable read projections plus explicit business commands; consistent errors/pagination/idempotency. | Detailed contracts are Phase 2 output. |
| Transactions/concurrency | **B** | Application use cases own transaction boundaries; optimistic/locking controls where needed; response helpers never commit; external calls use persisted attempts/events/outbox rather than fake distributed transactions. | Required detailed sequences. |
| Browser auth boundary | **B** | Secure revocable customer/admin sessions, cookie/BFF/CSRF/CORS design; no long-lived localStorage bearer-token target. | Phase 2 threat model. |
| Test/observability foundation | **B** | Unit/integration/contract/e2e/security/concurrency tests, structured redacted logs, correlation IDs, health/readiness, metrics/alerts and backup/restore proof are initialization requirements. | Phase 2 defines tools/gates. |
| Exact framework, DB engine/version, ORM/query tool, package versions, deployment/cloud, storage provider | **B** | Engineering ADRs evaluated and selected in Phase 2 against the fixed architecture; not business questions. | Phase 2 output, not Phase 1 block. |
| GraphQL, microservices, external search engine, Redis/cache, event-stream platform, warehouse | **E** | Add only on measured need. Module boundaries/outbox preserve future options. | No block. |
| BMS schema/code/auth/response conventions as compatibility target | **A** | Explicitly rejected. | Must not appear in detailed target design. |

## 16. Complete Conceptual Data Model Disposition

This table evaluates concepts; it is not a physical table list.

| Concept | Why/current evidence | Owner and key relationships | Important invariants | Disposition |
| --- | --- | --- | --- | --- |
| Product | Cards/detail/search/SEO | Catalog; has Variants, Options, Media, Attributes, Memberships | Unique canonical handle; public gating | **Required now** |
| ProductVariant | Option selection, price, cart merchandise | Product; Prices, Selections, Inventory, Cart/Order refs | Unique valid option combination per Product | **Required now** |
| ProductOption | Selector/filter dimensions | Product; owns ordered Values | Unique name/code per Product; deliberate dimensions only | **Required now** |
| ProductOptionValue | Size/color/etc. values | ProductOption; selected by Variants | Belongs to Product option; stable order | **Required now** |
| VariantOptionSelection | Exact purchasable combination | Variant -> OptionValue | One value per option; no foreign-product value | **Required now** |
| ProductMedia | Card/gallery/SEO/cart snapshots | Product -> MediaAsset; optional Variant later | Deterministic order/featured rule; durable source | **Required now** |
| AttributeDefinition | Displayed specifications | Catalog vocabulary -> ProductAttributeValue | Stable business code/type/cardinality | **Required now** |
| ProductAttributeValue | Fabric/care/material/etc. | Product -> AttributeDefinition | Type/cardinality valid; ordered list when needed | **Required now** |
| Collection | Home/gender/related | Merchandising; has Memberships | Unique slug; visible state | **Required now** |
| CollectionMembership | Curated ordered products | Collection -> Product | Unique pair; deterministic order; publication-aware read | **Required now** |
| InventoryLocation | Stock scope | Inventory; has Positions/Movements/Reservations | Stable native location; one default permitted | **Required now** |
| InventoryPosition | Authoritative quantities | Variant + Location | Unique pair; concurrency protected; non-negative rules per policy | **Required now** |
| InventoryMovement | Adjustment/order traceability | Variant/Location/source/actor | Append-only; atomic with balance change | **Required now** |
| InventoryReservation | Prevent oversell during checkout | Variant/Location -> Checkout/Order | Held/released/expired/consumed exactly once | **Required now** |
| Customer | Account/profile/history | Identity; Credentials, Sessions, Addresses, Wishlist, Orders | Unique normalized email; PII authorization | **Required now** |
| CustomerCredential | Email/password ownership | Customer | Secure hash only; lifecycle/revocation | **Required now** |
| CustomerSession | Login/current user/logout | Customer | Expiring, revocable, secret not logged | **Required now** |
| AccountToken | Recovery/verification/activation | Customer | Hashed, purpose-bound, expiring, single-use | **Required now** |
| CustomerAddress | Address book/default | Customer; source for Checkout, snapshot on Order | At most one default; owned authorization | **Required now** |
| WishlistItem | Saved products | Customer -> Product | Unique pair; unavailable Product handled | **Required now** |
| Cart | Anonymous selection/persistence | Cart module; owner/access -> CartItems; Checkout | Active/expired/converted; protected access | **Required now** |
| CartItem | Variant/quantity/current totals | Cart -> Variant | Positive quantity; one Variant line initially; server validation | **Required now** |
| Checkout | Persisted orchestration/idempotency | Cart/Customer -> quotes/reservations/payment/order | At most one accepted Order; quote/stock revalidation | **Required now** |
| ShippingQuote/Selection | Current hosted checkout responsibility | Checkout -> provider adapter; snapshot on Order | Current unexpired selection; exact money | **Required now** |
| TaxQuote/TaxLine | Tax displayed on orders | Checkout/Order/Item -> provider/rules | Exact reconciled amount; retained evidence | **Required now** |
| Order | Accepted purchase/history | Customer/Checkout; Items, Addresses, Payment, Fulfillment | Exactly-once creation; immutable commercial snapshot | **Required now** |
| OrderItem | History line display | Order -> optional live Product/Variant reference | Snapshot survives catalog change/deletion; totals reconcile | **Required now** |
| OrderAddress | Shipping history | Order | Immutable snapshot, not mutable CustomerAddress FK | **Required now** |
| PaymentAttempt/Transaction | Hosted payment replacement/status | Checkout/Order -> provider | Idempotent, verified, exact money, no card data | **Required now** |
| Fulfillment/FulfillmentItem | Complete physical orders | Order/OrderItem | Quantities cannot exceed ordered/eligible quantities | **Required now** |
| Shipment | Shipping handoff | Fulfillment -> provider/tracking | Normalized provider refs; customer visibility policy | **Required now** operationally; customer tracking deferred |
| TrackingEvent | No current functional UI | Shipment | Ordered verified events | **Conditional** on provider/customer tracking |
| Inquiry | Current public submission/admin status | Support; optional Customer plus contact snapshot | Correct Customer ownership; status/audit | **Required now** |
| InquiryAttachment | BMS capability unused by storefront | Inquiry -> MediaAsset | File safety/retention/authorization | **Deferred** |
| NewsletterSubscription | Current home capture | Engagement; normalized email | Unique email, explicit subscription/consent state | **Required now** |
| AdminUser/AdminSession | Separate multiple admins | Admin identity | Separate from Customer/BMS; revocable | **Required now** |
| Role/Permission assignment | Future complex authorization | Admin identity -> operations | API enforcement; least privilege | **Conditional**; simple initial admin role sufficient |
| AuditEvent | Admin/commerce traceability | Cross-module actor/target | Append-only; redacted; attributable | **Required now** |
| ExternalSourceMapping | Shopify/BMS migration | Import -> approved target entity | Unique source tuple; never native identity | **Required now for migration** |
| ImportRun/ImportRecord/ImportIssue | Repeatability/reconciliation | Import module | Versioned manifest/hash/result; safe rerun | **Required now for migration** |
| ProviderEvent | Webhook dedupe/replay | Integration -> Payment/Shipping/etc. | Unique verified event; idempotent processing | **Required now where callbacks exist** |
| StorefrontContent/Menu | Current content is source-controlled | Storefront source | Deployment/version control | **Deferred as database concepts** |
| Wallet/Referral | No current storefront consumer | None in current target | No automatic login side effects | **Deferred; do not migrate/build** |

## 17. `vastriqo-api` Responsibility Model

No endpoint paths are fixed here.

| Responsibility group | Authoritative owner | Reads | Writes/commands | Server-authoritative validation | External integration |
| --- | --- | --- | --- | --- | --- |
| Public catalog | Catalog | Published paginated cards/search/filter/sort/sitemap | None public except measured events if approved | Publication, price projection, sellability, pagination | Media delivery only |
| Product detail | Catalog + Inventory projection | Product/variants/options/media/specifications/price/availability by handle | None public | Handle/public eligibility, valid options, current sellability | Media delivery |
| Collections/merchandising | Merchandising + Catalog | Visible collections, ordered products, home/gender, related | Admin only | Collection/Product visibility and deterministic order | None final |
| Customer identity/profile | Identity/Customer | Current customer/profile/address/session view | Register/login/logout/recover/verify/profile/address | Credential/session security, ownership, normalization, default address | Email delivery later |
| Wishlist | Wishlist | Current saved Product projections | Add/remove/toggle | Customer ownership, Product existence, uniqueness | None |
| Cart | Cart + Catalog/Inventory reads | Current cart/lines/totals/errors | Create/add/change/remove/direct-purchase | Access ownership, quantity, current price/publication/stock | None |
| Checkout | Checkout coordinator | Current state, quotes, required action | Start/update/select/submit/resume | Customer/contact/address, price/stock, quote freshness, idempotency | Payment/shipping/tax adapters |
| Orders | Orders | Authorized list/detail/timeline | Create only via checkout; approved support commands | Exactly-once, snapshot reconciliation, ownership/state | Payment/fulfillment references |
| Payment | Payment | Attempt/transaction/support projection | Initiate/process verified event/reconcile/refund if later approved | Exact money, idempotency, signature/state | Payment provider |
| Shipping/fulfillment | Shipping/Fulfillment | Options/status/shipment projection | Quote/select/fulfill/ship/process event | Address/service/quantity/state | Shipping/carrier or manual adapter |
| Tax | Tax | Quote/line/evidence projection | Calculate/refresh | Jurisdiction/input/rounding/evidence | Provider or internal rules later |
| Admin operations | Owning domain modules | Draft/operational/search/audit/import views | Structured catalog/inventory/customer/order/support commands | Admin session, permission, domain invariant, audit | Never direct browser provider credentials |
| Migration/import | Import + target modules | Preview/run/issues/mappings | Dry-run/import/resume/delta | Mapping uniqueness, source hash, target invariant, conflict policy | Read-only Shopify/BMS source |
| Reconciliation | Owning modules + Import/Integration | Counts/differences/exceptions | Acknowledge/retry/correct through explicit command | No silent overwrite/delete; attributable resolution | Source/provider comparison |

The public API must not mirror Shopify GraphQL connections, GIDs, metafield JSON, BMS response envelopes or BMS table columns.

## 18. `vastriqo-admin` Architecture

### 18.1 Required first-party commerce management

- Product draft/edit/publication, variants/options/prices, structured specifications and media.
- Collections, active/visible state, ordered membership and public preview.
- Inventory positions, movements, reservations and authorized adjustments.
- Customer lookup and minimum support profile/order context with PII controls.
- Orders, payment exceptions, fulfillment/shipment progress and only approved actions.
- Inquiries and support status.
- Multiple independent admin accounts, secure sessions, operation authorization and audit.

### 18.2 Migration/reconciliation tooling

- Source inventory and import preview.
- Creates/updates/unchanged/warnings/failures/conflicts.
- External mapping/provenance for diagnosis, not editable native identity.
- Run/resume status and reconciliation outputs.
- Elevated authorization and attributable resolutions.

### 18.3 Safe future/optional modules

- dashboards after operational metrics are known;
- complex RBAC/approval workflows and SSO;
- CMS/navigation/announcement scheduling;
- newsletter campaigns/export, wishlist analytics and generic merchandising automation;
- customer self-service return/cancel/tracking administration;
- promotions, wallet/referrals and advanced settings.

### 18.4 Must remain outside Vastriqo

- BMS/CCA employees/vendors/suppliers;
- CCA internal products and production orders;
- invoices, expenses, payments, salaries and transactions;
- seller/bank/tax profiles, fabrics and IoT;
- BMS storefront-page/menu records with no current renderer;
- Shopify mirror sync as a permanent admin operation; and
- shared BMS authentication, database access, secrets or JWTs.

## 19. Migration Execution Requirements

The migration design is fixed at the protocol level:

1. Read Shopify/BMS only through approved read-only exports/APIs/database credentials.
2. Record immutable source manifest, snapshot/watermark, mapping version and source hashes.
3. Stage required source evidence outside core domain records with bounded retention/access.
4. Normalize through versioned deterministic transforms.
5. Validate identities, parents, collisions, money, option combinations, media, publication, inventory and relationships before target publication.
6. Upsert by unique external mapping, never title/handle/email alone.
7. Resolve dependent relationships only after parent mappings exist.
8. Update only approved import-owned fields and raise conflicts against later native edits.
9. Report source/target counts, hashes, monetary totals, unresolved records and exclusions.
10. Prove identical rerun creates no duplicates or business-data changes.
11. Apply bounded cutover delta and reconcile before switching authority.
12. Never delete a target record merely because a source row disappeared; produce a tombstone discrepancy for review.
13. Never modify Shopify/BMS.

No migration script or physical migration is authorized by this Phase 1 closure.

## 20. Production Data Validation Checklist

No item below blocks **starting** Phase 2. “Required before” identifies the later design/import checkpoint it protects. The excluded test-era customer, address, wishlist, order, inquiry and newsletter classes require no migration inventory or validation; their earlier proposed inspections were removed rather than marked complete.

| Inspection | Source | Why | Required before |
| --- | --- | --- | --- |
| Total products, published/unpublished products and complete pagination | Shopify Admin/export | Size import and detect first-100/truncation gaps | Catalog import dry run |
| Variant counts per product and maximum variants | Shopify | Validate nested query caps and combination model | Catalog schema/import mapping freeze |
| Actual option names/values/order and duplicate/inconsistent spellings | Shopify | Normalize Size/Color/etc. without inventing vocabulary | Catalog mapping freeze |
| Configured `SHOPIFY_PRODUCT_METAFIELD_IDENTIFIERS` | Deployed Vastriqo config | Find runtime-only displayed fields absent from repo defaults | Attribute contract/import mapping freeze |
| Actual metafield/metaobject types, cardinality, references and readable values | Shopify | Preserve displayed specifications, including runtime extras | Attribute contract/import mapping freeze |
| `styling_tips`, `about_this_item`, `style_no`, target-gender, age-group usage | Shopify/BMS | Decide import versus ignore from real usage | Attribute/import mapping freeze |
| SKU coverage, duplicates, blanks and BMS style-number correlation | Shopify/BMS | Validate nullable/unique SKU import rules | SKU constraint/import mapping freeze |
| Currency codes across product/variant/cart/order data | Shopify/BMS | Remove unreliable INR fallback and validate money model | Price import mapping |
| Compare-at, barcode, weight, variant-image and SEO override population | Shopify | Confirm they remain deferred or expose a real operational dependency | Relevant optional provider/feature design only |
| Product media counts/order/alt, source accessibility and duplicate URLs | Shopify | Plan rehosting and verify gallery fidelity | Media import dry run |
| Product publication/status/created/published timestamps | Shopify/BMS | Seed public and merchandising semantics; find stale mirror rows | Catalog import mapping |
| Active Shopify inventory locations/fulfillment services | Shopify | Select default/real location mapping | Inventory schema/import freeze |
| Per-variant tracked quantity, available quantity and oversell policy | Shopify | Preserve current availability and identify backorders | Inventory/sellability contract freeze |
| BMS collection definitions/slugs/status/order | BMS DB/API | Preserve home/Men/Women/Kids and detect duplicates | Collection import mapping |
| BMS collection membership/order/orphans/stale products | BMS | Reconcile native membership | Collection import dry run |
| BMS `image_url` differences from Shopify featured media | BMS + Shopify | Detect actual product-wide card overrides | Media/collection import mapping |
| Actual payment gateways/methods, COD, discounts and representative transactions | Shopify production settings/orders | Preserve real checkout behavior; reject static-copy assumptions | Payment integration contract/provider selection |
| Shipping profiles/zones/services/rates/free-shipping/carrier behavior | Shopify settings/orders | Preserve real delivery choices/charges | Shipping integration contract |
| Tax registrations/settings/rates/order tax lines/invoice output | Shopify + finance/legal evidence | Define compliant replacement | Tax/invoice contract |
| Fraud/risk and transactional notification configuration/events | Shopify/apps/observed orders | Identify hidden required external behavior | Integration/cutover plan |
| Deployed sitemap/logout/generic-product routes absent locally | Deployed BMS/Vastriqo read-only inspection | Confirm no deployed-only behavior is lost | Storefront cutover compatibility plan |
| Analytics destinations/events/consent behavior | Deployed storefront/config | Preserve only approved measurement safely | Analytics reconfiguration, not commerce architecture |

## 21. Compact Final Phase 1 Decision Register

| Decision | Status | Basis | Blocks Phase 2? |
| --- | --- | --- | --- |
| Independent Vastriqo ownership | **ALREADY DECIDED** | Project direction | No |
| Separate `vastriqo-api` / DB / `vastriqo-admin` | **ALREADY DECIDED** | Project direction | No |
| Product and Variant | **ALREADY DECIDED** | Current storefront | No |
| Options/values/selections | **ALREADY DECIDED** | Current detail/purchase | No |
| Product media and displayed specifications | **ALREADY DECIDED** | Current cards/detail | No |
| Complete search/filter/sort | **ALREADY DECIDED** | Current behavior; current cap is deficient | No |
| Native publication/sellability capability | **ALREADY DECIDED** | Current visibility/purchase | No |
| Publication/sellability representation and enforcement | **ENGINEERING DECISION** | Explicit lifecycle dimension and server gating | No |
| Optional SKU concept | **ENGINEERING DECISION** | Operations-friendly without cloning Shopify; validate data | No |
| Compare-at/variant media/brand UI/SEO override | **SAFE TO DEFER** | Not current customer behavior | No |
| Collections/membership/order/related | **ALREADY DECIDED** | Current BMS-backed storefront | No |
| `/collections` all-product alias | **ALREADY DECIDED** | Current storefront | No |
| Product-wide card override representation | **ENGINEERING DECISION** | Actual BMS persistence is product-wide | No |
| Actual live BMS image overrides | **PRODUCTION DATA VALIDATION** | Production rows must prove use/differences | No; before media import mapping |
| Native inventory/reservations/adjustments | **ALREADY DECIDED** | Independent sale requirement | No |
| Location-scoped inventory/no-oversell default | **ENGINEERING DECISION** | Safe durable model | No |
| Actual inventory/location/oversell data | **PRODUCTION DATA VALIDATION** | Production-only | No; before inventory freeze |
| Customer email/password/profile/address | **ALREADY DECIDED** | Project direction/current storefront | No |
| Authenticated India checkout baseline | **ALREADY DECIDED** | Current storefront | No |
| Secure revocable sessions/recovery/verification capability | **ENGINEERING DECISION** | Security required by native identity | No |
| Guest/international/social login/phone verification | **SAFE TO DEFER** | Not current behavior | No |
| Existing customer account continuity/import | **ALREADY DECIDED** | Start clean; current accounts are test users and will not migrate/link/activate | No |
| Wishlist | **ALREADY DECIDED** | Current storefront | No |
| Existing wishlist import | **ALREADY DECIDED** | Current test-era wishlist rows will not migrate | No |
| Anonymous Cart / no merge / separate Buy Now | **ALREADY DECIDED** | Current storefront | No |
| Cart expiry/concurrency/pricing rules | **ENGINEERING DECISION** | Safe server ownership | No |
| Checkout orchestration capability | **ALREADY DECIDED** | Current purchase behavior | No |
| Persisted/idempotent Checkout-to-Order design | **ENGINEERING DECISION** | Retry/payment/inventory safety | No |
| Payment/shipping/tax capability | **ALREADY DECIDED** | Current completed purchase/order display | No |
| Actual checkout/provider configuration | **PRODUCTION DATA VALIDATION** | Hidden in Shopify | No; before integration freeze |
| Payment/shipping/tax provider | **SAFE TO DEFER** | Provider not selected; boundary fixed | No |
| Native Order/history/items/money/status | **ALREADY DECIDED** | Current storefront | No |
| Fulfillment/shipment operations | **ALREADY DECIDED** | Required to complete accepted physical orders | No |
| Self-service tracking/cancel/refund/return/exchange | **SAFE TO DEFER** | Required future capability; current broken/incomplete behavior is not the target specification | No; design progressively in its relevant phase |
| Historical order import/archive | **ALREADY DECIDED** | Current test orders will not migrate and need no dedicated archive | No |
| Existing address import | **ALREADY DECIDED** | Current test-user addresses will not migrate | No |
| Existing inquiry/newsletter import | **ALREADY DECIDED** | Current test-era records will not migrate | No |
| Existing legal/financial migration | **ALREADY DECIDED** | No broad project; only safely reusable existing information may be carried forward where useful | No |
| Detailed tax/legal/invoice/privacy/consent policy | **SAFE TO DEFER** | Decide in the affected implementation phase behind established boundaries | No; required before affected production rollout |
| Independent multiple-admin management | **ALREADY DECIDED** | Project direction | No |
| Required commerce admin modules | **ALREADY DECIDED** | Replacement operations | No |
| Complex RBAC/dashboard/CMS/generic settings | **SAFE TO DEFER** | Not required now | No |
| Deterministic read-only idempotent migration | **ENGINEERING DECISION** | Safety and project direction | No |
| Product/collection/inventory migration | **ALREADY DECIDED** | Required cutover data | No |
| Tokens/BMS JSON/CCA data exclusion | **ALREADY DECIDED** | Not target domain data | No |
| Node/strict TypeScript/npm/modular monolith/relational/OpenAPI | **ENGINEERING DECISION** | Repository conventions and domain needs | No |
| Exact framework/DB/version/deployment tooling | **ENGINEERING DECISION** | Phase 2 ADR, not business requirement | No |
| Microservices/GraphQL/search engine/cache platform | **SAFE TO DEFER** | No measured need | No |

Each row has one final classification. Capabilities, their engineering representation and their production validation are separated into distinct decisions where necessary.

## 22. Phase 1 Business Decisions — Final Resolution

No genuine Phase 1 business decision remains unresolved:

1. **Customer clean start:** do not migrate/link current Shopify or BMS customer accounts, profiles, credentials, passwords or addresses. New customers register directly with Vastriqo and create new native addresses.
2. **Historical customer data:** do not migrate current wishlists or support/inquiry history.
3. **Historical orders:** do not migrate current test orders and do not create a dedicated archive for them. New customer-visible order history begins with native post-cutover orders.
4. **Newsletter/support records:** do not migrate current subscriber or inquiry records. This does not retire future native capabilities.
5. **Legal/financial data:** do not create a broad historical migration project. Already available information may be carried forward only when it is safely and straightforwardly reusable for an included requirement. Detailed GST/tax/invoice/privacy/consent policy is deliberately decided in the relevant implementation phase.
6. **Post-purchase capabilities:** cancellation, refund, return, exchange and customer-visible tracking are required future capabilities. They will be inspected, decided and designed progressively during the relevant order/post-purchase phase; current broken/incomplete behavior is not the target.

Provider selection, framework selection, database engine selection, exact technical state machines, detailed legal policy and detailed post-purchase operating rules are later-phase decisions at their applicable design/implementation checkpoints. They are not unresolved Phase 1 decisions and do not block Phase 2 or Product/Catalog work.

## 23. Phase 1 Completion Gate

### 23.1 Is Phase 1 conceptually complete?

**Yes.** Current behavior, target ownership, domain concepts, cross-domain invariants, public/admin/import responsibilities, dependency exits, migration safety and defer boundaries are sufficiently defined, and all Phase 1 business decisions are resolved.

### 23.2 Which remaining items prevent detailed Phase 2 database/API design?

**None prevent Phase 2 from starting.** No **D** item remains for Phase 1. Phase 2 must keep conditional concepts extensible and must not finalize affected product, inventory or provider contracts before their relevant **C** validation; later legal and post-purchase policy is approved at the applicable implementation checkpoint.

The earliest design checkpoints are:

- production metafield/options/variant inspection before catalog attribute/import mapping freeze;
- inventory location/policy inspection before inventory schema/sellability contract freeze;
- checkout/payment/shipping/tax inspection before external integration contract freeze; and
- explicit exclusion assertions for test-era customer/order/auxiliary records in migration manifests and dry-run reports.

### 23.3 Which items are safely deferred?

- guest/international/social/passwordless commerce;
- real best-seller/personalized/automated merchandising;
- compare-at/promotions, variant-media switching and brand/SEO override UI;
- advanced warehouse operations;
- saved/multiple/cross-device carts;
- detailed self-service cancellation/refund/return/exchange/tracking workflows and advanced partial fulfillment until the relevant order/post-purchase phase; the capabilities themselves remain required future work;
- multiple providers, stored payment instruments, wallet/referrals and complex promotions;
- dashboard, CMS, complex RBAC/SSO and optional admin analytics;
- microservices, GraphQL, cache/search/event/data platforms until measured need.

Each remains outside the initial detailed contract unless a later approved requirement activates it.

### 23.4 Which production inspections are required before migration but do not block architecture?

All inspections in §20. None blocks architecture or Phase 2 start. They gate only the relevant mapping, integration, migration dry run or cutover checkpoint.

### 23.5 Is it safe to begin Phase 2?

**Yes.** Phase 2 can begin detailed design immediately. Repository initialization remains a later phase and is not authorized by this closure.

### 23.6 What exactly should Phase 2 produce?

1. Architecture Decision Records for framework/database/version/query layer, native ID format, exact money/rounding, session/BFF/CORS/CSRF, jobs/outbox, media, deployment, secrets and observability.
2. Logical and physical relational schema with tables/embeddings, keys, indexes, constraints, retention/deletion and migration order.
3. Versioned OpenAPI contracts for public, customer, admin, provider-event, import and reconciliation responsibilities.
4. Customer/admin authentication, authorization and threat model.
5. Product publication, inventory reservation, cart, checkout, payment, order and fulfillment lifecycle/concurrency/failure sequence designs.
6. Provider port/adapter contracts and selection criteria without coupling domain semantics to a vendor.
7. Field-level Shopify/BMS mapping, import manifests, dry-run reports, reconciliation rules, delta/cutover and rollback specifications.
8. `vastriqo-admin` information architecture and workflows against authorized API commands.
9. Test, security, performance, observability, backup/restore, deployment and cutover acceptance plans.
10. A checkpoint schedule for the remaining **C** validations and later-phase legal/post-purchase/provider decisions that must occur before affected contract freezes or production rollout.

### 23.7 What must not be implemented yet?

- Do not create `vastriqo-api` or `vastriqo-admin`.
- Do not create physical database migrations or application code.
- Do not modify `vastriqo`, `bms-api` or `bms-admin`.
- Do not run migration/import scripts or mutate Shopify/BMS.
- Do not select/sign up/configure production providers as a side effect of design.
- Do not remove current dependencies or change the storefront before detailed design, implementation and cutover validation.
- Do not introduce deferred features into the foundation merely because the model can support them.

## 24. Phase 1 Closure Statement

Phase 1 is closed at the conceptual level. The platform is ready to move from requirements/conceptual architecture into Phase 2 detailed data, API, security, lifecycle, migration and admin design.

The final target remains a full independent commerce platform, not an MVP and not a proxy over Shopify/BMS. Phase 2 must preserve that full target while sequencing detailed design and later delivery safely.
