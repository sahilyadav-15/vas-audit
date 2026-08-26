# Phase 1 — Vastriqo Requirements and Feature Disposition

Date: 2026-08-24; closure reconciliation recorded 2026-08-26 (Asia/Kolkata)
Status: **Requirements baseline complete; unresolved items reclassified by the Phase 1 closure register**
Scope: requirements and feature decisions only; no implementation, schema, endpoint, migration-script, provider, or infrastructure design

## 1. Purpose and Status

This document converts the completed current-system evidence into an explicit, reviewable requirements baseline for an independent Vastriqo commerce platform. Phase 0/current-system audit is complete. The source-derived `PHASE-1-FUNCTIONAL-DEPENDENCY-AUDIT.md` is the authority for what the current applications actually do; the project goal/phased-direction document is the authority for the approved future boundary; `report.md` records project progress and decisions.

This is Phase 1 requirements and feature-disposition work. It distinguishes:

1. current implemented behavior;
2. customer and operational behavior that must continue;
3. behavior that must change to achieve independence;
4. Shopify/BMS data or capabilities that are present but not proven requirements; and
5. unresolved business policy.

It is not an implementation plan. It does not choose frameworks, physical tables, endpoints, providers, deployment topology, or detailed state machines. No application source, configuration, dependency, database, Shopify data, or BMS data is changed by this work.

`PRODUCT-CATALOG-DESIGN.md` and `PHASE-1-CONCEPTUAL-ARCHITECTURE.md` continue this baseline into the product vertical slice and full commerce conceptual architecture. `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` is now authoritative for final A–E classifications, remaining business decisions and Phase 1 readiness.

### Evidence and decision notation

- **Audit §n** — `PHASE-1-FUNCTIONAL-DEPENDENCY-AUDIT.md`, based on static source trace.
- **Direction §n** — `Vastriqo Platform — Project Goal & Phased Direction.md`.
- **Progress** — `report.md`.
- **Established** — directly supported by one of those sources.
- **Inferred requirement** — necessary to satisfy an established target or preserved behavior, but not an assertion that it exists today.
- **NEEDS BUSINESS INPUT** — a product or operating-policy choice is missing.
- **NEEDS DATA VALIDATION** — deployed configuration, counts, values, or quality must be inspected.
- **UNDECIDED** — no approved choice is established.

Feature dispositions in this document use exactly: **PRESERVE**, **PRESERVE + IMPROVE**, **REPLACE**, **RETIRE**, **DEFER**, or **UNDECIDED**. A disposition describes the future of a capability, not how its data will be moved.

## 2. Target Platform Definition

The approved requirements-level target is:

```text
vastriqo.in                    admin.vastriqo.in
     |                                |
     v                                v
vastriqo-api <---------------- vastriqo-admin
     |
     v
dedicated Vastriqo DB

CCA remains separate:
bms-admin -> bms-api -> BMS DB
```

The target requires:

- an independent Vastriqo storefront powered by `vastriqo-api`;
- an independent backend named `vastriqo-api`;
- a dedicated Vastriqo database that owns core commerce data;
- an independent administration application named `vastriqo-admin`;
- independent administrator authentication, initially supporting multiple admin accounts and not requiring CCA SSO;
- independent customer registration, email/password authentication, sessions, recovery, and verification;
- CCA/BMS to remain an independently operated business/domain boundary; and
- Shopify, `bms-api`, and the BMS database to leave the final critical path for Vastriqo core commerce.

Shopify may be a controlled temporary dependency during migration. Routing storefront requests through `vastriqo-api` while Shopify remains the commerce source of truth is not Shopify independence. No framework, database engine, cloud topology, provider, or physical module boundary is selected here. (Direction §§1–6, 10–13; Audit §§8–12.)

## 3. Feature Disposition Register

### 3.1 Storefront and content

| Domain | Feature | Current Behavior | Evidence/Source | Future Disposition | Future Requirement | Notes / Open Decision |
| --- | --- | --- | --- | --- | --- | --- |
| Storefront | Global shell/header/footer | Source-controlled shell; header checks BMS-bridged session and Shopify cart count. | Audit §3 | PRESERVE + IMPROVE | Preserve layout and core navigation while reading customer/cart state from Vastriqo. | Content storage remains separate decision. |
| Storefront | Announcement bar | Source-controlled announcement content. | Audit §§3, 5 | PRESERVE | Continue supporting a storefront announcement. | Admin editing is UNDECIDED. |
| Storefront | Home page | Source-controlled editorial content plus BMS `home-page` ordered products and newsletter. | Audit §3 | PRESERVE + IMPROVE | Preserve the experience; use Vastriqo-owned merchandising and independent assets. | Whether editorial content becomes admin-managed is UNDECIDED. |
| Merchandising | Featured products | BMS custom collection controls ordered cards. | Audit §§3, 5 | PRESERVE + IMPROVE | Support a curated, ordered featured-product set using Vastriqo product IDs. | Placement-specific image overrides need input. |
| Storefront | Collections landing | `/collections` currently repeats the all-products grid rather than listing collections. | Audit §§3, 14 | UNDECIDED | Define whether this remains an all-product alias or becomes a true collection landing. | NEEDS BUSINESS INPUT. |
| Storefront | Men | Fixed BMS collection slug with ordered membership. | Audit §3 | PRESERVE + IMPROVE | Preserve the destination with Vastriqo-owned collection/audience data. | Replace text heuristic with explicit classification if approved. |
| Storefront | Women | Fixed BMS collection slug with ordered membership. | Audit §3 | PRESERVE + IMPROVE | Same requirement as Men. | Audience vocabulary needs approval. |
| Storefront | Kids | Fixed BMS collection slug with ordered membership. | Audit §3 | PRESERVE + IMPROVE | Same requirement as Men. | Age/audience vocabulary needs approval. |
| Discovery | Search | Browser substring search over at most the first 100 Shopify products. | Audit §§1, 3 | REPLACE | Search the complete published Vastriqo catalog over approved searchable fields with pagination. | Search technology is a Phase 2 decision. |
| Discovery | Filtering | Browser filters product type, size, color, minimum price, and heuristic audience. | Audit §§3, 7 | PRESERVE + IMPROVE | Retain approved filters over the full catalog and explicit product data. | Approve filter vocabulary; retire unapproved heuristics. |
| Discovery | Sorting | Price and created-time sort locally; Recommended/Best Sellers is a no-op. | Audit §3 | PRESERVE + IMPROVE | Preserve stable default order, price sort, and a deliberate new-arrival rule. | Real best-seller ranking is UNDECIDED; do not label a no-op as such. |
| Storefront | Product cards | Render handle/title/image/minimum price/currency/type/color/size and wishlist/analytics behavior. | Audit §§3, 7 | PRESERVE + IMPROVE | Provide a stable card projection with canonical URL, sellability, price, media, and approved badges/options. | Card quick-add must validate the selected variant. |
| Storefront | Product detail | Shopify provides description, gallery, variants/options, price, specifications, and related products. | Audit §§3, 7 | PRESERVE + IMPROVE | Preserve the customer experience using a Vastriqo-owned product aggregate and safe content rendering. | Exact gallery/specification presentation can evolve. |
| Merchandising | Related products | BMS returns products sharing an active custom category. | Audit §§3, 5 | PRESERVE + IMPROVE | Return related Vastriqo products using an approved collection/rule and deterministic ordering. | Algorithm beyond current collection affinity is not required. |
| Content | Story/About | Source-controlled; `/about-us` redirects to `/our-story`. | Audit §§3, 13 | PRESERVE | Retain the story experience and intentional compatibility redirect. | Admin management is UNDECIDED. |
| Content | Why Vastriqo | Source-controlled editorial page. | Audit §3 | PRESERVE | Retain current content experience. | Content approval/refresh is outside this technical evidence. |
| Content | FAQ | Source-controlled with local search/category filtering. | Audit §3 | PRESERVE | Retain FAQ and its local interaction. | Admin management is UNDECIDED. |
| Content | Size chart | Source-controlled measurement tables. | Audit §3 | PRESERVE | Retain accessible size guidance. | Product/category-specific charts are UNDECIDED. |
| Content | Shipping/returns content | Static policy/marketing copy; it is not evidence of implemented return/shipping workflows. | Audit §§1, 3 | PRESERVE + IMPROVE | Retain content only after aligning every claim with approved operating policy and implemented behavior. | NEEDS BUSINESS INPUT before final copy. |
| Content | Privacy/legal content | Source-controlled privacy/legal material. | Audit §3 | PRESERVE + IMPROVE | Retain required legal surfaces and update them for independent data processing/providers. | Legal review required later. |
| Content | Navigation | Source-controlled header/footer links; BMS menus are unused. | Audit §§3, 5 | PRESERVE | Retain source navigation initially. | Admin-managed navigation is UNDECIDED. |
| Content | BMS storefront pages/menus | BMS modules exist, but the current Vastriqo renderer has no call sites. | Audit §§5, 13 | RETIRE | Do not treat disconnected BMS CMS data as a future content source. | A future Vastriqo CMS remains a separate UNDECIDED requirement. |
| SEO | Product sitemap | Product feed calls a missing BMS route; static entries remain. | Audit §§3, 13 | REPLACE | Generate sitemap entries from all published Vastriqo products with canonical handles and update time. | Include collection/content URLs according to final information architecture. |
| SEO | Metadata/canonical behavior | Product metadata is locally composed; queried Shopify SEO overrides are unused. | Audit §§3, 7 | PRESERVE + IMPROVE | Preserve canonical handle URLs and metadata generation; allow approved overrides if required. | Shopify SEO fields require quality/business review before any import. |
| Measurement | Analytics | GA4 events/page views and Meta Pixel PageView are active; consent behavior is not established. | Audit §§3, 11 | UNDECIDED | Define the measurement and consent requirement before reproducing integrations. | Core commerce must not depend on analytics. |
| Operations | Canonical host | Apex GET/HEAD redirects to `www`. | Audit §3 | PRESERVE | Maintain one canonical production host and redirect policy. | Exact host convention may be reconfirmed at cutover. |
| Storefront | Compatibility/legacy URLs | Several routes redirect or alias current pages; `/details?id=` translates Shopify ID to handle. | Audit §13 | UNDECIDED | Preserve only redirects needed for customer bookmarks, indexed URLs, and cutover compatibility. | NEEDS traffic/SEO data; Shopify-ID lookup must not remain a permanent dependency. |
| Storefront | Broken query fallbacks | Order-line `/products?search=` is ineffective; generic gender and missing sitemap/logout routes have no verified backend. | Audit §13 | RETIRE | Do not carry nonfunctional route assumptions forward; approved user journeys must use supported native routes. | Verify deployed-only behavior before removal during implementation. |
| Storefront | Overlapping/unused integration paths | Alternate cookie-based Shopify auth, unused `/api/cart`, missing custom-auth helpers, and generic product API have no active verified role. | Audit §13 | RETIRE | Exclude these paths from the future architecture; retain only evidence-backed customer/cart/catalog behavior through native capabilities. | Verify deployed-only callers before implementation-time removal. |

### 3.2 Catalog and inventory

| Domain | Feature | Current Behavior | Evidence/Source | Future Disposition | Future Requirement | Notes / Open Decision |
| --- | --- | --- | --- | --- | --- | --- |
| Catalog | Products | Shopify is product source of truth; BMS holds a partial mirror. | Audit §§2–5 | REPLACE | Vastriqo must own products, publication, projections, and admin writes. | Do not copy Shopify schema or BMS JSON as the target. |
| Catalog | Product handles/slugs | Handles form canonical product URLs. | Audit §§3, 7 | PRESERVE + IMPROVE | Unique stable handles, collision rules, and redirects for changed handles are required. | Redirect retention policy is UNDECIDED. |
| Catalog | Variants | Shopify variants determine selections, price, and cart merchandise identity. | Audit §§3, 7 | REPLACE | Vastriqo-owned variants must carry option selections, price, sellability, and stable identity. | SKU requirement remains separate. |
| Catalog | Options | Shopify option names/values and variant selected options build selectors/filters. | Audit §§3, 7 | PRESERVE + IMPROVE | Model deliberate selectable dimensions and valid variant combinations. | Avoid Shopify-shaped duplication. |
| Catalog | Option values | Size/color values are customer-facing and filterable. | Audit §§3, 7 | PRESERVE + IMPROVE | Normalize display/order values while retaining variant-specific selections. | Data normalization needs validation. |
| Catalog | Images/media | Featured image and ordered detail gallery are used; variant image is not used. | Audit §§3, 7 | PRESERVE + IMPROVE | Own ordered media, alt text, roles, and durable storage references. | Variant association is conditional. |
| Catalog | Descriptions | Plain description supports search; HTML/plain description is rendered on detail. | Audit §§3, 7 | PRESERVE + IMPROVE | Support searchable plain text and safe rich description content. | Sanitization behavior belongs to design. |
| Catalog | Product type/classification | Used in cards, search, filtering, analytics, and heuristics. | Audit §§3, 7 | PRESERVE + IMPROVE | Define an explicit Vastriqo classification vocabulary. | NEEDS BUSINESS INPUT on category versus type semantics. |
| Catalog | Tags | Used for search and audience heuristic. | Audit §7 | UNDECIDED | Retain only tags with approved discovery or operational meaning. | NEEDS DATA VALIDATION and business taxonomy review. |
| Catalog | Vendor/brand | Used in search/related data but not visibly on detail. | Audit §7 | UNDECIDED | Decide whether brand/vendor is customer-facing, searchable, or operational. | Do not infer requirement from Shopify query alone. |
| Catalog | SKU | Queried but not used by current storefront. | Audit §7 | UNDECIDED | Decide if SKU is required for admin, stock, shipping, accounting, or reconciliation. | Material operational decision. |
| Catalog | Pricing | Shopify variant/minimum prices and cart/order money are authoritative. | Audit §§3, 7 | REPLACE | Vastriqo must own variant prices/currency, cart revalidation, and order price snapshots. | Monetary conventions are Phase 2 design. |
| Catalog | Compare-at pricing | Queried but not displayed. | Audit §7 | UNDECIDED | Add only if strike-through/promotional pricing is approved. | Not a current experience requirement. |
| Catalog | Publication status | Shopify storefront visibility is implicit; BMS cards may be stale. | Audit §§3, 4, 7 | REPLACE | Explicit product/variant publication and visibility rules are required. | Lifecycle names/transitions belong to Phase 2. |
| Catalog | Availability/sellability | Shopify ultimately rejects invalid cart/checkout; UI helper is not authoritative. | Audit §§3, 7 | REPLACE | Compute sellability from publication, price, inventory, and approved oversell rules. | Must be server-authoritative. |
| Catalog | Product attributes | Shopify metafields/metaobjects are displayed generically or used for card values. | Audit §7 | REPLACE | Own approved typed attributes and readable values without Shopify coupling. | Attribute vocabulary requires validation. |
| Catalog | Metafield-derived specifications | Fabric/care/material/etc. render on detail. | Audit §7 | REPLACE | Preserve approved specification meaning in a Vastriqo attribute/content model. | Keys are source mappings, not target schema. |
| Catalog | Metaobject-derived information | Display resolves labels/values/handles from Shopify references. | Audit §7 | REPLACE | Store the business value directly or through deliberate reusable reference data. | Shopify metaobject IDs/images are not automatically required. |
| Merchandising | Collection membership | BMS custom collections drive home/gender/related cards. | Audit §§3, 5 | PRESERVE + IMPROVE | Vastriqo-owned active collections must reference Vastriqo products. | Shopify collection relationships are not the target. |
| Merchandising | Collection ordering | BMS `sort_order` controls curated order. | Audit §§3, 7 | PRESERVE + IMPROVE | Preserve explicit ordered membership and a stable default order. | Related-product tie-breaking must be defined. |
| Merchandising | Image overrides | BMS can override a mirrored product card image. | Audit §7 | UNDECIDED | Decide whether override is product-wide, placement-specific, or retired. | NEEDS BUSINESS INPUT and live-data validation. |
| Inventory | Inventory quantity | Shopify is current source; BMS mirror total is not an inventory system. | Audit §§3–5, 11 | REPLACE | Vastriqo must own authoritative sellable stock at approved granularity. | NEEDS DATA VALIDATION at migration. |
| Inventory | Availability | Current UI is permissive and Shopify enforces downstream. | Audit §§3, 7 | REPLACE | Server-authoritative availability is required on detail, cart, checkout, and order creation. | Exact rule depends on overselling/reservations. |
| Inventory | Reservations | No Vastriqo/BMS reservation model exists; Shopify hides enforcement. | Audit §§8–10 | REPLACE | Inferred requirement: reserve/release stock sufficiently to prevent inconsistent order acceptance. | Reservation timing/expiry NEEDS BUSINESS INPUT. |
| Inventory | Stock adjustments | No independent Vastriqo operation exists. | Direction §§1–5; Audit §§9–10 | REPLACE | Admin/API must support controlled stock change with reason and auditability. | Adjustment workflow and authorization belong to later design. |
| Inventory | Inventory reconciliation | BMS snapshots are incomplete; no independent process exists. | Audit §§6, 11–12 | REPLACE | Support migration reconciliation and external/provider reconciliation where relevant. | Frequency/source depends on cutover phase. |
| Inventory | Overselling behavior | Shopify behavior is not established; UI does not define policy. | Audit §§3, 14 | UNDECIDED | Business must approve whether any product/variant may sell below available stock. | Affects availability and reservations. |
| Inventory | Inventory locations | No current multi-location requirement is established. | Audit §§10, 14 | UNDECIDED | Add location-level stock only if operations require it. | NEEDS BUSINESS INPUT before physical model design. |

### 3.3 Customer, cart, checkout, and orders

| Domain | Feature | Current Behavior | Evidence/Source | Future Disposition | Future Requirement | Notes / Open Decision |
| --- | --- | --- | --- | --- | --- | --- |
| Identity | Registration | `/signup` runs Shopify Customer OAuth; no distinct registration flow. | Audit §§3, 13 | REPLACE | Vastriqo-owned email/password registration is required. | Verification policy is still to be approved. |
| Identity | Login | Shopify OAuth PKCE plus BMS bridge creates local JWT. | Audit §§3–5 | REPLACE | Authenticate customers independently with email/password initially. | Google login is deferred. |
| Identity | Email/password credentials | Not owned by Vastriqo today. | Established business direction in task; Direction §2 | REPLACE | Secure customer credential lifecycle is required without designing its implementation here. | Password policy belongs to Phase 2/security review. |
| Identity | Customer session | JWT and user data are in localStorage; current-user check reaches BMS/Shopify. | Audit §§3, 13 | REPLACE | Vastriqo must own secure session issuance, validation, rotation/expiry, and revocation. | Storage mechanism is not selected. |
| Identity | Logout | Missing BMS endpoint is ignored; only browser state is cleared. | Audit §§3, 13 | REPLACE | Logout must terminate or revoke the applicable Vastriqo session. | All-device logout is UNDECIDED. |
| Customer | Profile | BMS local profile edits can drift from Shopify. | Audit §§2, 3, 5 | PRESERVE + IMPROVE | Vastriqo must own one consistent customer profile. | Select migrated fields after validation. |
| Customer | Address book | BMS proxies live Shopify CRUD; country is effectively India in writes. | Audit §§3–5 | REPLACE | Vastriqo must own customer addresses and validation. | Supported countries/fields NEED BUSINESS INPUT. |
| Customer | Default address | Shopify owns the relation. | Audit §§3–4 | REPLACE | Customers must be able to select a default address if address book is retained. | Checkout override behavior belongs to design. |
| Identity | Account recovery | Delegated to Shopify/not established in app source. | Direction §§1–2; task direction | REPLACE | Independent email/password accounts require a recovery flow. | Channel, expiry, and support process are later design. |
| Identity | Verification | Shopify customer state is queried; independent policy is not defined. | Audit §§4, 14; task direction | REPLACE | Independent identity must support an approved verification lifecycle. | Whether verification is mandatory before purchase NEEDS BUSINESS INPUT. |
| Customer | Wishlist | BMS stores Shopify product/customer references and snapshots. | Audit §§3, 5 | PRESERVE + IMPROVE | Preserve saved products using Vastriqo identities and pending-login behavior. | Retention/migration depends on customer approval. |
| Support | Inquiry/contact | Public/optional-auth form writes BMS inquiries. | Audit §§3, 5 | PRESERVE + IMPROVE | Preserve inquiry submission with correct customer linkage and support handling. | File attachments are UNDECIDED. |
| Support | Inquiry history | Hidden from account nav and has an identity-type mismatch. | Audit §§3, 13 | UNDECIDED | Decide whether customers need inquiry history; if retained, redesign ownership. | NEEDS BUSINESS INPUT. |
| Engagement | Newsletter | BMS records normalized email/source; duplicates succeed. | Audit §§3, 5 | PRESERVE + IMPROVE | Preserve subscription capture with explicit status, consent, source, and unsubscribe policy. | Existing consent quality NEEDS DATA VALIDATION. |
| Support | Inquiry attachments | BMS accepts an optional S3 reference, but current contact/history UI neither sends nor renders one. | Audit §5 | UNDECIDED | Add attachment upload/view/retention only if support operations approve the use case. | Do not migrate S3 references automatically. |
| Engagement | Wallet/referrals | No current storefront call site; login creates a referral code only as a BMS side effect. | Audit §§5, 13 | DEFER | Keep outside the current independent-platform requirement set pending an explicit product decision. | Existing records may need archive/retention review; do not build or import automatically. |
| Cart | Cart creation | Direct Shopify cart mutation. | Audit §§3–4 | REPLACE | Vastriqo must create an independently owned cart. | Guest/authenticated ownership policy is open. |
| Cart | Persistence | Device `localStorage.cartId` points to Shopify cart. | Audit §3 | REPLACE | Preserve cart continuity using an approved anonymous/customer ownership model. | Client storage/session design is not selected. |
| Cart | Add/update/remove items | Direct Shopify mutations. | Audit §§3–4 | REPLACE | Support line creation, quantity change, removal, validation, and deterministic errors. | Must use Vastriqo variant identity. |
| Cart | Totals | Shopify supplies line and total values. | Audit §§3–4 | REPLACE | Calculate/revalidate subtotal and approved adjustments server-side. | Shipping/tax/discount inclusion depends on checkout policy. |
| Cart | Cart expiry | Invalid Shopify carts are cleared; no business expiry policy is established. | Audit §3 | UNDECIDED | Define anonymous and customer cart lifetime and cleanup behavior. | NEEDS BUSINESS INPUT. |
| Cart | Customer/cart relationship | Cart is device-local until Shopify buyer identity is attached at checkout. | Audit §3 | REPLACE | Define ownership and authorization for anonymous and signed-in carts. | Merge policy remains open. |
| Cart | Guest cart/checkout | Cart can be anonymous, but checkout requires login. | Audit §§3, 14 | UNDECIDED | Decide whether guests may complete checkout. | Current restriction may be Shopify-token driven. |
| Cart | Login/cart merge | No merge is implemented. | Audit §3 | UNDECIDED | Decide whether anonymous items merge, replace, or remain separate at login. | Affects sessions and inventory. |
| Checkout | Checkout | BMS validates profile and Shopify hosts checkout/payment. | Audit §§3–5 | REPLACE | Vastriqo must orchestrate contact/address, stock/price validation, shipping, tax, payment, and order creation. | Providers and detailed policies remain open. |
| Checkout | Buy Now | Creates a separate Shopify cart and enters checkout. | Audit §3 | PRESERVE + IMPROVE | Retain a direct-purchase path if it follows the same validations and order guarantees as checkout. | Confirm whether it should bypass/retain existing cart. |
| Checkout | Shipping | Shopify calculates it; static content makes claims. | Audit §§1, 3, 14 | REPLACE | Independent checkout requires an approved shipping-option/rate boundary and order snapshot. | Provider, service levels, rates, free-shipping rule are open. |
| Checkout | Tax/GST | Shopify owns calculation; exact behavior is not established. | Audit §§3, 14 | REPLACE | Independent checkout/order must calculate and retain legally required tax/GST results. | Rule engine/provider and invoice requirements need input. |
| Checkout | Discounts/promotions | No implemented storefront behavior is established. | Audit §§3, 14 | UNDECIDED | Define discount scope only if required by business. | Do not infer from Shopify capability. |
| Checkout | Payment | Shopify hosted checkout owns methods/results. | Audit §§1, 3–4 | REPLACE | Vastriqo must own the payment integration boundary, state, verification, and reconciliation. | Provider/methods/COD are UNDECIDED. |
| Checkout | Order creation | Shopify creates retail orders. | Audit §§1–4 | REPLACE | Create exactly one durable order from an accepted checkout with immutable purchase snapshots. | Detailed state machine belongs to Phase 2. |
| Checkout | Idempotency | Hidden inside Shopify; no independent behavior exists. | Audit §§9–10 | REPLACE | Inferred requirement: retries/callbacks must not create duplicate orders or charges. | Keys/transactions are design details. |
| Orders | Order history | Latest 10 Shopify orders are shown live. | Audit §3 | PRESERVE + IMPROVE | Customers must see paginated history for new Vastriqo orders. | Historical Shopify order treatment is UNDECIDED. |
| Orders | Order detail | Current cards show order, money, line, payment gateway, and shipping summary. | Audit §3 | PRESERVE + IMPROVE | Provide authorized order detail with complete approved status and snapshots. | Detail depth depends on fulfillment/returns policies. |
| Orders | Order status | Shopify financial/fulfillment strings are displayed. | Audit §3 | REPLACE | Define Vastriqo-owned order, payment, and fulfillment status semantics. | Exact states/transitions belong to Phase 2 after policy approval. |
| Orders | Order items | Shopify lines provide title/variant/quantity/price/image. | Audit §3 | REPLACE | Persist immutable item, variant/options, quantity, price, tax, and media/title snapshots needed for history. | SKU snapshot depends on SKU decision. |
| Orders | Pricing snapshots | Shopify returns totals/subtotal/shipping/tax and line money. | Audit §3 | REPLACE | Retain auditable order-level and line-level money snapshots. | Discount allocation is conditional on discount scope. |
| Orders | Tax | Displayed from Shopify order. | Audit §3 | REPLACE | Retain tax result/breakdown required by approved policy and law. | Exact invoice/breakdown requirement needs input. |
| Orders | Shipping | Shopify order contains shipping total/address. | Audit §3 | REPLACE | Retain selected shipping method, charge, destination snapshot, and status. | Provider data model comes later. |
| Orders | Payment state | Financial status and first gateway are displayed. | Audit §3 | REPLACE | Own payment state and provider-reference history sufficient for support/reconciliation. | Do not equate payment state with order state. |
| Orders | Fulfillment | Shopify query asks for fulfillment data, but UI ignores it. | Audit §§3, 13 | REPLACE | Independent order operations require approved fulfillment state at least sufficient to ship accepted orders. | Partial fulfillment is UNDECIDED. |
| Orders | Shipment | No customer shipment view is implemented. | Audit §§3, 14 | UNDECIDED | Define shipment records/customer visibility with the fulfillment policy. | Capability likely operational, but scope is not established. |
| Orders | Tracking | Button only scrolls the order card; no tracking data is shown. | Audit §§1, 3 | UNDECIDED | Decide whether real tracking number/URL/events are required and align content. | Do not classify static claims as implemented. |
| Orders | Cancellation | Redirects to contact page; performs no cancellation. | Audit §§1, 3 | UNDECIDED | Approve eligibility, states, customer/admin actions, and payment consequences. | Current query parameter is not consumed. |
| Orders | Refund | No workflow is implemented. | Audit §§1, 3, 14 | UNDECIDED | Approve refund policy, authority, amounts, and reconciliation. | Must align with payment provider later. |
| Orders | Return | Static policy only. | Audit §§1, 3, 14 | UNDECIDED | Approve eligibility/window/process and inventory/payment effects. | Marketing copy is not a system requirement by itself. |
| Orders | Exchange | Static policy only. | Audit §§1, 3, 14 | UNDECIDED | Approve exchange behavior and whether it is return plus new order. | NEEDS BUSINESS INPUT. |
| Orders | Notifications | Content may make claims; no commerce notification workflow is established. | Audit §§1, 14 | UNDECIDED | Decide event/channel requirements for account, checkout, payment, order, and shipment. | Email/SMS providers are not selected. |

### 3.4 Administration and integrations

| Domain | Feature | Current Behavior | Evidence/Source | Future Disposition | Future Requirement | Notes / Open Decision |
| --- | --- | --- | --- | --- | --- | --- |
| Admin | Admin authentication | Shopify Admin/BMS admin are not the future Vastriqo identity system. | Direction §§5, 8; Progress | REPLACE | Separate Vastriqo admin authentication with multiple accounts is required. | Architecture must permit later roles/permissions; no CCA SSO now. |
| Admin | Dashboard | Future direction lists it only as an example; current need is not audited. | Direction §5 | UNDECIDED | Define operational decisions/metrics before specifying a dashboard. | Must not block core domain design. |
| Admin | Products | Product ownership/admin writes currently sit in Shopify. | Direction §§1–5; Audit §§4–5 | REPLACE | Admin users must manage Vastriqo products and publication. | Exact screens/workflow come later. |
| Admin | Variants/options | Managed upstream in Shopify. | Audit §§3–4 | REPLACE | Manage variants, option/value combinations, price, and sellability inputs. | Bulk workflows are UNDECIDED. |
| Admin | Product attributes | Shopify metafields/metaobjects are upstream. | Audit §7 | REPLACE | Manage the approved Vastriqo attribute vocabulary and product values. | Vocabulary approval precedes detailed contract. |
| Admin | Media | Shopify/CCA S3/local assets are mixed. | Audit §§3, 7 | REPLACE | Manage product media and independent storage references with ordering/alt text. | Editorial asset management scope is separate. |
| Admin | Inventory | Shopify is authority; no native operations exist. | Direction §§1–5; Audit §§8–10 | REPLACE | Authorized users must view and adjust authoritative stock with traceability. | Locations/overselling/reservation policies are open. |
| Admin | Collections | BMS custom collections currently manage relevant merchandising. | Audit §§3, 5 | PRESERVE + IMPROVE | Manage Vastriqo collections, active state, membership, and order. | Import BMS meaning, not schema. |
| Admin | Merchandising | BMS ordering/image override plus source home content are split. | Audit §§3, 5, 7 | PRESERVE + IMPROVE | Support approved featured placement, collection ordering, and related-product control. | Override and CMS scope are open. |
| Admin | Orders | Retail order operations reside in Shopify; current storefront is read-only. | Direction §5; Audit §§3–4 | REPLACE | Authorized admins must find and inspect orders and perform approved operations. | Operations depend on cancellation/refund/fulfillment policy. |
| Admin | Customers | BMS/Shopify split customer data. | Direction §5; Audit §§2–5 | REPLACE | Authorized admins need customer lookup and approved support/account operations. | PII access and mutation permissions need design. |
| Admin | Customer addresses | Shopify owns address data; operational admin use is not established. | Audit §§3–4 | UNDECIDED | Decide which support roles may view/edit addresses and under what audit controls. | Customer self-service remains required. |
| Admin | Wishlist support | Wishlist exists; admin handling need is not established. | Audit §§3, 5 | UNDECIDED | Add admin visibility only for an approved support/merchandising use. | Privacy impact. |
| Admin | Inquiries/support | BMS persists inquiries; customer history path is broken. | Audit §§3, 5, 13 | PRESERVE + IMPROVE | If inquiry is retained, provide authorized status/response handling and correct ownership. | Exact support workflow is UNDECIDED. |
| Admin | Newsletter | BMS captures subscribers; admin/consent operations were not audited. | Audit §§3, 5 | UNDECIDED | Define export, unsubscribe, suppression, and consent operations before admin scope. | NEEDS BUSINESS INPUT/legal input. |
| Admin | Content/navigation | Current renderer is source-controlled; BMS CMS is unused. | Audit §§3, 5 | UNDECIDED | Add CMS/menu management only if business users need runtime editing. | Do not adopt BMS CMS by default. |
| Admin | Settings | Generic future example only. | Direction §5 | UNDECIDED | Define specific settings and ownership before creating a module. | Secrets/provider config must not become arbitrary editable settings. |
| Admin | Payment/order operations | Shopify currently owns payment boundary. | Direction §5; Audit §§3–4 | REPLACE | Provide only approved capture/refund/reconciliation/order actions with authorization and auditability. | Depends on provider and policy. |
| Admin | Fulfillment/shipping | Shopify owns current capability; storefront does not expose it fully. | Direction §5; Audit §§3–4 | REPLACE | Provide operational fulfillment/shipping actions required to complete independent orders. | Scope/state model NEEDS BUSINESS INPUT. |
| Admin | Auditability | No complete Vastriqo admin audit requirement exists; independent operations require traceability. | Inferred from Direction §§5, 10 and Audit §§9–10 | REPLACE | Inferred requirement: replace reliance on external-platform histories with attributable records for security-sensitive and commerce-changing admin actions. | Exact event coverage/retention belongs to Phase 2. |
| Integration | Shopify | Core catalog, identity, cart, checkout, payment, addresses, and orders depend on it. | Audit §§2–4, 12 | RETIRE | May remain temporary only behind explicit exit conditions; final core commerce state is NOT REQUIRED. | A proxy to Shopify is not independence. |
| Integration | BMS | Merchandising and auxiliary/customer bridge behaviors depend on it. | Audit §§2, 5, 12 | RETIRE | Replace retained Vastriqo concepts; final core commerce state is NOT REQUIRED. | Any permanent CCA link needs a separate approved contract. |
| Integration | Payment provider | Shopify currently hides provider/method details. | Audit §§3, 14 | UNDECIDED | Keep a replaceable payment boundary; select provider/methods later. | Required capability, undecided provider. |
| Integration | Shipping provider | Shopify currently hides rates/fulfillment integration. | Audit §§3, 14 | UNDECIDED | Keep an explicit shipping boundary if provider integration is required. | Manual fulfillment versus provider NEEDS BUSINESS INPUT. |
| Integration | Tax/GST | Shopify current behavior is not established. | Audit §§3, 14 | UNDECIDED | Define compliant tax/GST responsibility before detailed checkout design. | Service versus internal calculation is open. |
| Integration | Email | No transactional commerce email integration is established. | Audit §§3, 14 | UNDECIDED | Select only after notification and recovery requirements are approved. | Recovery likely requires a delivery channel. |
| Integration | SMS | Phone is collected; no implemented commerce SMS workflow is established. | Audit §§3, 14 | UNDECIDED | Add only for approved verification/notification use cases. | Do not infer from phone field. |
| Integration | Storage/CDN | Local assets, Shopify media, CCA S3 default, and Google Fonts are mixed. | Audit §3 | REPLACE | Vastriqo product/media operation must not depend on CCA asset ownership; use durable independent asset delivery. | Provider/topology is open. |
| Integration | Analytics | GA4/Meta are active. | Audit §3 | UNDECIDED | Reapprove destinations, events, identifiers, retention, and consent. | External and non-critical. |
| Integration | Fraud/abuse controls | Shopify/provider behavior is unknown. | Audit §§9, 14 | UNDECIDED | Define risk controls proportionate to approved auth/checkout/payment flows. | Specific service is not required now. |
| Integration | Permanent CCA/BMS link | No permanent cross-boundary need is established. | Direction §6; Audit §14 | UNDECIDED | Core commerce must remain isolated; add only an explicit, failure-isolated contract for an approved use case. | Shared tables, tokens, or secrets are not acceptable. |

## 4. Catalog Requirements

The future catalog represents Vastriqo's business concepts, not Shopify resources. The independent platform must support:

- **Identity and URLs:** an internal product identity and unique stable handle; source Shopify/BMS IDs may exist only as external migration/reconciliation mappings.
- **Customer content:** title, searchable plain description, safely rendered rich description where retained, and canonical metadata inputs.
- **Variants:** independent variant identity, valid option/value selections, current price/currency, publication/sellability, and relationships to the product.
- **Options and values:** ordered selectable dimensions such as size/color only where they define a product/variant choice; normalized values and valid combinations.
- **Pricing:** authoritative variant price, derived card/minimum price, currency, server revalidation, and immutable order snapshots. Compare-at/promotional price is not required until approved.
- **Availability:** explicit publication and computed sellability using authoritative inventory and approved reservation/oversell rules; UI presence of a variant ID cannot mean purchasable.
- **Media:** a featured/card image, ordered product gallery, alt text, durable independent object reference/URL, and optional variant/placement association only if approved.
- **Classification and discovery:** deliberate product classification, approved audience data, approved tags/brand fields, searchable/filterable projections, stable sorts, pagination, and ordered collections.
- **Specifications:** approved typed product attributes with readable values. Shopify namespace/key/type/metaobject shape must not dictate the physical target model.
- **Collections and merchandising:** active collections, ordered membership, featured placements, and a related-product rule using Vastriqo identities.
- **Publication and SEO:** explicit visibility lifecycle; handle-based canonical URLs; generated metadata and optional approved overrides; published product sitemap data.

### 4.1 Audited fields and target requirement classification

| Audited field/data | Classification | Requirement and rationale | Validation/decision |
| --- | --- | --- | --- |
| Product identity | Required | Needed for every product relation; must be Vastriqo-owned. | Preserve Shopify ID only as external mapping. |
| Handle | Required | Current canonical product route and SEO behavior depend on it. | Define collision/redirect policy. |
| Title | Required | Display, search, analytics, snapshots, and metadata use it. | Validate data quality. |
| Plain/rich description | Required | Search and product detail use them. | Define sanitization in Phase 2. |
| Variant/options/selections | Required | Customer selection, price, sellability, and cart identity depend on them. | Validate option normalization. |
| Variant/current price/currency | Required | Detail, cart, checkout, and order depend on it. | Define money conventions later. |
| Featured and ordered media/alt | Required | Cards, details, metadata, cart/wishlist/order snapshots use it. | Select source media; do not import blindly. |
| Product type/classification | Required | Current discovery/card behavior requires the concept. | Approve taxonomy rather than copy strings blindly. |
| Tags | Conditional | Only approved discovery/operational tags should survive. | NEEDS DATA VALIDATION and taxonomy input. |
| Vendor/brand | Undecided | Current search/related payload uses it, but customer-visible requirement is not established. | NEEDS BUSINESS INPUT. |
| Created/published time | Conditional | A deliberate new-arrival/publish sort requires it. | Do not assume Shopify creation time is correct. |
| Publication status | Required | Independent visibility and merchandising consistency require it. | Status model is Phase 2. |
| SKU | Undecided | Queried but no storefront consumer is established; may be operationally necessary. | NEEDS BUSINESS INPUT from inventory/fulfillment/finance. |
| Compare-at/max price | Not currently required | Current UI does not consume it. | Add only with approved promotional pricing. |
| Variant image | Not currently required | Current UI does not switch media by variant. | Conditional if experience changes. |
| Barcode/weight/shipping/tax flags | Undecided | Presence in Shopify query is not proof of requirement. | Shipping/tax/operations input required. |
| Shopify SEO title/description | Not currently required | Current product metadata is locally composed. | Import only after content/SEO review. |
| Shopify categories/collection relations | Not currently required | Current merchandising uses BMS custom collections. | Recreate approved Vastriqo collections instead. |
| BMS collection membership/order | Required | Drives home, gender pages, and related products. | Transform to Vastriqo IDs; validate live records. |
| BMS image override | Undecided | Current cards can consume it; intended scope is unclear. | NEEDS DATA VALIDATION and business input. |

### 4.2 Shopify custom data/metafield disposition

“Required” below means the business datum/current experience must be representable. It does not require retaining the Shopify key, namespace, type, or metaobject structure.

| Shopify key/custom data | Classification | Current evidence and independent-platform requirement |
| --- | --- | --- |
| `color-pattern` | Required | BMS collection cards use its value/reference handle as Color data. Preserve approved color values for cards/filtering/merchandising; variant options remain authoritative for selectable variants. |
| `size` | Required | BMS collection cards use it as Size data and size is a current filter/selection concept. Preserve normalized size values; variant selections remain authoritative. |
| `fabric` | Required | Displayed as a product-detail specification when present. |
| `age-group` | Not currently required | Queried but hidden and not established as a direct filter. It may become part of an approved audience taxonomy later. |
| `target-gender` | Conditional | Current direct usage is not established, but explicit audience data could replace fragile text heuristics and support Men/Women/Kids. Approve the audience model first. |
| `care-instructions` | Required | Displayed as a current product-detail specification. |
| `clothing-features` | Required | Displayed as a current product-detail specification. |
| `neckline` | Required | Displayed as a current product-detail specification. |
| `top-length-type` | Required | Displayed as a current product-detail specification. |
| `material` | Required | Displayed as a current product-detail specification. |
| `material-composition` | Required | Displayed as a current product-detail specification. |
| `sleeve-length-type` | Required | Displayed as a current product-detail specification. |
| `pattern` | Required | Displayed as a current product-detail specification. |
| `fit` | Required | Displayed as a current product-detail specification. |
| `styling_tips` | Undecided | No committed default identifier or named consumer; production configuration is unknown. **NEEDS DATA VALIDATION.** |
| `about_this_item` | Undecided | No committed default identifier or named consumer; production configuration is unknown. **NEEDS DATA VALIDATION.** |
| Other runtime-configured identifiers | Undecided | Non-hidden configured values may render generically, but production identifiers/data are unknown. Inventory production configuration before approval. |
| Metaobject label/name/title/value | Conditional | Preserve readable business values only for approved required attributes. |
| Metaobject ID or referenced image | Not currently required | Queried but not consumed by current specification rendering. Do not copy automatically. |

## 5. Customer and Identity Requirements

### 5.1 Customers

- A customer account is Vastriqo-owned and has one consistent profile rather than split Shopify/BMS authorities.
- Initial customer registration and login support **email + password**.
- Credentials, sessions, logout/revocation, password recovery, and the approved verification lifecycle are owned by Vastriqo.
- Implementation choices such as cookie/token format, hashing library, identity tables, and endpoint contracts are not decided here.
- Profile supports the approved name, email, and phone fields. Uniqueness, mutability, and re-verification rules require Phase 2 design after business approval.
- Address book supports the current address concepts and a default address. Supported countries, required fields, validation, and whether admins may edit addresses need business/security input.
- Wishlist remains a customer/product relationship using Vastriqo identities.
- Inquiry ownership must distinguish public contact from authenticated customer history; it must not reuse incompatible BMS user/customer IDs.
- If customer migration is approved, the platform must support explicit source identity mappings, duplicate resolution, account activation/linking, consent/verification treatment, and a safe retirement of Shopify tokens. Shopify credentials/tokens are not imported as Vastriqo credentials.
- Future Google login is **DEFERRED**. The identity model should be capable of adding another verified identity later without requiring Google login now.

### 5.2 Administrators

- Vastriqo administrators use a separate Vastriqo admin authentication system.
- Multiple administrator accounts are required.
- The architecture must not prevent future roles and permissions, although the initial role matrix is not defined here.
- CCA/BMS SSO is not required at this stage. Cross-admin navigation may be a redirect between independently authenticated applications.
- Sharing JWT secrets, session tables, credentials, or authentication tables with BMS is not an acceptable shortcut to cross-navigation.

## 6. Commerce Requirements

| Area | CURRENTLY IMPLEMENTED | REQUIRED FINAL BEHAVIOR | UNDECIDED BUSINESS POLICY |
| --- | --- | --- | --- |
| Catalog | Shopify rich products plus partial BMS cards; first-100 browser discovery. | Vastriqo-owned published products/variants/options/media/attributes/prices, complete paginated discovery, and stable merchandising. | Final taxonomy, brand/SKU/compare-at/extra field scope. |
| Inventory | Shopify ultimately enforces; current UI availability is unreliable. | Authoritative stock, availability, reservation/release, checkout validation, adjustments, and reconciliation sufficient to accept orders independently. | Locations, overselling, reservation timing, adjustment workflow. |
| Cart | Shopify cart with local device ID; add/update/remove; no merge policy. | Vastriqo-owned lifecycle, authorization, lines, price/stock validation, totals, expiry policy, and consistent errors. | Guest checkout, expiry durations, login merge behavior. |
| Checkout | Signed-in customer only; BMS validates name/phone; Shopify hosts checkout. | Independent orchestration of customer/contact, address, shipping, tax, discounts if approved, payment, and idempotent order creation. | Guest eligibility, required contact fields, shipping/tax/payment methods, discounts/COD. |
| Buy Now | Creates a separate Shopify cart then hosted checkout. | If retained, uses the same validation/payment/order guarantees as standard checkout. | Existing-cart interaction and whether the shortcut remains desired. |
| Orders | Shopify owns creation/history/status; UI shows latest 10. | Vastriqo-owned new orders/items with immutable price/address/product snapshots and authorized customer/admin history. | Historical Shopify order import/archive horizon and detailed status policy. |
| Payments | Entirely behind Shopify hosted checkout. | Explicit provider boundary, attempt/transaction state, verified callbacks, idempotency, reconciliation, and authorized refund actions if approved. | Provider, methods, capture timing, COD, partial/full refunds. |
| Fulfillment | Shopify data is queried but largely not rendered. | Enough owned fulfillment/shipping state and admin operations to complete independent orders. | Partial fulfillment, provider/manual process, shipment/tracking visibility. |
| Cancellations/returns/refunds | Contact redirect or static copy only; no commerce mutation. | Implement only the workflows approved by business and align storefront policy claims with them. | Eligibility, windows, actors, state transitions, fees, reverse logistics, exchange model. |
| Customer history | Shopify live orders/addresses; BMS profile/wishlist/inquiry/newsletter. | Vastriqo-owned new account data and retained features; historical data per approved migration/retention rules. | Customer/address/order/wishlist/inquiry migration scope and retention. |

Across checkout/payment/order creation, independent behavior must be retry-safe and prevent duplicate charges/orders. Exact transaction design is Phase 2 work. Provider integrations must be replaceable boundaries; this document does not select them.

## 7. Content and Merchandising Requirements

| Content/capability | Current owner | Requirement-level decision | Future management decision |
| --- | --- | --- | --- |
| Home editorial content/banners | Storefront source | PRESERVE; decouple assets from CCA ownership. | Source-controlled initially; CMS is UNDECIDED. |
| Header/footer/navigation | Storefront source | PRESERVE. | Admin management is UNDECIDED. |
| FAQ | Storefront source | PRESERVE current content/search behavior. | Admin management is UNDECIDED. |
| Story/About and Why Vastriqo | Storefront source | PRESERVE. | Admin management is UNDECIDED. |
| Size chart | Storefront source | PRESERVE. | Product-specific/admin-managed charts are UNDECIDED. |
| Shipping/returns | Storefront source | PRESERVE + IMPROVE only after copy matches approved workflows. | Business/legal ownership required. |
| Privacy/legal | Storefront source | PRESERVE + IMPROVE for new data processors/integrations. | Legal approval required; CMS not assumed. |
| Announcement/banner scheduling | Static source | Announcement concept PRESERVE; scheduling is UNDECIDED. | Add to admin only if operational need is approved. |
| Featured products | BMS custom collection | PRESERVE + IMPROVE as Vastriqo-owned ordered placement. | Must be manageable by Vastriqo admin. |
| Men/Women/Kids collections | BMS custom collections | PRESERVE + IMPROVE with deliberate taxonomy/membership. | Must be manageable by Vastriqo admin. |
| Collection ordering | BMS `sort_order` | PRESERVE + IMPROVE. | Must be manageable by Vastriqo admin. |
| Related products | Shared active BMS category | PRESERVE + IMPROVE using approved native rule. | Manual overrides/advanced recommendations are UNDECIDED. |
| Product media/card overrides | Shopify/BMS mix | Native media required; special overrides UNDECIDED. | Decide product-wide versus placement-level behavior. |
| BMS storefront pages/menus | BMS, unused by Vastriqo | RETIRE as an assumed source. | Do not import/adopt without a new CMS decision. |

Source-controlled content is already Vastriqo-owned and need not move into the database merely to achieve commerce independence. A CMS is justified only by approved editorial roles, publishing workflow, preview/scheduling, localization, or runtime change requirements.

## 8. Current vs Future Authentication Paths

| Authentication path | Status | Purpose today/future | Final disposition | Migration implication |
| --- | --- | --- | --- | --- |
| Shopify Customer Account OAuth/PKCE | Active current path | Customer login/signup and upstream identity. | REPLACE | May remain temporarily; establish Vastriqo accounts/linking before retirement. Shopify tokens do not become permanent credentials. |
| BMS customer bridge (`customers`, `customer_tokens`, JWT) | Active current bridge | Exchanges Shopify code, mirrors profile, stores token, issues 24-hour app JWT. | REPLACE | Selected profiles/external mappings may be transformed; token/session model is recreated. |
| Legacy native BMS front-user auth | Unused/overlapping | Uncalled login/signup/logout helpers and separate BMS `users` identity. | RETIRE | Do not use it as the new customer architecture or import its route assumptions. |
| Future Vastriqo customer auth | Required target | Email/password registration/login, verification/recovery, session and logout. | REPLACE | Define account migration/linking only if customer migration is approved; Google login is deferred. |
| Current BMS admin auth | Active for BMS/CCA | Secures `bms-admin` and current BMS operations. | PRESERVE | Remains inside the independent CCA/BMS boundary; not reused as Vastriqo admin auth. |
| Future Vastriqo admin auth | Required target | Secures multiple Vastriqo administrator accounts. | REPLACE | New independent identity/session boundary; allow later roles/permissions. No CCA SSO requirement now. |

## 9. Shopify Dependency Exit Register

Putting any dependency below behind `vastriqo-api` without changing its authority does **not** satisfy independence.

| Dependency | Current Use | Temporary Allowed? | Future Replacement | Exit Condition | Final State |
| --- | --- | --- | --- | --- | --- |
| Product catalog/list | First 100 products for all-products/search/collections alias. | Yes, controlled | Native published catalog query. | Catalog data reconciled; complete storefront list/search reads Vastriqo. | NOT REQUIRED |
| Product detail | Rich product, descriptions, media, specifications by handle/ID. | Yes | Native product-detail aggregate. | Detail pages and metadata use validated Vastriqo data. | NOT REQUIRED |
| Variants/options/prices | Selection, merchandise identity, selected price. | Yes | Native variant/option/price ownership. | Variant mappings, selection, price, and cart behavior pass reconciliation. | NOT REQUIRED |
| Inventory/availability | Shopify is effective sellability authority. | Yes until single authority cutover | Native inventory, availability, reservations, reconciliation. | Stock rules and concurrency/cutover tests pass with one Vastriqo authority. | NOT REQUIRED |
| Cart | Creation/read/line mutations/totals. | Yes | Native cart lifecycle/service. | Native cart persistence, mutations, pricing, stock validation, and expiry are production-ready. | NOT REQUIRED |
| Checkout | Buyer attachment and hosted checkout URL. | Yes | Native checkout orchestration. | Address, shipping, tax, payment handoff, and order creation are validated end-to-end. | NOT REQUIRED |
| Customer identity | OAuth, customer identity token. | Yes | Vastriqo email/password identity/session. | Registration/login/recovery/verification, linking/migration, and session cutover are validated. | NOT REQUIRED |
| Customer profile | Admin customer read/update and BMS sync. | Yes | Vastriqo customer profile. | Approved customer data is imported/activated or intentionally excluded; profile is native. | NOT REQUIRED |
| Addresses | Live Shopify Admin CRUD/default. | Yes | Native address book. | Address self-service and approved migration/recreation are validated. | NOT REQUIRED |
| Orders/history | Shopify creates and returns live retail orders. | Yes | Native orders/items/history; optional archive/import for old orders. | New orders are native and approved historical access policy is available. | NOT REQUIRED |
| Fulfillment/tracking | Shopify query supplies data, though UI ignores it. | Yes while Shopify orders remain | Native approved fulfillment/shipment/tracking capability. | Approved workflows work for native orders and old-order access is resolved. | NOT REQUIRED |
| Payment | Hosted checkout owns method/result. | Yes until payment cutover | Independent provider boundary and payment state. | Verified callbacks, failure handling, idempotency, refunds if approved, and reconciliation pass. | NOT REQUIRED |
| Shopify product/variant/customer/address/order GIDs | Identity threaded through storefront, BMS, cart, wishlist, and customer bridge. | Yes as external mapping | Vastriqo IDs plus bounded migration mapping. | Runtime/domain relations use Vastriqo IDs; mapping retained only for approved reconciliation/archive. | NOT REQUIRED for operation |
| Shopify cart IDs | Device-local cart pointer. | Yes only while Shopify carts are active | Vastriqo cart identity. | Shopify carts expire/bridge per cutover policy and no new ones are created. | NOT REQUIRED |
| Shopify access/refresh/customer tokens | OAuth/customer/cart buyer attachment and admin integrations. | Yes while explicit temporary calls remain | Vastriqo credentials/sessions and provider credentials as applicable. | All related dependencies exit; retention window passes; secrets/tokens are revoked/deleted. | NOT REQUIRED |
| Shopify product sync | Populates BMS mirror. | Yes for migration/reconciliation | Repeatable import into native domain, then native admin writes. | Final import/delta reconciliation completes and Shopify no longer changes native catalog. | NOT REQUIRED |
| Shopify publication semantics | Implicit visibility/cart enforcement. | Yes until catalog cutover | Explicit Vastriqo publication/sellability. | Native rules are authoritative across catalog, merchandising, cart, checkout. | NOT REQUIRED |

## 10. BMS Dependency Exit Register

| BMS dependency | Current Use | Temporary Allowed? | Future Replacement / Disposition | Exit Condition | Final State |
| --- | --- | --- | --- | --- | --- |
| Custom collections | Home/Men/Women/Kids membership/order. | Yes | Preserve concept in native collections. | Definitions/membership/order map to native products and admin management is validated. | NOT REQUIRED |
| Product mirror | Partial Shopify-shaped cards/JSON. | Yes for comparison/import | Replace with native catalog; raw JSON is not target data. | Native catalog projections cover all approved fields and reconcile to source. | NOT REQUIRED |
| Related products | Shared active collection lookup. | Yes | Preserve/improve rule natively. | Native related results meet approved behavior. | NOT REQUIRED |
| Customer bridge | Shopify exchange, local customer/token/JWT. | Yes | Replace with native customer identity. | Account migration/linking and session cutover complete. | NOT REQUIRED |
| Customer profile | Primary profile reads/writes BMS `customers`. | Yes | Preserve profile in native customer domain. | Approved profiles are imported/activated or intentionally excluded. | NOT REQUIRED |
| Checkout validation/details | Checks name/phone and exposes Shopify token; dual writes. | Yes | Replace with native customer/checkout validation. | Native validation is approved and no Shopify buyer token is needed. | NOT REQUIRED |
| Addresses | Pure Shopify Admin proxy. | Yes | Replace with native address book. | Native CRUD/default and migration policy complete. | NOT REQUIRED |
| Orders | Pure Shopify Admin query for retail orders. | Yes | Replace with native order history/archive policy. | New history is native and old-order access decision is implemented. | NOT REQUIRED |
| Wishlist | BMS records with Shopify/customer bridge IDs. | Yes | Preserve/improve native wishlist. | Records are remapped/reconciled or excluded per approved scope. | NOT REQUIRED |
| Inquiry submission | Contact form writes BMS records. | Yes | Preserve/improve submission natively. | Native public/authenticated submission and approved support handling are operational. | NOT REQUIRED |
| Inquiry history | Hidden route with incompatible BMS user/customer ownership. | Only if business retains it | UNDECIDED; redesign ownership if approved. | Decision is approved and either native history is validated or the path is retired. | NOT REQUIRED |
| Newsletter | Subscriber capture. | Yes | Preserve/improve if consent/retention approved. | Native capture/unsubscribe policy and approved import complete. | NOT REQUIRED |
| Wallet/referrals feature | No storefront consumer. | No final requirement established | DEFER. | Confirm later scope or retire after archive/retention review. | NOT REQUIRED |
| Automatic referral-code login side effect | BMS customer bridge creates a code without a storefront referral flow. | Not needed | RETIRE. | Verify no approved/deployed consumer and remove the coupling during implementation. | NOT REQUIRED |
| Storefront pages/menus | BMS modules exist but Vastriqo does not use them. | Not needed | RETIRE as assumed source; future CMS is separate decision. | Verify no deployed-only consumer; preserve source content independently. | NOT REQUIRED |
| Native front auth/profile | Uncalled/overlapping path. | Not needed | RETIRE. | Compatibility review completed; no active callers. | NOT REQUIRED |
| Missing/legacy logout/auth routes | Clients target unaudited missing routes. | Not needed | RETIRE and replace with native session behavior. | New auth is validated and legacy paths have no callers. | NOT REQUIRED |
| Missing product sitemap route | Vastriqo expects unaudited `/shopify-products/sitemap`. | Not needed | Replace with native published-product sitemap. | Sitemap uses native catalog and is verified. | NOT REQUIRED |
| Inquiry S3 attachment | BMS accepts a file but current UI never sends/displays it. | Only if deployed use is proven | UNDECIDED. | Decide attachment scope; migrate selected files or retire capability. | NOT REQUIRED unless separately approved |
| CCA/BMS database | Mixed business data including bridge records. | Only as read-only migration source where approved | Never the Vastriqo commerce database. | Required data transformed/reconciled; runtime reads/writes cease. | NOT REQUIRED |

## 11. Migration Scope — Requirements Only

This classifies scope; it does not authorize data access, scripts, imports, or writes. Product-related migration remains the initial priority.

| Data/capability | Migration classification | Requirement and boundary |
| --- | --- | --- |
| Product data | Required migration | Transform approved product identity/content/classification/publication from authoritative source; do not copy Shopify shape. |
| Variants | Required migration | Transform valid variant identities, selections, price, and approved operational fields. |
| Options/values | Required migration | Normalize approved selectable dimensions and variant relationships. |
| Media | Required migration | Copy/rehost selected product media and alt/order metadata into independent ownership; validate URLs/files. |
| Product attributes/specifications | Required migration | Transform only required/conditional approved values; inventory production-configured keys first. |
| Collections | Required migration | Transform active BMS custom collection meaning to Vastriqo identity. |
| Collection ordering | Required migration | Preserve validated ordered membership/merchandising rank. |
| Inventory | Required migration + recreate/reconcile | Obtain an authoritative cutover snapshot/delta from Shopify/operations; BMS total is insufficient; initialize native stock/rules. |
| Customers | Later migration — UNDECIDED | Do not expand initial product scope. Decide population, consent, duplicate/linking, activation, and verification first. |
| Addresses | Later migration — UNDECIDED | Shopify-live data; follows the customer migration and country/field policy. |
| Wishlists | Later migration | Transform only if customers and products are mapped and retention is approved. |
| Inquiries | Later migration — UNDECIDED | Correct identity ownership; decide history/status/attachment retention. |
| Newsletter subscribers | Likely later migration | Import only records with acceptable consent/retention evidence; create unsubscribe/suppression policy. |
| Historical orders | UNDECIDED | Choose full import, read-only archive, limited horizon, or no customer-visible migration. Do not map CCA BMS orders. |
| Active carts | Do not migrate by default | Prefer expiry or a temporary bridge; migrate only if cutover policy explicitly requires continuity. |
| Shopify tokens/customer access tokens | Do not migrate | Integration credentials are not Vastriqo credentials; revoke/delete after dependency exit and retention window. |
| BMS customer tokens/JWT/session records | Do not migrate | Recreate secure native credentials/sessions; selected profile data is separate. |
| BMS product mirror JSON | Archive/retain temporarily for reconciliation; do not migrate as target data | Useful evidence/source comparison only. |
| Shopify/BMS external IDs | Recreate as bounded migration mappings | Retain only where import reconciliation, redirects, or historical archive needs them; never primary identity. |
| Source-controlled content/navigation | Recreate/retain | Already Vastriqo-owned; continue in source unless CMS is approved. |
| Source/local/CCA-linked assets | Recreate/retain selectively | Keep valid local assets; rehost required CCA/Shopify assets under independent ownership. |
| BMS storefront pages/menus | Do not migrate | No current consumer. |
| Wallet/referrals | UNDECIDED; archive/retain if legally/operationally needed | No storefront requirement is established; do not silently add to scope. |
| Analytics configuration/events | Recreate if approved | Review identifiers, events, consent, and destinations rather than copy automatically. |

Every approved migration later requires source inventory, mapping, validation, reconciliation, repeatability, cutover ownership, and rollback criteria. Those artifacts are Phase 2/later work.

## 12. Requirements Explicitly Out of Scope for Now

The following do not block completion of this requirements artifact:

- selecting a payment gateway, payment methods, or COD provider;
- selecting a shipping/logistics provider or final service levels;
- selecting a tax/GST service or detailed calculation implementation;
- implementing future Google login;
- implementing CCA/Vastriqo SSO;
- implementing wallet/referrals without an approved product decision;
- building a CMS before content-management roles/workflows are approved;
- choosing AWS/cloud topology, object-storage vendor, networking, CI/CD, or observability products;
- choosing the `vastriqo-api`/`vastriqo-admin` frameworks and package tooling;
- physical database tables, columns, indexes, engine, migrations, or seed data;
- API endpoints, payloads, versioning, validation envelopes, or error codes;
- detailed authentication/session/authorization design;
- detailed order/payment/fulfillment state machines before business policy is approved;
- migration scripts, live data writes, cutover commands, or data deletion;
- application creation or storefront/admin/API code changes.

The independent capabilities for payment, shipping, tax, identity, inventory, orders, fulfillment, and data migration are not out of scope for the overall platform. Only provider selection and implementation design are deferred.

## 13. Open Business Decisions

Closure note: this broad list is retained as the requirements-stage discovery record. It has been narrowed and reclassified in `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`. Items established by the current storefront are no longer treated as approval questions; engineering choices and production-data checks are separated from genuine business decisions.

These decisions materially affect Phase 2 domain, API, data, or admin design and require approval:

1. **Catalog taxonomy and audience:** What are the approved product type/category, Men/Women/Kids/audience, color, size, tag, and brand/vendor vocabularies? This determines classification, filters, collections, attributes, and migration normalization.
2. **Operational product fields:** Are SKU, barcode, weight, shipping/tax flags, compare-at price, variant media, and SEO overrides required? These affect admin, fulfillment, pricing, and import scope.
3. **Production custom data:** Which runtime-configured Shopify metafields/metaobjects actually contain valuable data, including `styling_tips` and `about_this_item`? This needs a non-mutating production inventory before the attribute model is finalized.
4. **Collections page and merchandising overrides:** Should `/collections` list collections or all products, and are card-image overrides product-wide or placement-specific? This affects information architecture and collection/media relationships.
5. **Inventory policy:** Is inventory single- or multi-location; when is stock reserved/released; are backorders/overselling allowed; who adjusts stock and why? These decisions define sellability and concurrency rules.
6. **Guest commerce:** May guests complete checkout, and how should anonymous carts expire or merge at login? This affects identity, cart ownership, checkout, and migration continuity.
7. **Customer verification/profile policy:** Must email be verified before purchase; is phone required/verified; which fields can customers change; which countries/addresses are supported? This affects credential, customer, checkout, and address contracts.
8. **Customer data migration:** Will existing customers be migrated, how will they activate/link accounts, and how will duplicates/consent be handled? This determines identity mappings and rollout.
9. **Checkout scope:** Which shipping rules, tax/GST outcomes, discounts/promotions, payment methods, COD, and fraud checks are business requirements? Shopify currently hides them, so they must be stated before detailed checkout design.
10. **Order lifecycle:** What statuses and transitions are required, including payment failure, cancellation, fulfillment, shipment, and delivery? This defines the order/admin contract.
11. **Cancellation/refund/return/exchange:** Which workflows are genuinely offered, with what eligibility windows, actors, charges, inventory effects, and refund behavior? Current content is not implementation evidence.
12. **Tracking and notifications:** Is customer-visible shipment tracking required, and which account/order/payment/shipment events require email or SMS? This affects fulfillment records and external-service boundaries.
13. **Historical data:** Should historical Shopify orders be imported fully, imported read-only for a limited period, or retained in a separate archive? Should addresses, wishlists, inquiries, and newsletter records migrate? This affects migration volume, customer history, privacy, and support.
14. **Admin operating scope:** Which first-release admin capabilities and roles are required beyond products, variants, media, inventory, collections, orders, and customers? Decide support, newsletter, address access, dashboards, settings, fulfillment, payment, and audit-history depth.
15. **Content management:** Which content must business users edit without deployment, and are preview, scheduling, approval, or localization required? This decides whether any CMS domain belongs in Vastriqo.
16. **Permanent CCA/BMS integration:** Is there any approved cross-business use case after separation? If yes, its data owner, contract, privacy, failure isolation, and availability expectation must be explicit.
17. **Wallet/referrals:** Is this a future Vastriqo product feature or should the disconnected BMS behavior be retired? This determines whether any records/domain concepts survive.

Provider names, framework choices, and physical design are intentionally not business decisions in this register; they follow approved behavior.

## 14. Phase 1 Acceptance Criteria

This list was the conservative requirements-stage acceptance proposal. The final closure review found that several entries can be satisfied as Phase 2 checkpoints rather than prerequisites to start it. `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` §23 is the authoritative completion gate. The following remain useful acceptance/cutover checks:

- every meaningful audited feature has one approved disposition;
- the target independence boundary and CCA/BMS separation are approved without exceptions hidden in implementation plans;
- required catalog identity, fields, taxonomy, variant/option, media, pricing, publication, and specification vocabulary are approved;
- production custom field and catalog/collection/inventory data have been inventoried sufficiently to validate the requirements;
- customer email/password direction, verification/profile policy, guest policy, session expectations, and migration/account-link approach are approved or explicitly deferred with safe boundaries;
- inventory authority, granularity, reservation, overselling, adjustment, and reconciliation expectations are known;
- cart, checkout, shipping, tax, payment, order, fulfillment, cancellation/refund/return, tracking, and notification responsibilities are defined at business-behavior level;
- first-release Vastriqo admin operating responsibilities are approved;
- each data class is assigned required/likely/later/recreate/archive/do-not-migrate/undecided treatment;
- historical order and customer-data access/retention decisions are explicit;
- every temporary Shopify/BMS dependency has an owner, replacement, exit condition, and final NOT REQUIRED state;
- any permanent CCA/BMS integration is either explicitly approved with a narrow purpose or explicitly absent;
- unknowns are recorded as decisions/data validation rather than filled with generic e-commerce assumptions; and
- no implementation choice, Shopify schema detail, or BMS schema detail is presented as a product requirement.

### Current readiness assessment

This requirements document originally treated several policy, provider and production-data items as a single Phase 2 gate. The later source-led closure separates them. **Phase 1 is conceptually complete and Phase 2 detailed design may begin.** Conditional contract freezes and migration plans still wait for the specific validation or business decision identified in `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` §§20–23.

## 15. Recommended Phase 2 Inputs

Once the Phase 1 acceptance criteria are met, Phase 2 should consume:

- the approved feature disposition register;
- approved catalog/attribute/taxonomy and inventory rules;
- approved customer/admin identity and migration requirements;
- approved cart/checkout/order/payment/fulfillment business behavior;
- approved content/admin scope and external integration boundaries;
- the Shopify and BMS exit registers;
- the migration-scope classifications and validated source inventories; and
- the resolved business-decision log plus explicitly deferred items.

The conceptual domain model and platform boundaries exist in `PHASE-1-CONCEPTUAL-ARCHITECTURE.md` and are closed by `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`. Phase 2 should now produce the physical database schema and constraints, API contracts, detailed authentication/authorization design, lifecycle/concurrency rules, field-level migration specifications, and admin workflow contracts. None of those physical or executable designs is created by this requirements document.
