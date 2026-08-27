# Vastriqo Phase 2B — Product/Catalog Contracts and Delivery Specification

Date: 2026-08-26 (Asia/Kolkata)  
Status: **Design complete; production validation and Phase 2B review remain gates**  
Scope: Product/Catalog contracts and delivery specification only

## 1. Purpose, Authority, and Normative Artifacts

This document turns the approved Phase 2A foundation into deterministic Product/Catalog HTTP, command, concurrency, error, projection, import, media, administration, and acceptance contracts. It does not initialize or implement either target application.

Authority order:

1. The final Phase 1 closure controls business disposition.
2. `PHASE-2A-TECHNICAL-FOUNDATION-AND-PRODUCT-SCHEMA.md` controls the technical foundation and physical schema.
3. The Phase 2A approval/review instructions supplied for this phase control its six explicit review concerns. No separate repository approval document exists.
4. This document controls Product/Catalog behavior and delivery semantics.
5. `contracts/vastriqo-product-catalog-v1.openapi.yaml` is normative for the HTTP wire contract. A disagreement between it and this document blocks approval until corrected.
6. `PHASE-2B-READ-ONLY-PRODUCTION-VALIDATION-RUNBOOK.md` controls the validation gate and retained evidence.

Keywords **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are normative. Examples are synthetic contract illustrations, not production evidence.

## 2. Evidence and Traceability

### 2.1 Documents and source inspected

- Final Phase 1 closure, conceptual architecture, requirements disposition, dependency audit, and Product/Catalog design.
- Approved Phase 2A foundation and physical schema, project direction, and `report.md`.
- `vastriqo` Shopify client, Product list/search/category/detail routes, filters, cards, sitemap, BMS collection adapter, metafield formatting, related-products BFF, analytics, wishlist and cart-facing Product identity use.
- BMS Shopify Product mirror/collection service, controllers, routes, migrations, admin hooks, Product mirror screens, collection editor, ordering, and image selection behavior.

### 2.2 Evidence classification

| Class | Meaning in this specification |
| --- | --- |
| Approved architecture | Normative Phase 1/2A decision; Phase 2B must not silently change it. |
| Current implementation evidence | Demonstrable storefront/BMS behavior used to preserve or deliberately fix behavior. |
| Engineering decision | Phase 2B contract choice supported by the approved foundation. |
| Validation-required | Production fact is unknown; the contract exposes a bounded mapping point but does not assert a value. |
| Deferred | Valid later capability that is outside the Product/Catalog delivery boundary. |

### 2.3 Current behavior and treatment

| Evidence | Classification | Target treatment |
| --- | --- | --- |
| `/products`, `/collections`, and `/search` call Shopify `products(first: 100)` and filter/sort in the browser. | Current deficiency | Replace with complete server-side query, facets, exact count, and keyset pagination. Do not preserve the cap. |
| Search currently checks title, description, vendor, product type, tags, Color, and Size by substring. | Current behavior plus source coupling | Preserve approved discovery meaning through native classification, terms, options, and attributes. Do not expose vendor/tags unless separately activated. |
| “Best Sellers”/recommended has no ranking implementation. | Broken claim | Retire the claim. `recommended` uses stable merchandising order. |
| New-arrival sort uses Shopify `createdAt`. | Current intent | Use native `merchandising_at`, seeded only after validation. |
| Detail rich query caps Variants at 50, media previews at 5, images at 100, and falls back to a basic query. | Current deficiency | Native detail returns the complete approved Product aggregate; no silent rich-data fallback. |
| Detail selection accepts partial selection and can choose the first matching Variant. | Broken behavior | Fix: a purchasable selection must map to exactly one complete active Variant signature. |
| Detail shows ordered images, descriptions, selected Variant price, availability, options, and non-hidden specifications. | Current requirement | Preserve through native ordered contracts. |
| `/collections` renders all Products. | Approved behavior | Storefront continues to call the Product list. No public collection landing is invented. |
| Men/Women/Kids routes and related Products use BMS custom collections and membership order. | Current requirement | Native Collection reads/memberships preserve this behavior with publication enforcement. |
| BMS retains `shopify_products.image_url` across sync and lets admins choose/paste it while assigning Products. | Validation-required | Import a product-wide card override only if the read-only validation classifies a live difference as deliberate. |
| BMS sync/query limits and removed membership FK permit truncation, stale mirrors, and orphans. | Current deficiency | Complete source extraction, quarantine, and explicit reconciliation replace silent trust. |
| Current public Product and Variant IDs are Shopify identities used by analytics/wishlist/cart. | Transitional dependency | Public contracts use native UUIDs. Later consumers migrate without retaining Shopify identity. |

No inspected evidence contradicts the approved Phase 2A physical schema.

## 3. Inherited Decisions and Explicit Non-Goals

Inherited unchanged: separate peer repositories; Node/Nest/Next foundations; PostgreSQL 18 and Kysely; native UUIDv7 identity; Product handles/aliases; bigint minor-unit money; Product aggregate and Collection aggregate boundaries; normalized options/values/variants/selections; governed attributes; native media keys; location-aware Inventory Positions; synchronous Product projection; transactional outbox; typed source mappings; deterministic imports; RFC 9457-style problems; and secure admin-session/BFF topology.

This phase does **not**:

- create `vastriqo-api` or `vastriqo-admin`;
- create code, migrations, packages, database objects, fixtures, credentials, infrastructure, or provider configuration;
- perform production validation, imports, external writes, or Shopify/BMS sync;
- redesign the Phase 2A tables;
- select storage/CDN, queue, hosting, payment, shipping, or tax providers; or
- design customer, cart, checkout, order, promotion, CMS, or advanced warehouse domains.

## 4. HTTP and Versioning Conventions

### 4.1 Versioning and media types

- All Phase 2B HTTP operations use `/api/v1`.
- Breaking semantic or wire changes require a new major path. Additive optional response fields and new endpoints may remain in v1.
- Successful JSON uses `application/json`. Problems use `application/problem+json`.
- There is no generic `{ success, data, message }` envelope.
- Request/response property names use lower camel case. Domain codes/enums use lowercase snake case unless a documented external standard controls them.
- IDs are lowercase UUID strings. Timestamps are RFC 3339 UTC strings. Versions are unsigned decimal strings because the database value is `bigint`.
- Arrays are required and empty when no items exist. Singular absence uses an explicitly nullable field; arbitrary omission is not used to hide inconsistent loading.

### 4.2 Audience and authorization

| Audience | Authentication | Authorization |
| --- | --- | --- |
| Public catalog | Anonymous | Rate/abuse controls only; only published data. |
| Admin read | Opaque AdminSession via the admin BFF | `catalog:read`, `migration:read` where provenance is returned. |
| Admin Product write | Same | `catalog:write`; publication additionally needs `catalog:publish`; media needs `catalog:media`. |
| Merchandising write | Same | `merchandising:write`. |
| Import | Same | `migration:read` or `migration:run`. |
| Repair | Same | `catalog:repair`. |

An API handler MUST authorize every request. Browser route guards are not authorization.

### 4.3 Headers

| Header | Rule |
| --- | --- |
| `X-Correlation-ID` | Client MAY send a safe opaque value; API validates/normalizes or generates one and always returns it. |
| `ETag` | Admin aggregate reads and successful aggregate writes return a strong quoted decimal version, for example `"12"`. |
| `If-Match` | Required on every mutation of an existing Product, Collection, or AttributeDefinition aggregate. Wildcard is forbidden. |
| `Idempotency-Key` | UUID required on create commands, lifecycle POSTs, upload create/complete, import/cancellation, and rebuild requests. |
| `Location` | Returned for creates, accepted operations, and canonical-handle redirects. |

Idempotency scope is `(authenticated principal, HTTP operation, key)`. The server stores the normalized request hash and final/accepted response. Same key plus same request returns that result; same key plus a different request returns `409 IDEMPOTENCY_KEY_CONFLICT`.

## 5. Public Product/Catalog Contract

### 5.1 Public endpoint matrix

| Method/path | Purpose | Request | Success | Errors | Execution |
| --- | --- | --- | --- | --- | --- |
| `GET /api/v1/catalog/products` | List/search/filter Products | §5.3 query | `200 ProductListResponse` | 400 cursor/query; 422 range/currency | Sync, read-only |
| `GET /api/v1/catalog/products/{handle}` | Resolve Product detail | Normalized handle | `200 ProductDetail`; alias `308` | 404 handle | Sync, read-only |
| `GET /api/v1/catalog/products/{productId}/related` | Shared-Collection related cards | UUID, `limit` 1–12 default 8 | `200 ProductCardPage` | 400/404 | Sync, read-only |
| `GET /api/v1/catalog/collections` | Published collection metadata | Cursor, limit | `200 CollectionPage` | 400 | Sync, read-only |
| `GET /api/v1/catalog/collections/{slug}` | Published Collection metadata | Slug | `200 Collection` | 404 | Sync, read-only |
| `GET /api/v1/catalog/collections/{slug}/products` | Ordered published members | Cursor, limit | `200 ProductListResponse` | 400/404 | Sync, read-only |
| `GET /api/v1/catalog/sitemap/products` | Published sitemap/delta | `updatedAfter`, cursor, limit | `200 SitemapProductPage` | 400/422 | Sync, read-only |

There is no separate search endpoint: `q` on the complete Product list is the single search/filter/facet contract. `/collections` in the current storefront continues to consume the all-Product list rather than inventing a Collection landing.

### 5.2 Exact public schemas

`Money`:

```json
{ "amount": "1299.00", "currency": "INR" }
```

- Both fields are required. `amount` is a non-negative decimal string at the currency exponent. Public Product prices are positive.

`PublicMedia`:

```text
id: UUID
url: absolute HTTPS URL
altText: string (never null; Product title fallback already resolved)
role: gallery | card_override
position: non-negative integer
featured: boolean
```

Neither `asset_key` nor source/provider identifiers appear.

`ClassificationSummary`: required `id`, `code`, and `name`. The whole field on a Product is nullable until validation confirms publication requires classification.

`OptionValue`: required `id`, `value`, `position`.

`ProductOption`: required `id`, `code`, `name`, `position`, and ordered `values`.

`VariantSelection`: required `optionId`, `optionValueId`, `optionCode`, `optionName`, and display `value`. The repeated display fields make the public projection usable without exposing persistence rows; IDs remain authoritative.

`ProductVariant`:

```text
id: UUID
title: string | null
position: integer
selections: VariantSelection[] (ordered by Product Option position)
price: Money
availability: available | sold_out
```

SKU, quantities, inventory mode/policy, lifecycle status, source identity, and lock version are not public.

`SpecificationValue`: required display `value` and `position`.

`ProductSpecification`: required `code`, `label`, `position`, and ordered non-empty `values`. Only active public AttributeDefinitions are projected.

`ProductCard`:

```text
id: UUID
handle: canonical handle
title: non-blank string
cardMedia: PublicMedia | null
price: Money
classification: ClassificationSummary | null
optionSummaries: [{ code, name, values: string[] }]
availability: available | sold_out
```

`price` is the minimum eligible active Variant price. `optionSummaries` contains the approved current card/filter dimensions (initially Color/Size after validation), ordered by Product Option position. There is no default Variant ID because current cards do not quick-add.

`ProductDetail`:

```text
id, handle, title
descriptionText: string | null
descriptionHtml: sanitized string | null
classification: ClassificationSummary | null
media: PublicMedia[]
options: ProductOption[]
variants: ProductVariant[]
specifications: ProductSpecification[]
price: Money
availability: available | sold_out
seo: { title: string, description: string, image: PublicMedia | null }
updatedAt: RFC 3339 UTC
```

Every array is complete and ordered. Detail `price` is also the minimum eligible active Variant price. `seo` is generated from native content/media; no Phase 2A override field is implied.

`Collection` contains required `id`, `slug`, `title`, nullable `description`, and `updatedAt`. It does not expose membership internals or BMS identity.

`PageInfo` contains required `hasNextPage` and nullable `nextCursor`.

`ProductListResponse` contains required `items`, `totalCount`, `facets`, and `pageInfo`. A search result uses this same shape; ranking metadata is not exposed.

`Facet` contains `code`, `label`, `kind` (`terms` or `range`), ordered `values`, and nullable `range`. A term value contains `value`, `label`, `count`, and `selected`. A range contains Money `minimum`/`maximum` or null when there are no results.

`SitemapProduct` contains `handle`, `title`, and `updatedAt` only.

### 5.3 Product query contract

| Parameter | Shape/default | Rule |
| --- | --- | --- |
| `q` | trimmed string, max 200 | Empty becomes absent. Search approved fields only. |
| `classification` | repeatable normalized code | OR within group. |
| `size` | repeatable normalized value | OR within group; native Option/approved Attribute mapping validation required. |
| `color` | repeatable normalized value | OR within group. |
| `audience` | repeatable normalized value | OR within group; validated Collection/Attribute mapping. |
| `minPrice`, `maxPrice` | decimal strings | Both use `currency`; inclusive; min must not exceed max. |
| `currency` | ISO code | Required with a price bound; current validated value expected INR. |
| `availability` | `all` or `available`; default `all` | Sold-out published Products remain visible by default. |
| `sort` | §5.4 enum | Absent means relevance when `q` exists, otherwise `recommended`. |
| `limit` | 1–100, default 24 | Server capped. |
| `cursor` | opaque string | Must match the normalized query excluding cursor. |

Groups combine with AND; repeated values inside a group combine with OR. Unknown filter values produce an empty result, not a transport error, unless syntactically invalid.

### 5.4 Sort and facet rules

| Sort | Ordered tuple |
| --- | --- |
| `recommended` | `merchandising_at DESC NULLS LAST, product_id DESC` |
| `newest` | Same tuple; the customer label is “New Arrivals,” backed by the deliberate merchandising timestamp rather than Shopify creation time. |
| `price_asc` | `currency_code ASC, min_price_minor ASC, product_id ASC` |
| `price_desc` | `currency_code ASC, min_price_minor DESC, product_id DESC` |
| implicit search relevance | full-text rank DESC, trigram similarity DESC, `merchandising_at DESC`, Product ID DESC |

An explicit sort with `q` overrides relevance ordering while search still restricts matches. Price sorts require a single requested/validated currency and never compare currencies.

Facets are calculated over the full filtered result, not the page. AND applies between groups and OR within groups. For each facet, its count query applies search and every other filter but excludes its own selected group; this preserves alternative choices. Price range excludes the current price bounds. Facet ordering follows governed vocabulary/Option position, then normalized label.

### 5.5 Cursor contract

- Cursor bytes are a server-signed base64url structure containing cursor schema version, normalized-query SHA-256, selected sort, and the exact ordered anchor values.
- Cursor contents are opaque and MUST NOT be parsed or constructed by clients.
- Invalid encoding/signature/version/query hash/anchor returns `400 CATALOG_CURSOR_INVALID`.
- Changing search, any filter, sort, or page size requires restarting without a cursor.
- Cursor reads are not snapshot isolation. A Product changed between pages may move; clients refresh from the first page after an intentional query refresh.
- Every ordering ends with native Product ID, preventing ties from producing nondeterministic SQL order.

### 5.6 Handle and related-product behavior

- Canonical published handle returns `200`.
- Active alias for a published Product returns `308` with `Location` set to its canonical API detail URL. The storefront translates this into a permanent customer-route redirect.
- Draft/retired Product, retired alias, or unknown handle returns the same `404 PRODUCT_HANDLE_NOT_FOUND`; public callers cannot enumerate lifecycle state.
- Related Products share at least one published Collection, exclude the current Product, include only published Products, deduplicate by Product ID, and order by best shared membership position then Product ID. Default limit is 8; maximum 12.

## 6. Admin Product and Collection Contract

### 6.1 Admin representations

`AdminProductSummary` includes native ID, canonical handle, title, classification, card media, minimum Money, publication status, availability, updated time, `version`, and optional import-health state visible only with `migration:read`.

`AdminProduct` includes:

- `id`, `version`, timestamps, publication status/timestamps, canonical handle and ordered aliases;
- editable content, classification, and approved search terms;
- complete ordered Options/Values and Variants/Selections including nullable SKU, current Money, lifecycle, inventory mode, and oversell policy;
- active and retired ProductMedia metadata, but never `asset_key` or provider credentials;
- complete governed specifications;
- Collection assignment summaries;
- read-only inventory availability/position summaries appropriate to authorization;
- publication validation summary; the public-shape preview is fetched through the dedicated preview operation; and
- source mapping/reconciliation summaries only with `migration:read`.

Admin and public DTOs are separate. `AdminProduct` is a read model, not a generic write payload.

### 6.2 Common command rules

- All existing-Product commands require `catalog:write` unless a narrower permission is stated, a valid Product `If-Match`, and a Product-row lock.
- A successful synchronous Product command returns `200 AdminProduct`, its new `ETag`, and the same `version` in the body. Create returns `201` and `Location`.
- Each command validates its complete post-command aggregate, not just changed fields.
- Canonical Product state, catalog projection, applicable outbox records, and Product version change in one transaction. A validation/conflict/provider error commits none of them.
- Transport rejects unknown properties. Command bodies never accept source IDs, Product ID, version, timestamps, projection fields, or arbitrary metadata.
- `PATCH` distinguishes omitted (unchanged) from explicit nullable values. `PUT` replaces the complete scoped subresource described below, never the whole Product.

### 6.3 Product read/create/content/handle operations

| Operation | Permission; input | Result | Idempotency/concurrency | Transaction and principal errors |
| --- | --- | --- | --- | --- |
| `GET /api/v1/admin/products` | `catalog:read`; `q`, status, classification, availability, limit/cursor | `200 AdminProductPage` | None; read-only | 400 cursor/query |
| `POST /api/v1/admin/products` | `catalog:write`; `{ title, handle, descriptionText?, descriptionHtml?, classificationId?, searchTerms[] }` | `201 AdminProduct` draft | Required key; no `If-Match` | One Product transaction. 409 handle; 422 validation. Creates native UUID and canonical handle. |
| `GET /api/v1/admin/products/{productId}` | `catalog:read`; UUID | `200 AdminProduct` + ETag | Read-only | 404 Product |
| `GET /api/v1/admin/products/{productId}/preview` | `catalog:read`; UUID | `200 ProductPreview` containing Product `version`, nullable public `ProductDetail`-shape data, validation warnings, and equivalent Product ETag | Read-only; never grants a public draft URL. Projection is null when the draft lacks the minimum data needed to construct it. | 404 Product |
| `PATCH /api/v1/admin/products/{productId}/content` | `catalog:write`; any of `{ title, descriptionText, descriptionHtml, classificationId, searchTerms }`, at least one | `200 AdminProduct` | `If-Match`; no automatic retry | Product transaction. 404/412/422; HTML sanitized before canonical write. |
| `POST /api/v1/admin/products/{productId}/handle-changes` | `catalog:write`; `{ handle }` | `200 AdminProduct` | Required key + `If-Match` | Product transaction locks root, demotes prior canonical, inserts new canonical, projection/outbox. 409 duplicate; 412 stale; 422 invalid. |

`searchTerms` is a complete ordered replacement of approved native terms. Empty removes all terms. Classification null is allowed only while aggregate/publication rules allow it.

### 6.4 Option and Variant operations

| Operation | Request intent/schema | Result | Transaction and errors |
| --- | --- | --- | --- |
| `PUT /products/{productId}/options` | Replace ordered Option topology: `{ options: [{ id?, code, name, position, values: [{ id?, value, position }] }] }`. Existing IDs identify retained rows; missing existing rows request removal. | `200 AdminProduct` | Product transaction; validates affected selections/signatures before change. 409 topology in use/combination/position; 412 stale; 422 shape. |
| `POST /products/{productId}/variants` | Create one Variant: `{ title?, sku?, position, selections: [{ optionId, optionValueId }], price: Money, inventoryMode, oversellPolicy }` | `201 AdminProduct` and created Variant ID in `Location` | Required key + `If-Match`; Product transaction. 409 SKU/signature/position; 422 missing/foreign selection/price. |
| `PATCH /products/{productId}/variants/{variantId}` | Update display/operational fields only: `{ title?, sku?, position? }` | `200 AdminProduct` | `If-Match`; Product transaction. 404 ownership; 409 SKU/position; 412 stale. |
| `PUT /products/{productId}/variants/order` | Complete active ordering: `{ variantIds: UUID[] }` | `200 AdminProduct` | `If-Match`; Product transaction. List must contain every active Variant exactly once. |
| `PUT /products/{productId}/variants/{variantId}/selections` | Complete replacement: `{ selections: [{ optionId, optionValueId }] }` | `200 AdminProduct` | `If-Match`; Product transaction. 409 duplicate signature; 422 incomplete/foreign selection. |
| `PUT /products/{productId}/variants/{variantId}/price` | `{ price: Money }` | `200 AdminProduct` | `If-Match`; Product transaction. 422 precision/currency/negative; publication consistency revalidated. |
| `PUT /products/{productId}/variants/{variantId}/inventory-policy` | `{ inventoryMode: tracked|not_tracked, oversellPolicy: deny|continue }` | `200 AdminProduct` | `If-Match`; Product transaction plus Inventory application interface; 409 invalid policy/state; 422. No quantity adjustment. |
| `POST /products/{productId}/variants/{variantId}/retirement` | `{ reason: non-blank string }` | `200 AdminProduct` | Required key + `If-Match`; Product transaction. Retiring last publishable Variant is rejected while Product remains published. Already retired with same key returns prior result. |

All paths in §§6.4–6.7 are prefixed `/api/v1/admin`. Option topology replacement is deliberately scoped and is not arbitrary Product JSON. Removing an Option/Value used by a Variant requires the same request to leave every retained active Variant complete and unique; otherwise the entire command fails.

### 6.5 Media operations

| Operation | Permission/input | Result | Transaction/errors |
| --- | --- | --- | --- |
| `POST /products/{productId}/media` | `catalog:media`; `{ uploadId, altText?, usage: gallery|card_override, position?, featured? }` | `201 AdminProduct` | Required key + Product `If-Match`; attach only verified completed native upload. Product transaction. 409 role/position/upload reuse; 422. |
| `PUT /products/{productId}/media/order` | `catalog:media`; `{ mediaIds: UUID[] }` containing every active gallery item exactly once | `200 AdminProduct` | Product `If-Match`; one Product transaction; 409 missing/foreign/duplicate. |
| `PUT /products/{productId}/media/{mediaId}/featured` | `catalog:media`; empty object | `200 AdminProduct` | Product `If-Match`; media must be active gallery. Atomically clears previous featured. |
| `PUT /products/{productId}/card-media` | `catalog:media`; `{ mediaId: UUID|null }` | `200 AdminProduct` | Product `If-Match`; null removes active card override; selected media must belong to Product and have/transition to card role according to command validation. |
| `POST /products/{productId}/media/{mediaId}/retirement` | `catalog:media`; `{ reason }` | `200 AdminProduct` | Required key + Product `If-Match`; retires relation, clears/falls back projection roles atomically. Imported mapping remains. |

Missing media is a publication warning rather than a blocker. Card override capability remains inactive for import until production validation proves use; native admins may use it only after that feature is approved in the delivery configuration.

### 6.6 Specifications and publication lifecycle

| Operation | Permission/input | Result | Rules |
| --- | --- | --- | --- |
| `PUT /products/{productId}/specifications` | `catalog:write`; `{ specifications: [{ definitionId, values: [{ allowedValueId?, textValue?, position }] }] }` | `200 AdminProduct` | Complete replacement. `If-Match`. Validates Definition status/mode/cardinality and controlled ownership. |
| `POST /products/{productId}/publication-validations` | `catalog:read`; empty body | `200 PublicationValidation` | Synchronous, no state/version change, no `If-Match`; returns all blockers and warnings. |
| `POST /products/{productId}/publish` | `catalog:publish`; empty body | `200 AdminProduct` | Required key + `If-Match`; draft only; all blockers must be absent. |
| `POST /products/{productId}/unpublish` | `catalog:publish`; `{ reason }` | `200 AdminProduct` | Required key + `If-Match`; published -> draft. |
| `POST /products/{productId}/retirement` | `catalog:publish`; `{ reason }` | `200 AdminProduct` | Required key + `If-Match`; draft/published -> retired. |
| `POST /products/{productId}/restore` | `catalog:publish`; empty body | `200 AdminProduct` | Required key + `If-Match`; retired -> draft only. |

`PublicationValidation` contains `eligible`, ordered `blockers`, and ordered `warnings`; each issue has stable `code`, `path`, and safe message. Publication requires nonblank title, canonical handle, at least one active Variant with complete unique selections, positive price, consistent currency, and every other approved invariant. Media absence warns. Publication and sellability are separate.

### 6.7 Projection operations

| Operation | Input/result | Behavior |
| --- | --- | --- |
| `POST /products/{productId}/projection-rebuilds` | `catalog:repair`; required key; Product `If-Match`; `202 Operation` | Enqueues a Product-scoped canonical-to-projection verification/rebuild. It never changes canonical Product state. |
| `POST /catalog/projection-rebuilds` | `catalog:repair`; required key; `{ scope: all|product_ids, productIds: UUID[] }`; `202 Operation` | Bounded background operation. `productIds` required only for `product_ids`; maximum batch documented as 1,000. |

Ordinary commands synchronously maintain the projection and never require these repair endpoints for correctness.

### 6.8 Collections and membership operations

Admin Collection reads use `GET /api/v1/admin/collections` and `GET /api/v1/admin/collections/{collectionId}` with `catalog:read`. The detail returns `version`, metadata, lifecycle, and complete ordered membership summaries.

| Operation | Input | Result/rules |
| --- | --- | --- |
| `POST /api/v1/admin/collections/{collectionId}/memberships` | `merchandising:write`; required key + Collection `If-Match`; `{ productIds: UUID[] }` | `200 AdminCollection`; append in request order after current maximum position. Duplicate/existing member -> 409. |
| `DELETE /api/v1/admin/collections/{collectionId}/memberships/{productId}` | `merchandising:write`; Collection `If-Match` | `200 AdminCollection`; unknown membership -> 404. Remaining positions are normalized contiguously in the transaction. |
| `PUT /api/v1/admin/collections/{collectionId}/memberships/order` | `merchandising:write`; Collection `If-Match`; `{ productIds: UUID[] }` | `200 AdminCollection`; list must contain every current member exactly once; positions become zero-based list order. |

Collection membership does not publish a Product. Public Collection reads always enforce Collection and Product publication.

## 7. Optimistic Concurrency Contract

### 7.1 Version acquisition and mutation

- `AdminProduct`, `AdminCollection`, and mutable vocabulary responses carry `version` and `ETag`.
- `If-Match` contains exactly the quoted version received from that aggregate. Body versions are not accepted.
- The command selects/locks the aggregate root, compares the version, performs all work, increments once, and returns the new value.
- Missing precondition returns `428 CONCURRENCY_PRECONDITION_REQUIRED`.
- Mismatch returns `412 CONCURRENCY_STALE_VERSION` with extensions `{ currentVersion, current: { id, updatedAt, updatedBy? } }`. No submitted content is echoed.
- Neither API nor admin automatically retries a 412. The administrator reloads and deliberately reapplies changes.

### 7.2 Race behavior

| Race | Required outcome |
| --- | --- |
| Same Product content edits | One commits; the stale command receives 412. |
| Two new Products request same handle | Unique registry permits one; the other receives 409 `PRODUCT_HANDLE_CONFLICT`. |
| Two handle changes on one Product | Product lock/version permits one; the other receives 412. Exactly one canonical remains at commit. |
| Different Products change to the same handle/alias | Global handle PK permits one; other receives 409. |
| Concurrent Option/Variant commands | Product version serializes them; stale command receives 412. |
| Concurrent membership reorder/change | Collection version serializes; stale command receives 412. |
| Concurrent Definition edits | Definition version serializes. Mode/cardinality edit after use receives 409 even when version is current. |

Exactly-one-active-canonical is enforced by: global handle PK and partial unique maximum in PostgreSQL; Product-root locking and command-only writes in application code; domain postcondition requiring one active canonical; and integration/concurrency tests proving the minimum-one invariant at transaction commit.

## 8. Product Aggregate Invariant Responsibility Matrix

| Invariant | Transport | Command/application | Domain | Database | Required proof |
| --- | --- | --- | --- | --- | --- |
| Exactly one active canonical handle | Handle syntax | Lock Product; atomic demote/insert | Postcondition exactly one | Handle PK; partial unique max; canonical-not-retired check | Integration + parallel race |
| Alias continuity | — | Never remove old canonical during change | Old canonical becomes active alias | Handle registry uniqueness/FK | Domain + integration |
| Option/Value same-Product ownership | UUID/array shape | Resolve IDs under Product | Reject foreign ownership | Composite FKs | Integration |
| Complete active Variant selections | Array uniqueness | Load current Options | Exactly one value per Option | PK prevents two values per Option; FKs | Domain + integration |
| Unique Variant signature | — | Recompute canonical signature | Reject duplicate combination | Unique Product/signature | Domain + integration/concurrency |
| Ordered positions | Non-negative integers | Validate complete scoped order | No gaps for admin-managed active lists | Scoped unique/check constraints | Unit + integration |
| Publication eligibility | Shape only | Collect all failures | Lifecycle/publish predicate | Supporting checks only | Domain + HTTP |
| Positive publishable price | Decimal/currency syntax | Exact conversion | Published eligible Variant > 0 | Non-negative amount/check | Money unit + integration |
| Product currency consistency | ISO syntax | Resolve all active Variants | One currency while publishable | Query-supported; cross-row not forced | Domain + integration |
| Lifecycle transitions | Enum body/action | Authorize/current-state check | Explicit transition graph | Status/timestamp checks | Unit + HTTP |
| Attribute value mode | One-of shape | Load active Definition/value | Text vs controlled match | Exactly-one value; composite FK | Unit + integration |
| Attribute cardinality | Array shape | Load all values | Single <= 1; multiple ordered unique | Scoped position/value unique | Unit + integration |
| Controlled value ownership | UUID shape | Resolve against Definition | Reject retired/foreign for new assignment | Composite FK | Integration |
| Media roles | Shape | Resolve owned active media | One featured gallery/one card; retired not featured | Partial uniques/checks | Unit + integration |
| Collection visibility | — | Query joins lifecycle | Membership cannot publish | FKs | Query integration |
| Sellability vs publication | — | Inventory interface/projection update | Separate predicates | Canonical balances/checks | Unit + integration |
| Projection consistency | — | Same Product/Inventory transaction | Projection derived only | FK/checks | Fault-injection integration |

Cross-row/domain rules remain in domain/application code when a database constraint would require hidden triggers or duplicate business logic. Database constraints remain the final protection for local referential, uniqueness, shape, and scalar rules.

## 9. Problem Details and Error Catalogue

### 9.1 Problem shape

```json
{
  "type": "https://api.vastriqo.in/problems/concurrency-stale-version",
  "title": "The product changed",
  "status": 412,
  "detail": "Reload the product before applying this change.",
  "instance": "/api/v1/admin/products/018f.../content",
  "code": "CONCURRENCY_STALE_VERSION",
  "correlationId": "01J...",
  "retryable": false,
  "errors": [
    { "path": "If-Match", "code": "stale", "message": "Expected version 12; current version is 13." }
  ],
  "currentVersion": "13"
}
```

`errors` is present only for field/multi-violation cases. Extensions are allow-listed per code. Public detail is safe and does not confirm hidden resource state.

### 9.2 Stable catalogue

| Code | HTTP | Level | Retryable | Exposure/rule |
| --- | ---: | --- | --- | --- |
| `REQUEST_MALFORMED` | 400 | Request | No | Invalid JSON/query encoding; safe generic detail. |
| `AUTHENTICATION_REQUIRED` | 401 | Request | No | Admin session absent/invalid; no credential detail. |
| `PERMISSION_DENIED` | 403 | Request | No | Authenticated principal lacks the operation permission; no hidden-resource detail. |
| `VALIDATION_FAILED` | 422 | Field | No | `errors[]` for transport/domain field failures. |
| `CATALOG_CURSOR_INVALID` | 400 | Request | No | Invalid/mismatched cursor; do not reveal signature internals. |
| `PRODUCT_NOT_FOUND` | 404 | Aggregate | No | Admin safe; public ID reads use safe generic detail. |
| `PRODUCT_HANDLE_NOT_FOUND` | 404 | Aggregate | No | Same for unknown/non-public/retired. |
| `PRODUCT_HANDLE_CONFLICT` | 409 | Field | No | Admin only; handle already reserved by canonical or alias. |
| `CONCURRENCY_PRECONDITION_REQUIRED` | 428 | Request | No | Missing `If-Match`. |
| `CONCURRENCY_STALE_VERSION` | 412 | Aggregate | No | Current version/summary only for authorized admin. |
| `VARIANT_COMBINATION_INVALID` | 422 | Aggregate | No | Foreign/duplicate selections or invalid signature. |
| `VARIANT_SELECTION_REQUIRED` | 422 | Field | No | Missing one or more current Options. |
| `VARIANT_COMBINATION_CONFLICT` | 409 | Aggregate | No | Combination already belongs to another Variant. |
| `ATTRIBUTE_VALUE_INVALID` | 422 | Field | No | Mode/value/allowed-value mismatch. |
| `ATTRIBUTE_CARDINALITY_VIOLATION` | 422 | Aggregate | No | Too many/few values. |
| `ATTRIBUTE_DEFINITION_IN_USE` | 409 | Aggregate | No | Incompatible vocabulary mode/cardinality change. |
| `PUBLICATION_INELIGIBLE` | 422 | Aggregate | No | All blockers returned as `errors[]`. |
| `LIFECYCLE_TRANSITION_INVALID` | 409 | Aggregate | No | Invalid current -> requested state. |
| `COLLECTION_MEMBERSHIP_CONFLICT` | 409 | Aggregate | No | Duplicate/incomplete order/position conflict. |
| `MEDIA_CONFLICT` | 409 | Aggregate | No | Featured/card/order/ownership/upload reuse conflict. |
| `MEDIA_VALIDATION_FAILED` | 422 | Field | No | Type, size, checksum, or safe-content validation failed; no source URL exposed. |
| `MEDIA_DEPENDENCY_FAILED` | 502 | Aggregate | Yes | A verified upload/copy dependency returned an invalid/failing result; provider detail is hidden. |
| `IDEMPOTENCY_KEY_CONFLICT` | 409 | Request | No | Same key, different normalized request. |
| `IMPORT_CONFLICT` | 409 | Aggregate | No | Native edit/source disagreement; details redacted. |
| `SOURCE_MAPPING_CONFLICT` | 409 | Aggregate | No | Source tuple/target already maps inconsistently. |
| `IMPORT_RUN_NOT_CANCELLABLE` | 409 | Aggregate | No | Terminal run or cancellation boundary passed. |
| `RATE_LIMITED` | 429 | Request | Yes | Includes `Retry-After`. |
| `DEPENDENCY_UNAVAILABLE` | 503 | System | Yes | Safe provider/database readiness failure. |
| `INTERNAL_ERROR` | 500 | System | No | Generic detail and correlation ID only; clients do not replay a write unless idempotency semantics make that explicit. |

SQL exceptions, stack traces, source payloads, tokens, asset keys, and credentials are never serialized.

## 10. Projection and Transactional Outbox Contract

### 10.1 Atomic Product write

Every normal Product mutation follows this boundary:

```text
authorize + validate transport
  -> begin database transaction
  -> lock Product root and compare If-Match
  -> load required aggregate/vocabulary/inventory references
  -> execute domain command and invariants
  -> write canonical Product-owned rows
  -> rebuild affected product_catalog_projections row
  -> append outbox event row(s)
  -> increment Product lock_version
  -> commit
  -> return response
```

Any exception before commit rolls back canonical mutation, projection, outbox, and version. External calls never occur inside this transaction. Inventory-triggered sellability changes update affected projection and outbox in the Inventory-owned transaction through an explicit Catalog projection interface.

### 10.2 Event envelope

```text
eventId: UUIDv7
eventType: stable name including .v1
eventVersion: integer (starts 1)
occurredAt: RFC 3339 UTC
aggregateType: product | collection | variant
aggregateId: native UUID
aggregateVersion: decimal string
correlationId: opaque request correlation ID
causationId: event/command ID | null
idempotencyKey: UUID | null
payload: bounded event-specific object
```

The version in `eventType` identifies the semantic event contract; `eventVersion` identifies its schema revision and must agree. Breaking payload semantics create a new event type/version. Payloads carry only native IDs, state/change kind, and fields a consumer cannot safely obtain later. They never contain a full Product, source JSON, credentials, PII, raw HTML, asset keys, or public signed URLs.

### 10.3 Event catalogue

| Event type | Required payload |
| --- | --- |
| `catalog.product.changed.v1` | `productId`, ordered `changeKinds` from `content|classification|search_terms|options|variants|price|inventory_policy|specifications` |
| `catalog.product.publication-changed.v1` | `productId`, `from`, `to` |
| `catalog.product.handle-changed.v1` | `productId`, `previousHandle`, `canonicalHandle` |
| `catalog.product.media-changed.v1` | `productId`, `changeKind`, `mediaId|null` |
| `merchandising.collection-membership.changed.v1` | `collectionId`, `changeKind`, affected `productIds` capped/batched |
| `inventory.variant-sellability.changed.v1` | `variantId`, `productId`, `availability` |
| `catalog.projection-rebuild-requested.v1` | `operationId`, `scope`, bounded `productIds` or null |

One command may emit multiple semantic events with unique event IDs but the same correlation/idempotency context. Large membership/product lists are split into bounded events rather than creating unbounded JSON.

### 10.4 Delivery and failure behavior

- A worker claims committed outbox rows; uncommitted events are invisible.
- Each consumer records `(consumer_name, event_id)` before/with its effect and treats duplicates as success.
- Transient failures use jittered exponential backoff, capped at 15 minutes, for at most 10 attempts by default.
- Permanent failures or exhausted attempts enter a terminal/dead-letter state with safe code, attempt count, and correlation information. Operators can deliberately replay after correction.
- A terminal downstream event never rolls back committed commerce state. Core Product reads remain correct because the projection was synchronous.
- Repair compares projection with canonical state, reports differences, and replaces only derived projection fields. It never changes canonical rows to match a projection.

## 11. Import and Reconciliation Contract

### 11.1 HTTP operations

| Operation | Input | Result | Authorization/execution |
| --- | --- | --- | --- |
| `POST /api/v1/admin/import-runs` | `{ sourceSystem: shopify|bms|approved_file, scope: product_catalog, mode, mappingVersion, selection: { watermark?, externalIds[]? } }` | `202 ImportRun` + `Location` | `migration:run`, required Idempotency-Key; async |
| `GET /api/v1/admin/import-runs` | source/mode/state/cursor/limit | `200 ImportRunPage` | `migration:read` |
| `GET /api/v1/admin/import-runs/{runId}` | UUID | `200 ImportRun` | `migration:read` |
| `GET /api/v1/admin/import-runs/{runId}/records` | action/status/entity/cursor/limit | `200 ImportRecordPage` | `migration:read` |
| `POST /api/v1/admin/import-runs/{runId}/cancellation` | `{ reason }` | `202 ImportRun` | `migration:run`, required key; cooperative async |

The caller never supplies source credentials, arbitrary source URLs, raw manifests, target IDs, source hashes, or mapping rows. Before inserting an `import_runs` row, the request performs complete read-only source extraction outside a database transaction, constructs the immutable bounded manifest, and calculates its SHA-256. Extraction failure returns a dependency problem and creates no run. The short insert transaction then persists the non-null manifest/hash and a pending run before returning `202`; `Idempotency-Key` is persisted as `request_key`. Processing of manifest records remains asynchronous.

### 11.2 Mode semantics

| Mode | Source scope | Permitted target writes |
| --- | --- | --- |
| `dry_run` | Complete/deliberately selected immutable snapshot | Import run/record diagnostics only. No canonical data or external mappings. |
| `apply` | Complete immutable manifest | Per-aggregate canonical create/update plus mappings and import records, subject to conflicts. |
| `delta` | Bounded records strictly after a validated source watermark/version | Same permitted writes as apply. Cannot infer deletions/absence; scheduled full reconcile remains mandatory. |
| `reconcile` | Complete immutable snapshot | Run/record differences only. No canonical or mapping changes and no baseline advancement. |

An `apply` is not accepted until a matching successful dry run exists for the same manifest hash and mapping version, unless an explicitly authorized emergency override contract is designed later. Phase 2B defines no override.

### 11.3 Run and record behavior

- Run lifecycle: `pending -> running -> succeeded|failed|cancelled`. A run with record conflicts may succeed operationally with summary conflicts; infrastructure/extraction failure makes the run failed.
- Record action: `create`, `update`, `unchanged`, `exclude`, `conflict`, or `error`; status: `pending`, `succeeded`, `warning`, or `failed`.
- Each Product is one bounded transaction including its Options, Variants, Media attachments that are already natively copied, Attributes, Product mapping, projection, outbox, and record outcome linkage.
- Collection/membership and Inventory initialization use their owning aggregate transactions after Product/Variant mappings resolve.
- Failure of one aggregate rolls back that aggregate only and does not undo previously committed aggregates. The final summary is exact.
- No source/provider network call occurs inside a target database transaction.
- Missing source records are reported; they never automatically retire/delete native records.

### 11.4 Mapping and identity resolution

Resolution order:

1. Exact `(source_system, source_entity_type, external_id)` mapping.
2. If absent, create is allowed only when no conflicting native/mapped identity exists.
3. Cross-source BMS curation resolves through its Shopify Product ID correlation to the Shopify mapping, never through title/handle guessing.
4. An existing source tuple pointing elsewhere or a target already mapped incompatibly is `SOURCE_MAPPING_CONFLICT`; neither mapping is changed automatically.

Mappings are created only for included Product, Variant, Media, Collection, and Location records. Excluded test-era customer/order/auxiliary domains receive no targets or mappings.

### 11.5 Field groups and native-edit protection

Each successful imported Product record retains hashes of the normalized source and the exact applied target representation for these field groups in bounded import evidence:

- `content`: title and sanitized descriptions;
- `classification_search`: classification and approved search terms;
- `handle`: canonical handle at initial migration;
- `variant_topology`: Options, Values, Variants, Selections, SKU;
- `pricing_policy`: current Variant Money and inventory policies;
- `media`: copied native media identities/order/roles/alt;
- `specifications`: approved Definition/value representation;
- `publication`: mapped lifecycle and merchandising timestamp; and
- separate Collection and Inventory ownership groups under their aggregates.

The hashes and group codes are stored in the bounded redacted `migration.import_records.details` of the successful record and reached through the mapping's `last_import_run_id`; canonical Product rows do not acquire import columns or source JSON.

For a later apply/delta:

1. Derive desired target group and source hash using the same `mappingVersion` rules.
2. Compare current target group hash with the last successfully applied target hash.
3. If equal, source change may update the group.
4. If current differs but equals the new desired result, record unchanged/source-converged and advance evidence.
5. If current differs and desired also differs, mark the entire Product `IMPORT_CONFLICT`; do not partially update that Product.
6. Admin-only groups/fields are never overwritten by import after their declared handoff.

This protects native edits without adding Shopify-shaped canonical fields. Import diagnostics identify groups and safe hashes, not full sensitive payloads.

### 11.6 Idempotency, reruns, and cancellation

- Same request key and normalized request returns the original run; different request returns 409.
- A deliberate identical-manifest rerun uses a new request key and is allowed. Existing mappings and equal hashes yield `unchanged`, with no duplicate entity/media/value/membership.
- Mapping-version change is a new transform and requires a new dry run even with the same manifest.
- Cancellation is accepted only for pending/running runs. Workers stop claiming new records, finish or roll back the current aggregate transaction, retain completed records, and mark the run cancelled with exact counts.
- A cancelled/failed run is safely rerunnable with a new request key because each aggregate and mapping is idempotent.

### 11.7 Source-to-native mapping procedure

Every mapping row in the final Phase 2B mapping appendix/run output follows:

```text
source field/record
  -> normalization algorithm and mapping-version rule
  -> validation and completeness rule
  -> native target/group
  -> conflict/quarantine behavior
  -> explicit excluded behavior
```

Current mapping boundaries remain those in Phase 2A. Production-specific vocabularies, shapes, live overrides, currencies, SKUs, locations, and policies remain validation-required and cannot be frozen from repository schema alone.

### 11.8 Conditional source-to-native mapping matrix

Every row below is normative in treatment but remains `validation-required` where a production value/shape is unknown.

| Source | Normalization | Validation | Native target/group | Conflict behavior | Excluded behavior |
| --- | --- | --- | --- | --- | --- |
| Shopify Product GID | Preserve exact opaque string | Unique, nonblank, Product type | External mapping -> Product | Existing tuple/other target is mapping conflict | Never native/API ID |
| Shopify `handle` | Trim, lowercase ASCII slug; deterministic collision report | Syntax, normalized uniqueness, historical route inventory | Canonical ProductHandle on initial apply | Collision quarantines Product; later native edit protected | Never route through Shopify at runtime |
| `title` | Unicode-preserving trim/space normalization | Nonblank, <= native maximum | Product content | Native hash divergence conflicts | No BMS fallback without explicit reconciliation decision |
| `description` / `descriptionHtml` | Sanitize allow-listed HTML; derive normalized plain text | Sanitizer result, length/render sample | Product content | Group conflict | Raw/unsafe HTML and scripts excluded |
| `productType` | Map normalized source value through approved vocabulary table | Every publishable value mapped or quarantined | ProductClassification | Unknown value conflicts/quarantines publication | Raw uncontrolled string not stored as classification |
| Shopify `vendor` / `tags` | Map only approved discovery terms; normalize/deduplicate/order | Frequency/use review and vocabulary approval | ProductSearchTerm | Unknowns reported, not silently added | No vendor/tag columns or arbitrary tag copy |
| `createdAt` / `publishedAt` | Select approved stable source timestamp rule | Missing/inconsistent/future timestamps | Initial `merchandising_at` and publication evidence | Native post-handoff edit conflicts | Never overwrite native `created_at`/`updated_at` |
| `updatedAt` | RFC 3339 source token, plus normalized record hash | Coverage/monotonicity for delta | Mapping source version/hash | Regressing/ambiguous token disables delta for record/source | Not Product `updated_at` |
| Shopify Product status/visibility | Versioned map to draft/published/retired input | State/timestamp/Storefront visibility combinations | Publication group | Native lifecycle divergence conflicts | Shopify enum not copied |
| Product Option | Normalize code/name, preserve source order | Duplicate/blank names, count maximum | ProductOption | Topology conflict quarantines aggregate | Source Option ID not native identity/mapping unless later proven necessary |
| Option values | Trim/display preserve, comparison normalize, preserve order | Blank/duplicate normalized value | ProductOptionValue | Topology conflict | No comma-flattened values |
| Shopify Variant GID | Preserve exact opaque string | Unique, belongs to one Product | External mapping -> ProductVariant | Mapping conflict quarantines Product | Never native purchase ID |
| Variant selected options | Resolve normalized name/value to current native Option/Value; canonical ordered signature | Complete, exactly one per Option, unique signature | VariantOptionSelection | Invalid/duplicate combination quarantines Product | Never accept partial storefront selection as authority |
| Variant title | Trim; nullable when derivable | Maximum length | Variant display | Variant topology group conflict | No synthetic title required |
| Variant SKU | Trim, blank -> null, case-normalized comparison | Coverage, duplicates, BMS style-number correlation | Nullable ProductVariant SKU | Duplicate active SKU quarantines affected import pending approved resolution | Barcode/style data not silently substituted |
| Variant price/currency | Parse exact decimal at ISO exponent; convert losslessly to minor units | Missing, negative, zero published, excess scale, mixed Product currency | Variant price/currency | Invalid publishable Product quarantined; no rounding | Compare-at price excluded |
| `inventoryItem.tracked` | Boolean mapping | Complete coverage | Variant inventory mode | Ambiguity blocks Inventory freeze | Do not infer from quantity alone |
| `inventoryPolicy` | `DENY` -> deny; `CONTINUE` only with evidence -> continue | Per-Variant policy and live zero/negative selling behavior | Variant oversell policy | Unknown defaults to deny only for new native data; source record quarantined for migration decision | No global backorder invention |
| Inventory Location GID | Preserve exact opaque ID; approved code/name normalization | Active/online fulfillment meaning, uniqueness | External mapping -> InventoryLocation | Mapping conflict blocks Position | Never assume one Location |
| Inventory quantity name/value | Map only the validated authoritative Shopify quantity semantic | Quantity name, timestamp, tracked policy, multi-location totals | InventoryPosition initialization | Ambiguity blocks tracked Variant initialization | No fabricated movements/history |
| Shopify Media ID | Preserve exact opaque string after successful native copy | Unique Product ownership | External mapping -> ProductMedia | Changed identity/content reported | Never public/media PK |
| Media URL/bytes | Strip query for comparison; safe fetch; decode; SHA-256 bytes; native rehost | Reachability, MIME, size, dimensions, checksum | Native asset then ProductMedia | Failed copy produces error/no active media | Source URL not canonical Product state |
| Media order/alt | Preserve validated source order; trim alt; Product-title fallback at read | Duplicate positions/URLs, missing alt | ProductMedia position/alt | Normalize only by versioned deterministic rule | Signed URLs/provider fields excluded |
| Featured image | Resolve to successfully copied Product media; fallback to first active gallery | Identity/order mismatch | ProductMedia featured role/projection | Missing image warns and reports | No external featured URL in projection |
| Required Shopify metafields | Match approved namespace/key then flatten scalar/list/reference readable value | Type, cardinality, references, unknown values | AttributeDefinition/ProductAttributeValue | Unknown shape/value quarantines affected value/Product as gate defines | Namespace/key/GID/raw JSON not canonical |
| `fabric`, `care-instructions`, `clothing-features`, `neckline`, `top-length-type`, `material`, `material-composition`, `sleeve-length-type`, `pattern`, `fit` | Map to Phase 2A native codes; controlled/free and cardinality from signed vocabulary | 100% shape/value classification | Specification group | Definition mismatch blocks affected mapping | No generic fallback definition |
| `color-pattern` / `size` metafields | Compare with Variant Options; map to controlled filter attribute only when independent meaning is proven | Divergence/population/use | Optional approved Product attributes | Contradiction is validation exception | Do not duplicate Variant purchase truth blindly |
| `target-gender` / `age-group` | Normalize through approved audience vocabulary only if required beyond Collections | Values versus Men/Women/Kids membership and current filter need | Audience Attribute or search/Collection mapping as signed | Unknown/conflicting audience quarantined from facet | No title/tag guessing in target |
| `styling_tips`, `about_this_item`, runtime extras | No mapping until production use/shape is approved | Configuration and storefront-use evidence | Future approved Attribute/content field only | Report as excluded/validation exception | No speculative domain field |
| Shopify SEO override | Compare for validation only | Actual use/quality | None in Phase 2A; generated SEO | No target overwrite | Override fields excluded until activated |
| Shopify compare-at/barcode/weight/tax/shipping/variant image | Inventory/population report only | Later provider/operation requirement | None in Phase 2B | No Product conflict because excluded | Do not migrate |
| BMS `shopify_products.shopify_product_id` | Normalize numeric ID/GID and join exact Shopify mapping | One-to-one correlation | Reconciliation join; optional BMS mapping only if needed | Ambiguous/missing join reported/quarantined | Never Product identity |
| BMS title/price/meta/status/SEO | Normalize for comparison and hash | Difference versus complete Shopify extraction/freshness | ImportRecord diagnostics | Does not overwrite preferred Shopify source | Mirror JSON never canonical |
| BMS `image_url` | Canonicalize comparison URL; classify difference; copy bytes only if deliberate | Every live difference signed as deliberate/stale/equivalent/unresolved | ProductMedia `card_override` only when approved | Unresolved difference disables override import | No Collection placement media semantics |
| BMS `shopify_categories` | Normalize slug/title/description/status/order | Active current set, slug collision, expected placement meaning | Collection + mapping | Unknown/duplicate slug quarantines Collection | Inactive/obsolete rows excluded with report |
| BMS `shopify_category_products` | Resolve exact Shopify Product mapping; normalize zero-based order by `(sort_order,id)` only after exception approval | Orphans, duplicates, gaps, unpublished Products | CollectionMembership | Missing/ambiguous Product or duplicate order reported/quarantined | Orphans never create placeholder Products |
| BMS `gender` / style-number fields | Comparison evidence against approved audience/SKU/search mapping | Coverage/derivation/staleness | Only signed target mapping above | Conflict reported, no automatic priority | No dedicated columns merely because BMS has them |

### 11.9 Ownership and handoff

| State | Source/import rights | Native admin rights |
| --- | --- | --- |
| Pre-apply/dry run | No canonical writes | Normal native editing if Product exists; later apply must detect it. |
| Initial apply to a new Product | May initialize every approved field group | Can edit immediately after commit; edit changes current target hash. |
| Repeated apply/delta before cutover | May update a group only when its current hash equals last applied baseline or source converged to current | Never overwritten on divergence; conflict requires deliberate resolution. |
| Collection/Inventory initialization | Owned by their import operation and aggregate/version, after mappings validate | Membership/policy/native changes receive the same baseline protection. |
| Post-cutover | Import is disabled as a routine Product writer; reconcile remains read-only | Native admin/API is authoritative. A separately approved correction import must use the same conflict rules. |

System-controlled source mappings, hashes, run links, projections, versions, IDs, and timestamps are never admin-editable. Canonical handle, merchandising timestamp, and all Product field groups become native-admin authoritative after cutover; Shopify/BMS status or later changes have no runtime authority.

## 12. Provider-Neutral Media Contract

### 12.1 Direct upload

`POST /api/v1/admin/media/uploads` requires `catalog:media`, an Idempotency-Key, and:

```text
fileName: display/diagnostic name, max 255
contentType: image/jpeg | image/png | image/webp
contentLength: positive integer, maximum 20 MiB
sha256: lowercase 64-character byte checksum
```

It returns `201 MediaUploadIntent` with opaque UUID `uploadId`, `state`, expiry, and provider-neutral upload instructions. Instructions may contain a short-lived HTTPS URL, method, and allow-listed headers but never permanent credentials or an asset key. Intent expiry defaults to 30 minutes.

`POST /api/v1/admin/media/uploads/{uploadId}/complete` requires the same permission and a new Idempotency-Key. The server verifies ownership, expiry, byte count, MIME sniffing, SHA-256, decoder safety, and provider completion outside a Product transaction. Success returns `200 CompletedMediaUpload { uploadId, state: completed, width, height }`. The upload ID may then be attached once to a Product command.

Accepted initial formats are non-animated JPEG, PNG, and WebP. SVG, GIF/animation, executable/polyglot content, invalid decodes, dimensions above 20,000 pixels on either side, and files over 20 MiB are rejected. EXIF and unnecessary metadata are stripped during native processing; orientation is normalized.

### 12.2 Import copy

- Source URL/ID exists only in the import adapter/record and is never accepted from a general Product editor command.
- Import worker fetches outside a database transaction with scheme/host/redirect/size/time limits that prevent SSRF.
- It verifies bytes and checksum, writes the native asset, then opens the Product transaction to create/attach `ProductMedia` and mapping.
- Same source mapping/checksum reuses the prior completed native asset/relation outcome. A changed source byte hash is a media field-group change subject to native-edit conflict detection.
- Copy failure records `MEDIA_PROCESSING_FAILED`; no active ProductMedia is created and publication media warning behavior remains honest.

### 12.3 Identity, lifecycle, cleanup, and public delivery

- `ProductMedia.id` is the native media identity exposed to public/admin contracts. `asset_key` is server-internal.
- Gallery order is zero-based and complete. One active gallery item may be featured; one active Product-wide card override may exist.
- Retirement preserves ProductMedia and external mapping provenance, removes it from public projection, and resolves featured/card fallback atomically.
- Unattached completed uploads expire and become cleanup-eligible after 24 hours. Physical deletion occurs only when no active/retired ProductMedia or mapping references the native asset and provider deletion is confirmed/retriable.
- Public `url` is an HTTPS delivery URL resolved from the native key. Provider name, key, upload URL, checksum, and source URL are never exposed publicly.

## 13. Admin Product Information Architecture

### 13.1 Navigation and sections

The authenticated admin navigation contains **Products** and **Collections**. Product list opens `/products/{id}` with these sections:

1. **Overview** — title, handle action, descriptions, classification, search terms, native ID/status/version summary.
2. **Options & Variants** — topology, combinations, SKU, ordering, price, inventory policy, lifecycle.
3. **Media** — upload, gallery order, alt, featured, optional activated card override, retired items.
4. **Specifications** — controlled/free and single/multiple inputs driven by Definitions.
5. **Publication** — eligibility, warnings, preview, publish/unpublish/retire/restore history summary.
6. **Collections** — assignments and links to Collection ordering workspace.
7. **Inventory** — read-only current positions/sellability; adjustment workflows are outside Product editor.
8. **Migration** — authorized source mappings, hashes, latest run/outcome/conflicts; never editable source IDs.

### 13.2 Save, validation, conflict, and loading behavior

- Each section uses its explicit command and owns an independent dirty state. There is no global arbitrary Product save.
- Leaving a dirty section or closing the browser invokes a standard unsaved-change warning. Successful save replaces local data/version/ETag with the response.
- Client validation provides immediate shape feedback; API violations are authoritative and map by JSON path. Aggregate issues remain in a persistent summary linked to sections.
- A 412 opens a conflict panel showing safe current summary and offers **Reload and discard local changes** or **Copy local values, reload, and reapply manually**. There is no automatic merge/retry.
- Loading preserves the existing screen where safe and disables only conflicting mutations. Initial load has skeleton; 404 has a Product-not-found state; retryable problems expose explicit retry.
- Publish always runs validation first, displays all blockers/warnings, requires confirmation, then sends the versioned publish command. Preview never creates a public draft URL.

### 13.3 Permission behavior

- Read-only users see no enabled mutation controls.
- `catalog:write` edits content/topology/specifications but cannot publish or upload media without the additional scope.
- `catalog:publish` owns lifecycle actions.
- `catalog:media` owns upload/attachment/role/retirement.
- `merchandising:write` owns membership/order.
- `migration:read|run` gates provenance and run controls.
- `catalog:repair` gates projection operations, which are absent from ordinary editor actions.

## 14. Test and Acceptance Specification

### 14.1 Layered tests

| Layer | Mandatory scenarios |
| --- | --- |
| Domain unit | Handle transition postcondition; complete/unique combinations; lifecycle; Money precision/currency; publication; attribute modes/cardinality; media roles; sellability. |
| Application unit | Permissions; If-Match; idempotency; command orchestration; field-group ownership; cancellation; provider-after-commit; stable problem mapping. |
| PostgreSQL integration | Real PostgreSQL 18 constraints/indexes; migrations when later created; row locks; concurrent handles/Product/Collection edits; projection/outbox atomicity. |
| HTTP integration | Every OpenAPI operation, content type, status, headers, nullability, unknown-property rejection, authorization, problem extensions. |
| Search/query | Complete catalog, no unpublished leakage, AND/OR filters, self-excluding facets, exact totals, deterministic sorts, signed cursor/query mismatch, sitemap delta. |
| OpenAPI | 3.1 syntax/reference validation; operation IDs unique; examples validate; generated client compilation when repositories later exist. |
| Import | Four modes, dry-run/reconcile immutability, mapping resolution, identical reruns, source/native conflicts, partial failure, cancellation, truncation detection, no source writes. |
| Media | MIME/size/checksum, upload idempotency, attach once, safe copy, SSRF controls, featured/card uniqueness, retirement/fallback, cleanup. |
| E2E | Admin create -> topology/media/specs -> validate/publish -> public list/detail/collection/sitemap; stale conflict; sold-out visibility; alias redirect. |

### 14.2 Six Phase 2A review proofs

1. **Canonical handle:** parallel changes/creates prove no commit with zero/two active canonical handles and preserve the prior alias.
2. **Attribute mode/cardinality:** table-driven unit plus PostgreSQL command tests cover text/controlled and single/multiple, retired/foreign allowed values, and Definition-in-use edits.
3. **Media validation gate:** import fixture/mapping freeze is rejected until the runbook classifies production image differences and source completeness.
4. **Atomicity:** injected failures after canonical write, projection write, and outbox insert each leave all pre-command state unchanged.
5. **Production validation:** mapping/fixture approval checks a signed validation manifest and refuses missing/failed gate evidence.
6. **Repository gate:** documentation acceptance asserts no target repositories/code/migrations exist and reports initialization as separately authorized only after validation/sign-off.

### 14.3 Contract acceptance checklist

- OpenAPI validates as 3.1 and matches every endpoint in this document.
- Every operation declares audience, permission, request, response, status/problem codes, idempotency, concurrency, transaction ownership, and sync/async behavior.
- Every Phase 2A invariant maps to enforcement and at least one test layer.
- Public schemas contain no source IDs/mappings, versions, quantities, policies, raw JSON, asset keys, or database concepts.
- All list orders are deterministic and every unbounded public scan uses a cursor.
- Source mapping facts marked validation-required remain conditional rather than fabricated.

## 15. Production Validation Dependency

The exact procedure, outputs, thresholds, exception categories, and sign-off are in `PHASE-2B-READ-ONLY-PRODUCTION-VALIDATION-RUNBOOK.md`. Until that gate passes:

- production attribute modes/cardinalities/vocabularies are not frozen;
- SKU/currency/classification/audience mappings are not frozen;
- Product/Variant/media/import fixtures are not frozen;
- BMS card overrides are not activated for import;
- Collection membership normalization and Inventory initialization are not frozen; and
- neither target repository, physical migrations, nor implementation is authorized.

These are production-validation dependencies, not unanswered Phase 1 business decisions.

## 16. Final Phase 2B Decisions and Delivery Gate

### 16.1 Decisions made

1. `/api/v1` path versioning, direct success resources, and RFC 9457 problem responses.
2. One public Product list contract owns listing/search/filter/facets/sort and exact counts.
3. Signed query-bound keyset cursors; default 24 and maximum 100.
4. Stable merchandising order replaces the false best-seller claim; search relevance is deterministic.
5. Exact separate public/admin DTOs; no source/internal data in public responses.
6. Scoped admin commands rather than generic Product saves.
7. `ETag`/`If-Match`, 428/412 behavior, and no stale-write automatic retry.
8. Idempotency keys for non-idempotent create/action/async boundaries.
9. Explicit invariant ownership across transport/application/domain/database/tests.
10. Synchronous canonical/projection/outbox Product transaction and versioned bounded events.
11. Four deterministic import modes, per-aggregate transactions, field-group hashes, native-edit conflict protection, cooperative cancellation, and identical-rerun behavior.
12. Provider-neutral upload/copy contract with native ProductMedia identity, byte checksum, verification, retirement, and cleanup.
13. Section-based admin workflow, deliberate publication, conflict UX, and permission boundaries.
14. Measurable read-only production validation before repository/schema/import freeze.

### 16.2 Deferred decisions

Hosting, storage/CDN provider, exact job package, admin identity implementation, optional external search, richer SEO/promotions, variant media, placement-specific card media, advanced Collection rules, full Inventory movements/reservations, and later commerce domains remain deferred within their Phase 2A boundaries.

### 16.3 Readiness answers

- **Is the Product/Catalog HTTP/command contract implementation-ready?** Yes, subject to Phase 2B review and OpenAPI/document consistency.
- **Can production mappings/fixtures be frozen?** No; the read-only validation gate must pass.
- **Can `vastriqo-api` be initialized now?** No. The latest approved gate requires validation sign-off plus explicit repository-initialization authorization.
- **Can `vastriqo-admin` be initialized now?** No. It additionally depends on acceptance of this OpenAPI/admin-session boundary and the same explicit authorization.
- **What follows?** Execute the separately authorized read-only validation runbook, resolve/quarantine every gate exception, approve Phase 2B, then authorize repository initialization as a distinct phase.

## 17. Non-Implementation Confirmation

This phase creates documentation/contracts only. It does not create application repositories, application code, migrations, packages, databases, provider configuration, production queries, imports, or external writes, and it does not modify `vastriqo`, `bms-api`, or `bms-admin`.
