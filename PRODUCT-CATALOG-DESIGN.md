# Vastriqo Product/Catalog Analysis and Conceptual Design

Date: 2026-08-25 (Asia/Kolkata)  
Status: **Analysis complete; unresolved items finally classified in `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`**
Scope: Phase A repository/source inspection and Phase B product dependency analysis only

## 1. Purpose, Scope, and Evidence

This document defines the evidence-backed Product/Catalog vertical slice for the independent Vastriqo platform. It does not implement or approve a database schema, endpoint set, API framework, admin framework, inventory policy, migration script, or storefront change.

The current storefront is the functional reference. Shopify and BMS are sources and temporary dependencies, not the target domain model.

### Authoritative context read

- `README.md` — current-system audit baseline.
- `PHASE-1-FUNCTIONAL-DEPENDENCY-AUDIT.md` — most authoritative source-derived functional dependency audit.
- `PHASE-1-REQUIREMENTS-AND-FEATURE-DISPOSITION.md` — approved direction and unresolved requirements.
- `Vastriqo Platform — Project Goal & Phased Direction.md` — system boundary and migration principles.
- `report.md` — current phase/repository status.
- `vastriqo/AI_SEO_RESEARCH_BRIEF.md` — current route/metadata inventory; used only where it agrees with source and the later functional audit.

No `PROJECT-AUDIT.md` or `Project-Structure.txt` exists in this workspace. `README.md` is the current-system audit equivalent. Root-level Markdown is the existing migration-document convention, so this document is placed at the repository root rather than creating a new documentation hierarchy.

### Source inspected

The trace covered:

- Shopify Storefront product/list/detail/cart queries in `vastriqo/lib/shopify/index.ts`;
- product list/card/search/filter/sort behavior in `vastriqo/components/collections/sections/collections-grid-section/products-client.tsx`;
- product detail, variants, media, specifications, related products, and purchase handoff in `vastriqo/components/collections/details-page/index.tsx`;
- all product/list/category/detail routes and home featured-product use;
- BMS collection-card normalization and metafield-derived Color/Size mapping;
- sitemap, SEO, analytics, cart, and wishlist product consumers;
- product-related environment/configuration;
- BMS Shopify Admin synchronization, mirror schema, collection membership/order, related-product selection, routes, controllers, and migrations;
- current BMS admin Shopify product and collection screens/types/hooks; and
- package manifests and API/admin conventions relevant to later implementation.

This remains a static local-source analysis. No live Shopify store, deployed environment, BMS database, or production metafield configuration was queried. All such facts are marked **NEEDS DATA VALIDATION**.

## 2. Executive Findings

1. Vastriqo currently has two product projections, neither of which is independent:
   - a rich Shopify Storefront projection for `/products`, `/collections`, `/search`, and product detail; and
   - a thin BMS mirror projection for home, Men/Women/Kids, and related-product cards.
2. Direct catalog list/search/filter/sort operates over `products(first: 100)` with no pagination. Search and filtering are browser-side, so results are incomplete when the live catalog exceeds 100 products.
3. Product detail uses Shopify by handle (or legacy Shopify ID), derives option selectors from variant selections, displays selected variant price, combines three media sources, and renders approved/non-hidden metafields as specifications.
4. BMS `shopify_products` is a one-way, partial Shopify snapshot plus opaque `meta` JSON. It is not an independent product, variant, pricing, publication, or inventory model.
5. BMS custom collection membership/order is real current merchandising behavior. The four configured slugs and shared-category related-product logic require either a minimal native collection relationship in this slice or an explicitly temporary BMS dependency.
6. BMS product sync paginates Shopify products but truncates each product to 20 images, 100 variants, 30 metafields, and 10 metafield references. It has no verified deletion/unpublish reconciliation. It cannot be treated as a complete migration authority.
7. Current product availability is not reliable in Vastriqo. `quantityAvailable` is typed but not queried; both card/detail helpers treat any variant with an ID as purchasable when quantity is absent. Shopify cart/checkout is the effective enforcement point.
8. Current product SEO overrides are queried/mirrored but not used by the canonical slug page. The page generates title/description from the product title and uses the featured image.
9. The BMS admin Product view is a synced, read-only Shopify inspection screen. Its “Edit details” control opens the same read-only modal. Native create/edit/variant/media/price/publication management does not exist.
10. The BMS collection “display image” editor writes `shopify_products.image_url`, not a collection-membership field. The override is therefore product-wide across BMS collection cards despite being edited inside a collection screen. The target meaning must be approved.
11. No `vastriqo-api` or `vastriqo-admin` repository exists. Existing project direction requires separate applications, but their exact framework, database engine, repository initialization, API envelope, and test tooling remain undecided.

## 3. Current Product Data Flow

### 3.1 Direct Shopify catalog flow

```text
Shopify Storefront GraphQL
  products(first: 100)
      |
      +--> /products
      +--> /collections (currently another all-products grid)
      `--> /search?q=...
              |
              v
      browser search/filter/sort
              |
              v
         ProductCard
              |
              v
      /{product-handle}
              |
              v
 Shopify productByHandle
 descriptions + variants + prices + images/media + selected metafields
```

The legacy `/details?id=...` path queries Shopify by product ID and redirects to the handle route when a handle is returned.

### 3.2 BMS merchandising flow

```text
Shopify Admin GraphQL
      |
      v
BMS incremental product sync
      |
      v
shopify_products columns + Shopify-shaped meta JSON
      |
      +--> shopify_categories + ordered shopify_category_products
      |       +--> home-page
      |       +--> mens-collection
      |       +--> womens-collection
      |       `--> kids-collection
      |
      `--> products sharing active category membership -> related products
              |
              v
thin card mapper in Vastriqo
              |
              v
handle link -> Shopify product detail
```

BMS cards contain mainly product ID, handle, title, one image, minimum price/currency, and Color/Size values extracted from mirrored metafields. They do not prove that the product is currently published, sellable, or complete.

### 3.3 Downstream product identity flow

```text
Shopify product GID
  +--> React keys and product analytics
  +--> BMS mirror and collection membership
  +--> related-product lookup
  `--> BMS wishlist snapshots

Shopify variant GID
  +--> selected detail merchandise
  `--> Shopify cart line creation
```

The independent replacement must use Vastriqo product and variant identities in all runtime relations. Shopify/BMS IDs may survive only as explicit external-source mappings for import, reconciliation, redirects, or approved archives.

## 4. Current Product Dependencies

| Dependency | Current responsibility | Product-slice consequence |
| --- | --- | --- |
| Shopify Storefront product list | Rich first-100 product projection. | Replace with complete, paginated native catalog list/search/filter/sort. |
| Shopify Storefront product by handle/ID | Product detail, variants, selected price, media, descriptions, specifications. | Replace with native detail aggregate keyed by Vastriqo handle/ID. |
| Shopify Storefront availability | Supplies flags; cart ultimately enforces sellability. | Native API must be authoritative before Shopify product runtime exits. |
| Shopify variant GID | Merchandise identity for cart. | Map to stable Vastriqo variant ID; cart integration later consumes that ID. |
| Shopify Admin product sync | Supplies BMS mirror and migration-shaped data. | May be an isolated import source only; must not sit under the runtime API. |
| BMS `shopify_products` | Thin card columns and broad JSON snapshot. | Use for comparison/reconciliation, not as target schema or primary source where Shopify is available. |
| BMS custom collections | Home/gender card membership and order. | Minimal relationship is required to switch those current product surfaces; implementation scope needs approval. |
| BMS related products | Shared active-category selection. | Preserve through native product/collection identities or leave as an explicit temporary dependency. |
| BMS image override | One global mirror image edited from collection UI. | Meaning is ambiguous; do not reproduce until approved. |
| `SHOPIFY_PRODUCT_METAFIELD_IDENTIFIERS` | Adds runtime-selected product custom fields. | Production value must be inventoried before the attribute model/import is finalized. |
| Shopify media URLs / CCA asset default | Product/media delivery is externally owned. | Product slice needs durable independent media references eventually; provider/topology is not selected. |
| Wishlist snapshots | Stores Shopify product ID/handle/title/image. | Wishlist is out of implementation scope, but product IDs/handles must be stable for later remapping. |
| Cart product projection | Uses variant ID, variant title/price, product title/handle/featured image. | Product/variant model must be sufficient for later cart snapshots; cart itself is out of scope. |
| Analytics | Uses product/variant ID, title, product type, value, and currency. | Native identifiers and stable product classification/price fields must replace Shopify values. |
| Product sitemap | Expects handle/title/updated time from a missing BMS route. | Native catalog must expose a published-product sitemap projection. |

## 5. Actual Product Fields Consumed by Vastriqo

The “future representation” column describes a domain need, not a physical table/column choice.

| Field/concept | Current source and transformation | Actual consumer/behavior | Required future representation |
| --- | --- | --- | --- |
| Product ID | Shopify GID; BMS sometimes supplies numeric/external ID as string. | React keys, analytics, wishlist, related lookup, legacy URL fallback. | Required Vastriqo product identity; external mapping for Shopify/BMS IDs. |
| Handle | Shopify handle; BMS `meta.handle`, `meta.slug`, or `url`. | Canonical `/{handle}` route, card links, wishlist/order links, sitemap. | Required unique stable handle plus an approved change/redirect policy. |
| Title | Shopify/BMS. | Cards, detail H1, search, links, metadata templates, analytics, snapshots. | Required product title. |
| Plain description | Shopify list/detail. | Browser search and fallback detail copy. | Required searchable plain text or a reliable derivation from approved rich content. |
| Description HTML | Shopify detail/BMS meta. | Preferred product-detail “Fabric Details” content via raw HTML rendering. | Required rich product content if current display is preserved; sanitization is required at a later design level. |
| Product type/classification | Shopify `productType`; BMS meta. | Card badge, category filter, search, audience heuristic, analytics. | Required classification concept; vocabulary/structure is unresolved. |
| Vendor/brand | Shopify/BMS meta. | Browser search and related-card shape; not visibly rendered on detail. | **UNDECIDED** business concept. |
| Tags | Shopify list. | Browser search and audience heuristic. | Conditional approved discovery data; exact tags need validation/taxonomy review. |
| Created/publish timestamp | Shopify `createdAt`; other timestamps queried/mirrored. | “New Arrivals” sorts by `createdAt`; sitemap expects update time. | A deliberate merchandising/publish time if new-arrival sort remains, plus update time for sitemap. Do not blindly reuse Shopify creation time. |
| Publication/visibility | Shopify Storefront visibility and Admin `status`/`publishedAt`; BMS collection status only gates collection. | Implicitly controls direct catalog presence; stale BMS cards can bypass product-status checks. | Required explicit product visibility/publication decision and native query enforcement. Exact states are unresolved. |
| Featured image | Shopify/BMS image URL. | Card, product metadata image, gallery, cart/wishlist/order snapshots. | Required featured media role or deterministic first-media rule. |
| Image URL/object reference | Shopify URLs; BMS may preserve override. | All product imagery. | Required independent durable media reference; storage provider is unresolved. |
| Image alt text | Shopify; BMS cards substitute product title. | Card/detail accessibility and metadata image alt. | Required nullable alt text with title fallback behavior. |
| Media ordering | Shopify image order; BMS image positions; detail deduplicates by URL. | Detail thumbnail/gallery order. | Required stable ordered product media. |
| Product options | Shopify list; BMS derives only Color/Size from metafields. | List filter values. | Required ordered option definitions and values for actual selectable dimensions. |
| Variant selected options | Shopify detail/list. | Detail option names/values, exact selected-variant matching, card Color fallback. | Required normalized variant-to-option-value relationships. |
| Variant ID | Shopify GID. | Selected merchandise passed to cart and analytics. | Required stable Vastriqo variant identity plus external mapping. |
| Variant title | Shopify cart projection; queried on product. | Cart displays it unless “Default Title”; detail does not use it. | Conditional display/snapshot value; may be derived from selected options. |
| Variant price | Shopify detail/cart. | Selected detail price, future cart input, analytics. | Required current variant amount and currency. |
| Minimum product price | Shopify price range; BMS computes lowest variant price. | Cards, price filters/sorts, detail fallback. | Required catalog projection derived from eligible variant prices. |
| Currency | Shopify money; BMS extracts/falls back to INR. | Formatting, filters/sorts, analytics, cart. | Required ISO currency value; single- versus multi-currency scope is unresolved. |
| Variant availability | Shopify flags; quantity is typed but not fetched by storefront. | Variant fallback and enable/disable intent, but current helper effectively accepts any ID. | Required server-authoritative sellability result based on approved publication/inventory rules. |
| Inventory quantity | BMS sync stores per-variant/total quantity in JSON; storefront does not consume it. | Shopify cart/checkout indirectly enforces stock. | Required input to independent sellability, but stock granularity and physical model are unresolved. |
| Required specifications | Selected Shopify metafields/metaobjects. | Detail renders non-hidden, non-empty fields with humanized labels/readable values. | Required structured Vastriqo product attributes for approved keys. |
| Color values | Variant options for direct catalog; `color-pattern` value/reference handle for BMS cards. | Filters, detail selectors, card swatches. | Required normalized color values; selectable variant values remain authoritative for purchase. |
| Size values | Variant options for direct catalog; `size` value/reference handle for BMS cards. | Filters, detail selectors, card size summary. | Required normalized size values; selectable variant values remain authoritative for purchase. |
| Collection membership | BMS ordered custom category relationships. | Home, Men/Women/Kids, related products. | Required if all current product surfaces cut over in this slice; otherwise explicit temporary BMS dependency. |
| Collection order | BMS `sort_order`. | Default product order for curated surfaces and related-product priority. | Preserve stable ordered membership when the relationship moves. |
| Product-wide card image override | BMS `shopify_products.image_url`, retained during resync. | Used by every BMS card projection. | **UNDECIDED**; current UI implies placement editing but persistence is global. |
| SEO title/description inputs | Shopify fields queried/mirrored but not used on canonical slug page. | Slug page composes metadata from product title and a fixed template; featured image supplies OG image. | Generated SEO behavior is required; imported overrides are conditional/undecided. |
| Updated timestamp | Shopify/BMS. | Intended product sitemap `lastModified`. | Required update time for catalog/sitemap behavior. |

### 5.1 Required specification data

These values are displayed by the current product detail when present and therefore must be representable independently:

| Shopify source key | Required Vastriqo meaning |
| --- | --- |
| `fabric` | Fabric specification. |
| `care-instructions` | Care instructions. |
| `clothing-features` | Clothing features, potentially multi-valued. |
| `neckline` | Neckline. |
| `top-length-type` | Top length type. |
| `material` | Material. |
| `material-composition` | Material composition. |
| `sleeve-length-type` | Sleeve length type. |
| `pattern` | Pattern. |
| `fit` | Fit. |

The target uses business attribute codes/labels, not Shopify namespaces or metaobject IDs. Multi-value ordering and controlled versus free-form values require approval.

### 5.2 Conditional or differently used custom data

| Shopify source key | Verified use | Requirement status |
| --- | --- | --- |
| `color-pattern` | Hidden from detail specifications; BMS mirror values/reference handles become card Color values. | Required business color data for current cards/filters, but not a replacement for variant Color selections. |
| `size` | Hidden from detail specifications; BMS mirror values/reference handles become card Size values. | Required business size data for current cards/filters, but not a replacement for variant Size selections. |
| `age-group` | Queried and explicitly hidden; no direct storefront filter uses it. | Not currently required; possible future taxonomy input. |
| `target-gender` | Queried/hidden; BMS derives `gender`, but its collection-card mapper does not expose the field. | Conditional audience-taxonomy input; current runtime use is not established. |
| `styling_tips` | Only queried/rendered if production runtime configuration adds it. | **UNDECIDED — NEEDS DATA VALIDATION.** |
| `about_this_item` | Only queried/rendered if production runtime configuration adds it. | **UNDECIDED — NEEDS DATA VALIDATION.** |
| Other configured identifiers | Every configured, non-hidden, non-empty field renders generically. | **UNDECIDED — NEEDS DATA VALIDATION.** |
| `custom.style_no` / `shopify.style_no` | BMS admin labels/filters it; no current storefront consumer. | **UNDECIDED** operational/admin identifier; do not confuse it with SKU. |

## 6. Fields Explicitly Not Required by Current Evidence

These must not enter the target merely because Shopify queries or BMS stores them:

- Shopify GraphQL `edges`/`nodes` wrapper structure;
- Shopify GIDs as product or variant primary keys;
- BMS mirror primary key or raw `shopify_products.meta` as the product model;
- Shopify `onlineStoreUrl` and `requiresSellingPlan`;
- Shopify product category/ancestor and Shopify collection relationships, because current UI does not consume them;
- maximum product price and compare-at price in the current UI;
- variant image association in the current UI;
- barcode, weight, weight unit, `requiresShipping`, and `taxable` in the current product experience;
- image/media source IDs, width, and height for current rendering;
- Shopify metaobject IDs and referenced MetaImage objects for specification rendering;
- Shopify SEO override values as a current canonical-page requirement;
- Shopify variant-to-product back-reference returned inside list queries;
- BMS sync timestamps as product business data; they belong to import/reconciliation records;
- BMS `gender` as an automatically trusted taxonomy value; its derivation and production values require validation; and
- BMS/Shopify status strings as the target publication lifecycle.

Some of these may become valid operational requirements after a business decision. Until then they are excluded, not silently discarded from source inventories.

## 7. Fields and Semantics That Cannot Yet Be Determined

| Topic | Why source is insufficient | Required resolution |
| --- | --- | --- |
| SKU | Queried/mirrored but unused by storefront. | Business/operations decision. |
| Vendor/brand | Used in search payload but not displayed as a product concept. | Product/merchandising decision. |
| Tags | Current heuristic use does not establish which tags have lasting meaning. | Production inventory plus taxonomy decision. |
| Compare-at pricing | Queried but not displayed. | Pricing/merchandising decision. |
| Barcode/weight/shipping/tax flags | No current consumer; future fulfillment may need some values. | Operations/shipping/tax decision later. |
| SEO overrides | Current canonical metadata ignores Shopify SEO values. | SEO/content decision after data-quality review. |
| Variant-specific media | Queried on list/BMS but not used to change gallery. | UX/product decision. |
| Audience/gender/age model | Current routes use BMS collections plus fragile text heuristics. | Approved taxonomy and migration mapping. |
| Product type/category model | Current `productType` is a single string; BMS custom collections serve a different purpose. | Decide classification versus merchandising collections. |
| Attribute value governance | Source mixes scalar, JSON list, and metaobject values. | Decide controlled values versus product-specific free text per attribute. |
| Additional production metafields | Environment value is not committed. | Non-mutating production configuration/data inventory. |
| Image override scope | BMS UI is collection-oriented but persistence is product-wide. | Decide product-wide, collection-specific, or retire. |
| Media ownership/provider | URLs are Shopify/CCA/local; no target provider is approved. | Infrastructure decision after functional media requirements. |
| Inventory granularity | No independent inventory model exists. | Single/multiple locations, reservation, overselling, and adjustment policy. |
| Currency scope/storage | UI carries currency codes and BMS defaults INR. | Decide INR-only versus multiple currencies and monetary representation. |
| Publication lifecycle | Current Shopify/BMS states do not define target semantics. | Approve states or a simpler visibility rule and publication prerequisites. |
| Handle changes | Current handle is canonical; no redirect registry exists. | Approve immutability versus redirect history and retention. |
| “Best Sellers” | Current sort is a no-op. | Approve a real ranking rule or relabel as default/recommended order. |
| New-arrival timestamp | Current UI uses Shopify creation time. | Approve product creation, first publication, or merchandising date. |
| Publish prerequisites | Current UI can render missing image/description and BMS price defaults to zero. | Decide minimum completeness and whether zero price is valid. |
| Collection scope in this slice | Current home/gender/related products require BMS relationships, while collections are otherwise a separate module. | Approve minimal native collection import/read now or an explicit temporary BMS dependency. |

## 8. Proposed Vastriqo Product Domain

This proposal is deliberately conceptual and decision-gated.

```text
Product
  |-- ordered ProductMedia
  |-- ordered ProductOption
  |       `-- ordered ProductOptionValue
  |-- ProductVariant
  |       `-- selected ProductOptionValue(s)
  |-- ProductAttributeValue
  |       `-- AttributeDefinition / optional controlled value
  `-- ExternalSourceMapping(s)

Decision-gated dependencies:
Product <-- ordered membership --> CatalogCollection
ProductVariant --> Inventory authority
Product/Variant --> optional handle history / variant media / SEO overrides
```

### 8.1 Core concepts

| Concept | Responsibility | Evidence/constraint |
| --- | --- | --- |
| Product | Stable identity, handle, title, descriptions, classification inputs, publication decision, timestamps, and product-level projections. | Required by list/detail/SEO/analytics. It is not a Shopify Product clone. |
| ProductVariant | Purchasable identity, selected option values, current amount/currency, and effective sellability input/result. | Required by detail selection and later cart integration. Shopify ID is external only. |
| ProductOption | Ordered selectable dimension owned by one product, such as Size or Color. | Current product detail discovers dimensions from variant selections. Names must not be hard-coded to only Size/Color. |
| ProductOptionValue | Ordered normalized/display value belonging to an option. | Required for selectors and filters. |
| VariantOptionSelection | Relates a variant to at most one value from each relevant product option; variant combinations must be unique per product. | Prevents opaque Shopify selected-option arrays and supports exact selection. Whether every variant must select every option needs data validation. |
| ProductMedia | Ordered product media reference, alt text, and an approved role such as featured/gallery. | Multiple ordered images are required. Provider and variant association are not selected. |
| AttributeDefinition | Business code, display label, ordering, and approved value behavior for specifications/discovery. | Justified by ten current specifications plus runtime-configured fields; avoids one column per Shopify key and avoids opaque JSON. |
| ProductAttributeValue | One or more product values for an approved definition, preserving display order where relevant. | Must represent readable scalar/multi-values without Shopify metaobject coupling. Controlled/free-form representation is decision-gated. |
| ExternalSourceMapping | Source system, source entity type, external ID, Vastriqo entity identity, and reconciliation provenance. | Required for deterministic Shopify/BMS import without using external IDs as primary keys. Product and variant mappings are required. |
| CatalogCollection and ordered membership | Named/slugged curated grouping and ordered product relationship. | Required only if home/gender/related surfaces cut over in this slice; not a license to build the full collection module now. |
| ProductHandleHistory | Old handle to current product redirect. | Conditional on the handle-change decision. |

### 8.2 Deliberate non-concepts

- No generic Shopify `metadata` JSON bucket is proposed.
- No separate “Shopify product” runtime entity is proposed.
- No price-list, promotion, discount, subscription/selling-plan, bundle, or multi-market model is proposed.
- No inventory-location/reservation schema is proposed before inventory policy is approved.
- No vendor/brand, SKU, barcode, or SEO-override entity is mandatory before the corresponding decisions.
- No product status enum is invented. The domain requires a public visibility/publication decision, but its state vocabulary is not yet approved.

### 8.3 Domain invariants established by current behavior

- A product has one stable Vastriqo identity.
- A public product URL resolves by unique handle.
- A variant belongs to exactly one product.
- An option and its values belong to one product unless a future shared vocabulary is explicitly approved.
- A variant cannot select two values from the same option.
- Two variants of one product cannot represent the same complete option-value combination.
- Media has deterministic order; a card/metadata image is selected deterministically.
- Current price carries amount and currency and is authoritative at the variant level.
- Sellability is server-authoritative; the presence of an ID is not evidence of availability.
- External source identifiers are unique within source system/entity type and are not domain primary keys.

The minimum variant completeness, price constraints, media requirement, option completeness, and publication prerequisites remain unapproved.

## 9. Proposed Database Concepts

This is not a physical schema. Names and table boundaries remain subject to the approved `vastriqo-api` database conventions.

| Conceptual persistence | Likely relationships and constraints | Status |
| --- | --- | --- |
| Product record | Internal PK; unique current handle; title; approved description forms; classification reference/value; public-visibility data; created/updated and approved merchandising/publish time. | Required concept; exact fields/status unresolved. |
| Variant record | Internal PK; required product FK; current amount/currency; optional approved operational identifiers; publication/sellability inputs. | Required; SKU and physical money representation unresolved. |
| Product option record | Product FK; unique normalized name per product; display name; position. | Required. |
| Product option value record | Option FK; normalized/display value; position; uniqueness appropriate to one option. | Required. |
| Variant-option-value relation | Variant FK + option-value FK; uniqueness preventing duplicate option assignment and duplicate variant combinations. | Required; precise constraint design follows data validation. |
| Product media record | Product FK; durable object/URL reference; alt text; position; featured/role semantics; timestamps. | Required; variant relation and storage provider conditional. |
| Attribute definition record | Stable business code; label; display order; approved value kind/governance. | Required for approved specifications; admin-managed definition scope unresolved. |
| Product attribute value record(s) | Product FK + definition FK; scalar or controlled value relationship; optional position for multi-values. | Required; do not use an undifferentiated JSON document as a substitute. |
| External source mapping record | Source (`SHOPIFY`, `BMS`, etc.), source entity type, source ID, target entity identity, import/reconciliation timestamps; unique source tuple. | Required for product and variant migration. Exact enum/naming is design work. |
| Import/reconciliation record | Run/source/version/checksum/status/counts/error references. | Required migration support concept; not product business data. |
| Collection and membership records | Collection identity/slug/active decision plus product FK and position. | Decision-gated minimal dependency for current merchandising surfaces. |
| Handle redirect record | Unique old handle, product FK, current/retired timestamps. | Conditional. |
| Inventory records | Variant-level stock, optional location, movements/reservations. | Explicitly not designed until inventory decisions are answered. |

### 9.1 Index/query needs established before physical design

- unique current product handle lookup;
- product-to-variant and product-to-media ordered reads;
- product option/value and variant-combination reads;
- published catalog listing with deterministic pagination/sort;
- case-insensitive or normalized search over approved fields;
- filters over classification, Color, Size, audience, and price where approved;
- external product/variant ID lookup for import/reconciliation;
- published handle/update-time sitemap projection; and
- collection membership/order lookup if included.

Index types, full-text technology, cursor/page pagination, collation, and database engine are not selected.

## 10. Proposed Product API Responsibilities and Logical Contracts

Exact endpoint paths and response envelopes are intentionally not proposed. There is no `vastriqo-api` repository yet, and current BMS responses are inconsistent enough that copying their paths/envelopes would be an unsupported decision.

### 10.1 Public catalog responsibilities

1. **List published products** across the complete catalog with deterministic pagination.
2. **Search** approved fields. Current evidence supports title, description, product classification, approved tags/brand, Color, and Size; undecided fields must not be promised before approval.
3. **Filter** by approved classification, Size, Color, price range, and audience. Preserve current semantics unless intentionally changed: AND between filter groups, OR within a multi-selected group.
4. **Sort** by price ascending/descending and approved new-arrival time. A deterministic default order is required; “Best Sellers” requires an approved ranking rule.
5. **Return full-result facets**, not merely values found on the current page, so a paginated UI can render complete Category/Size/Color choices.
6. **Resolve product detail by current handle** and optionally internal ID for admin/integration use. Public legacy Shopify-ID resolution must be bounded migration compatibility, not a permanent product API.
7. **Return a product-detail aggregate** with ordered media, ordered options/values, variants with option selections, current price/currency, authoritative sellability, descriptions, and approved specifications.
8. **Return a sitemap projection** for published products: handle, title, and meaningful update time.
9. **Return collection/related card projections** only if the minimal collection dependency is approved for this slice.

### 10.2 Logical public response projections

`ProductCard` must contain only what current card/discovery consumers need:

- Vastriqo product ID;
- handle and title;
- featured media URL/reference and alt text;
- minimum eligible variant price and currency;
- approved classification/badge;
- Color and Size summary values where present;
- effective product sellability; and
- a stable purchasable/default variant ID only if a card action actually requires it.

Current source note: `ProductCard` contains optional quick-add code, but every audited call site leaves `allowAddToCart` false. A default variant is therefore not a current card-rendering requirement, though stable variant identity is required for later cart integration.

`ProductDetail` must contain:

- Vastriqo ID, handle, title, and descriptions;
- ordered media and selected featured media;
- ordered option definitions/values;
- variants, each with its option selections, current price/currency, and authoritative sellability;
- approved specifications as ordered label/value data or stable attribute codes plus display values;
- product-level price summary and sellability projection; and
- metadata inputs required by the current slug page.

It must not expose Shopify GraphQL edges/nodes, metafield namespaces, metaobject IDs, Shopify GIDs, or BMS mirror JSON.

### 10.3 Admin product responsibilities

- paginated product list/search/filter;
- product detail for editing;
- create and update approved product fields;
- manage options, values, variants, and unique combinations;
- manage variant pricing and approved sellability/publication inputs;
- manage product media and order;
- manage required attributes/specifications;
- validate handle uniqueness and product completeness;
- preview/evaluate publication eligibility without inventing lifecycle states;
- expose migration source mappings/reconciliation evidence to authorized operations where useful; and
- provide deterministic validation/conflict/not-found behavior.

Admin writes require an authenticated/authorized Vastriqo admin boundary. Admin authentication is a separate platform capability and is not authorized for implementation in this slice without an explicit prerequisite decision.

### 10.4 Import/reconciliation responsibilities

- read Shopify/BMS through isolated import adapters, never through runtime catalog reads;
- transform source records into approved domain commands;
- use external-source mappings for repeatable upsert behavior;
- preview changes and validation failures before writes;
- record source identity/version/checksum and per-record outcomes;
- detect missing/deleted/unpublished source records without automatically deleting native data;
- compare counts and required fields; and
- support safe reruns without duplicate products, variants, media, or option values.

### 10.5 Validation/error behavior requiring later contract design

The API must distinguish at least invalid input, not found, unique-handle conflict, invalid option/variant combination, invalid price/currency, publication-ineligible product, stale/concurrent update where applicable, and import mapping/reconciliation failure. Exact HTTP status mapping, error codes, payload envelope, pagination style, and concurrency mechanism await `vastriqo-api` conventions.

## 11. Vastriqo Admin Products Requirements

The first native admin module must manage the Vastriqo model, not a Shopify snapshot.

### 11.1 Product list

- Paginated list with image, title, handle/internal ID, current price summary, approved classification, publication visibility, and sellability summary.
- Search by title/handle and approved operational identifiers.
- Filters only for approved concepts; do not reproduce the current ineffective BMS gender filter or Shopify status strings blindly.
- Deterministic sorting and clear empty/error/loading states.
- Create action and detail/edit navigation.
- Optional source-mapping/import health indicators only where useful for migration; Shopify IDs must not dominate the UI.

### 11.2 Create/edit/detail

- Edit title, handle under the approved handle policy, descriptions, classification, and approved tags/brand/SEO inputs.
- Define ordered options/values and valid variant combinations.
- Edit each variant's current price/currency and approved SKU/operational fields.
- Order media, select/derive featured media, and edit alt text.
- Edit approved attributes/specifications with the correct controlled/free-form behavior.
- Show publication eligibility and allow only approved publication transitions.
- Show authoritative sellability/inventory information only after the inventory boundary is approved.
- Prevent silent data loss when removing an option/value used by variants.
- Surface validation at field and aggregate level.

### 11.3 Migration/admin support

- Import preview showing creates, updates, unchanged records, warnings, and failures.
- Source comparison for handle/title/variant count/price/currency/media/options/specifications/publication/inventory inputs.
- Per-product source identifiers available for diagnosis, not as editable primary identity.
- No final-state “sync from Shopify” dependency. Import is migration tooling with an exit condition.

### 11.4 Explicit exclusions

- No customers, carts, checkout, orders, payment, shipping, tax, wishlist, CMS, wallet, CCA product, or CCA production-order screens.
- No generic Shopify metafield editor.
- No generic JSON metadata editor.
- No Shopify Admin link as a required final operation.
- No inventory adjustment UI until location/stock/overselling/reservation responsibilities are approved.

## 12. Shopify-to-Vastriqo Migration Mapping

Shopify is the preferred product-data source where direct export/Admin data is available. BMS mirror data is a secondary comparison/curation source.

| Shopify source | Target concept | Treatment |
| --- | --- | --- |
| Product GID | ExternalSourceMapping -> Product | Required external mapping; never target PK. |
| `handle` | Product handle | Required; validate uniqueness and decide redirect handling. |
| `title` | Product title | Required. |
| `description` | Plain/searchable description | Required when available or derive safely from approved rich description. |
| `descriptionHtml` | Rich description | Required for current detail experience; sanitize/validate. |
| `productType` | Product classification | Transform through approved taxonomy; do not copy uncontrolled values blindly. |
| `vendor` | Brand/vendor | Import only if approved. |
| `tags` | Approved tags/discovery values | Conditional; inventory and map approved values only. |
| `createdAt` / `publishedAt` | Approved merchandising/publication timestamps | Conditional transformation after the new-arrival rule is chosen. |
| `updatedAt` | Source version/reconciliation; product update/sitemap input | Required for import comparison; do not overwrite native update semantics blindly. |
| Storefront visibility/Admin status | Product publication input | Transform with an approved state/visibility mapping; never copy enum names automatically. |
| Featured image | ProductMedia featured role | Required selected media. |
| Images/media order | Ordered ProductMedia | Required selected set; deduplicate and validate URLs/files/alt/order. |
| Media/image IDs, dimensions | External mapping or omit | Not current domain requirements; retain only if import/rehosting needs them. |
| Product options | ProductOption/ProductOptionValue | Required; normalize names/values/order. |
| Variant GID | ExternalSourceMapping -> ProductVariant | Required external mapping. |
| Variant selected options | VariantOptionSelection | Required; validate combinations and missing values. |
| Variant price/currency | Current variant price | Required. |
| Minimum price | Derived product card projection | Recompute from eligible native variants rather than treating source range as authority. |
| Variant availability/inventory quantity | Inventory/sellability initialization input | Required for cutover validation; mapping waits on inventory policy and authoritative snapshot timing. |
| SKU | Optional variant operational field | Do not import into the domain until approved; retain in source staging/report. |
| Compare-at price | Optional promotional price | Do not import into active domain until approved. |
| Barcode/weight/shipping/tax flags | Optional operational fields | Do not import into active domain until approved by later operations requirements. |
| Variant image | Optional variant-media relation | Do not model/import until approved. |
| SEO title/description | Optional SEO override | Not a current requirement; review quality and approve before import. |
| `fabric` | Fabric attribute | Required. |
| `care-instructions` | Care instructions attribute | Required. |
| `clothing-features` | Clothing features attribute value(s) | Required. |
| `neckline` | Neckline attribute | Required. |
| `top-length-type` | Top length type attribute | Required. |
| `material` | Material attribute | Required. |
| `material-composition` | Material composition attribute | Required. |
| `sleeve-length-type` | Sleeve length type attribute | Required. |
| `pattern` | Pattern attribute | Required. |
| `fit` | Fit attribute | Required. |
| `color-pattern` | Color merchandising/filter input | Transform approved readable values; variant Color remains purchase authority. |
| `size` | Size merchandising/filter input | Transform approved readable values; variant Size remains purchase authority. |
| `target-gender` | Possible audience classification | Conditional; map only after taxonomy and data validation. |
| `age-group` | Possible audience classification | Not currently required; retain only in source inventory until approved. |
| `styling_tips`, `about_this_item`, runtime extras | Possible approved attributes/content | Do not map until production configuration/data and business use are confirmed. |
| Metaobject references | Attribute values | Flatten approved readable business value or map to an approved controlled vocabulary; do not retain Shopify metaobject identity as the domain. |
| Shopify categories/collections | None by default | Current storefront does not use them for merchandising; BMS custom relationships are the relevant source. |

### 12.1 Shopify source limitations to validate

- Storefront list is capped at 100 products and is unsuitable for migration counts.
- Detail query caps variants at 50, media previews at 5, and images at 100.
- BMS Admin sync caps variants at 100, images at 20, metafields at 30, and references at 10 per field.
- Production optional metafield identifiers are unknown.
- No audited deletion/unpublish webhook/reconciliation exists.

Migration must use a complete paginated source export/query and report truncation or source inconsistencies rather than relying on current storefront/BMS projections.

## 13. BMS-to-Vastriqo Migration Mapping

| BMS source | Target concept | Treatment |
| --- | --- | --- |
| `shopify_products.id` | None as domain identity | BMS-only mirror key; optional external mapping only if reconciliation needs it. |
| `shopify_products.shopify_product_id` | Shopify external mapping correlation | Use to join BMS curation to the Shopify-imported native product. |
| `title`, `price`, `url`, base `image_url` | Reconciliation comparison | Shopify/direct export remains preferred product source; report differences. |
| `meta` JSON | Staging/reconciliation evidence | Parse explicitly; never copy as target product metadata. |
| `gender` | Possible audience mapping evidence | Validate derivation/values; do not trust automatically. |
| `seo_title`, `seo_description` | Optional comparison | They are Shopify/fallback snapshots, not proven target requirements. |
| Shopify/source/sync timestamps | Import provenance | Store in migration/reconciliation evidence only as appropriate. |
| `shopify_categories` | Minimal CatalogCollection | Required only if current home/gender/related merchandising cuts over in this slice; validate active records/slugs. |
| `shopify_category_products` | Ordered collection membership | Transform Shopify product IDs to Vastriqo IDs and preserve validated order. |
| `shopify_products.image_url` retained override | Product or membership media override | **UNDECIDED.** Detect differences from Shopify featured image and report; do not assign target meaning yet. |
| Related-product shared-category rule | Related-product read rule | Preserve only if minimal collections are included; use native identities and deterministic tie-breaking. |

### 13.1 BMS data-quality risks

- One-way sync may leave deleted/unpublished products in the mirror.
- Collection membership foreign-key enforcement was removed, so orphan/stale memberships are possible.
- Public collection reads gate collection status but do not independently gate mirrored product publication.
- Existing `image_url` is preserved on every resync, so it may differ intentionally or accidentally from Shopify.
- BMS collection-card Color/Size comes from metafields, not necessarily variant option truth.
- The admin passes a `gender` filter, but `SharedShopifyCollectionService.listProducts` contains no corresponding `query.gender` filter branch.
- BMS admin status filtering reads `meta.status`; case/value consistency with Shopify data needs validation.
- BMS product “editing” is read-only; there is no native data-entry history to preserve.

## 14. Required Testing and Reconciliation for the Later Implementation

No tests are added in this analysis phase. The current applications contain no established automated test runner, so test tooling must be selected with the new repositories rather than invented here.

The approved implementation must eventually prove:

### Domain/database

- product and handle uniqueness/validation;
- option/value ownership and ordering;
- unique variant combinations and valid selections;
- media ordering/featured selection;
- approved attribute scalar/multi-value handling;
- current price/currency validation;
- publication/sellability rules after approval; and
- idempotent external mapping/import behavior.

### API

- published list pagination, deterministic sorting, filters, search, and facets;
- detail-by-handle response and not-found behavior;
- no unpublished/ineligible record leakage;
- stable Vastriqo IDs in public/admin responses;
- validation/conflict errors for admin writes; and
- sitemap projection contains only approved published products.

### Migration

- product, variant, option/value, media, required attribute, price/currency, handle, and publication-input mappings;
- deterministic reruns without duplicates;
- count reconciliation and explicit discrepancies;
- source truncation detection;
- BMS collection membership/order mapping where approved;
- external-ID uniqueness; and
- no writes to Shopify or BMS.

### Admin and storefront

- admin create/edit/detail for every approved product concept;
- variant/option/media/attribute ordering and validation;
- storefront `/products`, `/collections` as approved, `/search`, category surfaces as approved, product cards, slug detail, variants, price, media, specifications, and sellability against `vastriqo-api`;
- product analytics continue with Vastriqo IDs;
- wishlist/cart consumers can later map to stable Vastriqo IDs without preserving Shopify IDs; and
- manual local storefront verification against representative imported products, not API tests alone.

### Reconciliation report

At minimum report source versus target:

- total products and published products;
- variants per product and total variants;
- handle collisions/changes;
- missing or changed titles/descriptions;
- price/currency discrepancies;
- missing/duplicate media and ordering differences;
- option/value/combination differences;
- missing/unknown required specifications;
- publication/sellability/inventory discrepancies; and
- collection membership/order differences where included.

## 15. Questions Requiring User/Business Decision

This section records the decision gaps found when the product vertical slice was analyzed. The later full-platform architecture resolved three cross-domain questions: native collections/ordered membership are included now (#16), independent admin authentication/authorization is required before admin writes (#18), and the API uses the selected-convention rules in `PHASE-1-CONCEPTUAL-ARCHITECTURE.md` §11 (#19). `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` now reclassifies every remaining item into already decided, engineering decision, production-data validation, genuine business decision or safe deferral. This retained list is not the current approval gate.

### 15.1 Product semantics

1. **SKU:** Is SKU required for every variant, optional, or excluded from the first product slice? Must it be unique globally?
2. **Brand/vendor:** Should Vastriqo have a first-class brand/vendor field, and should it be visible/filterable/searchable?
3. **Classification and audience:** What controlled vocabulary should replace raw `productType`, tag/text gender heuristics, `target-gender`, and `age-group`? Are product classification and Men/Women/Kids audience separate concepts?
4. **Tags:** Should tags remain an admin-managed/searchable concept? If yes, which production tags have approved meaning?
5. **Compare-at pricing:** Is strike-through/compare-at pricing required now, later, or not at all?
6. **Currency:** Is the catalog INR-only, or must a variant support prices in more than one currency?
7. **New arrivals/default ordering:** Should “New Arrivals” use first publication time, product creation time, or a manual merchandising date? Should the current no-op “Best Sellers” be renamed to default/recommended ordering or replaced by a defined ranking?
8. **Publication:** What public visibility states/rules are required, and what minimum data must exist before publication (variant, nonzero price, media, description, inventory)?
9. **Handle changes:** Are handles immutable after publication, or must old handles redirect? If redirects are required, for how long?

### 15.2 Attributes and media

10. **Attribute governance:** For each required specification, are values free-form, controlled, or multi-select? Who may manage the vocabulary?
11. **Runtime custom data:** What is the production value of `SHOPIFY_PRODUCT_METAFIELD_IDENTIFIERS`, and are `styling_tips`, `about_this_item`, `style_no`, or other fields actually populated/required?
12. **Variant media:** Should selecting Color/Size change the product image, or is product-level ordered media sufficient?
13. **Image override:** Is the current BMS override intended to be product-wide, collection/placement-specific, or retired?
14. **Publishable media rule:** Must a product have a featured image before publication, or is the current fallback/no-image behavior acceptable?

### 15.3 Inventory and merchandising dependency

15. **Inventory:** Is stock single-location or multi-location for the first independent implementation? Are overselling/backorders allowed, and may the product slice expose imported quantity before reservation/adjustment workflows exist?
16. **Collection dependency:** Should this product slice include the minimal native collection records and ordered membership needed for home/Men/Women/Kids/related products, or should those surfaces remain explicitly BMS-backed until a later collection slice?

### 15.4 Implementation foundation

17. **New repositories:** Where should the required separate `vastriqo-api` and `vastriqo-admin` repositories be created, and which approved runtime/framework, database engine, package manager, and baseline versions should they use? Existing documents intentionally leave these open.
18. **Admin write security:** Must a minimal independent admin-auth foundation be authorized as a prerequisite before enabling Product admin writes, or should the first Product admin deliverable be limited to non-production/local analysis and UI contracts until the authentication phase?
19. **API conventions:** Should the new API deliberately adopt selected BMS conventions (route/controller/service split, Knex migrations, snake_case, pagination/response helpers), and which known BMS inconsistencies must be excluded? The source establishes candidates, not an approved contract.

## 16. Assumptions Explicitly Refused

This analysis intentionally did not assume:

- that Shopify's product, variant, collection, metafield, metaobject, SEO, publication, or inventory schema is the target;
- that BMS `shopify_products`, its raw `meta` JSON, or CCA `products` is reusable as the target model;
- that Shopify/BMS IDs are permanent Vastriqo IDs;
- that every queried/stored field is required;
- that SKU, brand/vendor, tags, barcode, weight, compare-at price, tax/shipping flags, or SEO overrides are required;
- that current Shopify status strings should become Vastriqo statuses;
- that INR-only or multi-currency pricing is correct;
- that inventory is single-location, multi-location, reservable, or oversellable;
- that `availableForSale` or an existing variant ID is sufficient independent sellability logic;
- that BMS image overrides are product-wide or collection-specific in business intent;
- that `styling_tips`, `about_this_item`, `style_no`, or production-only custom fields are required;
- that every attribute should be free-form, controlled, or stored as JSON;
- that “Best Sellers” has a real current rule;
- that Shopify creation time is the correct new-arrival date;
- that a CMS, generic metadata editor, full collections module, wishlist, cart, checkout, customer, order, payment, shipping, or tax implementation belongs in this slice;
- that media should remain on Shopify/CCA S3 or use any particular provider;
- that an unauthenticated production admin write API is acceptable;
- that `bms-api` response/path conventions are automatically the new API standard; or
- that framework, database, repository scaffolding, test runner, or deployment choices may be selected silently.

## 17. Stop Gate

Phase A repository inspection and Phase B product dependency analysis are complete. The final closure register establishes that Phase 2 detailed design may begin without waiting for every optional product/provider/data choice. Production product/metafield/collection/inventory validation must occur before the affected catalog/import contracts are frozen. Application implementation and repository initialization remain outside Phase 2 and are not authorized here.

No application source, configuration, dependency, database, Shopify data, or BMS data was changed to produce this document.

## 18. Evidence Reference Index

### Storefront

- `vastriqo/lib/shopify/index.ts`
- `vastriqo/components/collections/sections/collections-grid-section/products-client.tsx`
- `vastriqo/components/collections/details-page/index.tsx`
- `vastriqo/lib/shopify-api-collection-products.ts`
- `vastriqo/lib/product-metafield-options.ts`
- `vastriqo/components/collections/category-collection-page/index.tsx`
- `vastriqo/components/home/home-page/index.tsx`
- `vastriqo/app/products/page.tsx`
- `vastriqo/app/collections/page.tsx`
- `vastriqo/app/search/page.tsx`
- `vastriqo/app/[product-slug]/page.tsx`
- `vastriqo/app/details/page.tsx`
- `vastriqo/components/cart/cart-context.tsx`
- `vastriqo/components/cart/cart-page.tsx`
- `vastriqo/components/account/account-pages/wishlist-page.tsx`
- `vastriqo/lib/analytics.ts`
- `vastriqo/lib/seo.ts`
- `vastriqo/lib/internal-products.ts`
- `vastriqo/app/sitemap.ts`
- `vastriqo/lib/config/server-env.ts`
- `vastriqo/.env.example`

### BMS product mirror and merchandising

- `bms-api/src/shared-services/shopifyCollection.ts`
- `bms-api/src/types/shopifyCollection.ts`
- `bms-api/src/api/admin/modules/shopify-collections/route.ts`
- `bms-api/src/api/admin/modules/shopify-collections/controller.ts`
- `bms-api/src/api/admin/modules/shopify-apis/route.ts`
- `bms-api/src/api/admin/modules/shopify-apis/controller.ts`
- `bms-api/src/api/front/modules/collections/route.ts`
- `bms-api/src/api/front/modules/collections/controller.ts`
- `bms-api/src/db-migrations/20260527130000_shopify_product_collections.ts`
- `bms-api/src/db-migrations/20260528100000_add_slug_to_shopify_categories.ts`
- `bms-api/src/db-migrations/20260528110000_store_shopify_product_handle_in_url.ts`
- `bms-api/src/db-migrations/20260615100000_add_gender_to_shopify_products.ts`
- `bms-api/src/db-migrations/20260615101000_add_seo_to_shopify_products.ts`
- `bms-api/src/db-migrations/20260618100000_add_shopify_updated_at_to_products.ts`
- `bms-api/src/db-migrations/20260618110000_drop_shopify_category_products_product_fk.ts`

### Current admin

- `bms-admin/app/components/shopify/ProductsTab.tsx`
- `bms-admin/app/components/shopify/CategoryProductsManager.tsx`
- `bms-admin/app/components/shopify/CategoriesTab.tsx`
- `bms-admin/app/components/shopify/CategoryFormModal.tsx`
- `bms-admin/app/components/shopify/CategoryDetail.tsx`
- `bms-admin/app/api-hooks/shopify.ts`
- `bms-admin/app/types/shopify.ts`
