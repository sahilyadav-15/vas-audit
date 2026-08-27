# Vastriqo Phase 2B — Read-Only Product/Catalog Production Validation Runbook

Date: 2026-08-26 (Asia/Kolkata)  
Status: **Procedure defined; not executed**  
Gate: required before mapping/fixture freeze, repository initialization, physical migrations, or implementation based on production assumptions

## 1. Purpose and Safety Boundary

This runbook gathers the minimum authoritative Shopify/BMS evidence needed to freeze Product/Catalog mappings without changing either source.

Execution requires separate authorization and an identified operator. This Phase 2B documentation task does not execute it.

Mandatory controls:

- Shopify access is read-query/export only. Do not call mutations, sync endpoints, webhooks, bulk-operation mutations, or write-capable REST methods.
- BMS uses a dedicated `SELECT`-only account and a read-only consistent transaction or approved replica.
- Do not use `bms-api` sync/admin mutation routes as validation sources.
- Do not create target data, mappings, fixtures, media copies, or imports.
- Never record credentials, access tokens, cookies, signed media query strings, customer/order data, or unbounded raw responses.
- Stop on unexpected scope, authorization, source-version, pagination, or query inconsistency; record `GATE_EXECUTION_INVALID` rather than improvising.

## 2. Roles, Execution Identity, and Sign-Off

| Role | Responsibility |
| --- | --- |
| Validation operator | Runs approved read queries, records manifests/hashes, never changes sources. |
| Architecture/API owner | Confirms evidence answers schema/contract questions and exceptions are bounded. |
| Migration owner | Authors mapping version and proves deterministic normalization/rerun rules. |
| Vastriqo product/operations owner | Confirms classifications, collections, media overrides, audience, and sellability meaning. |

The operator records their identity, access-ticket/authorization reference, source environments, Shopify shop identifier, BMS database identifier, API version, start/end UTC timestamps, and tool versions. Secrets are represented only as `[REDACTED]` and never hashed into evidence.

All three approving owners sign the final gate summary. One person may fill multiple roles only if the project owner explicitly approves that responsibility combination.

## 3. Evidence Package Layout

Retain the following outside application repositories in an access-controlled evidence location; only the redacted summary may later be committed:

```text
phase-2b-validation/<run-id>/
  manifest.json
  queries/
    shopify-products.graphql
    shopify-product-children.graphql
    shopify-locations.graphql
    bms-product-catalog.sql
  summaries/
    source-counts.json
    product-shapes.csv
    option-variant-exceptions.csv
    sku-currency-price-exceptions.csv
    handle-exceptions.csv
    classification-audience-vocabulary.csv
    attribute-shapes.csv
    media-differences.csv
    collection-membership-exceptions.csv
    inventory-mapping-exceptions.csv
    source-identity-conflicts.csv
    mapping-proposal.json
  gate-summary.md
  sha256sums.txt
```

`manifest.json` contains run UUID, timestamps, environment labels, source/API versions, query-file hashes, page counts, final cursors represented as hashes, record counts, mapping candidate version, and redaction policy version. It contains no source rows or credentials.

Every retained file receives SHA-256 in `sha256sums.txt`. Hashes are calculated after redaction.

## 4. Snapshot and Pagination Protocol

### 4.1 Shopify

1. Pin one supported Shopify Admin GraphQL API version already approved for the shop. Record it; do not silently upgrade between pages.
2. Capture validation start UTC and a source upper-bound timestamp if the API supports it. If a consistent export cannot be obtained, record Product `updatedAt` and rerun any record changed during extraction.
3. For every connection, start with `after: null`, persist only a hash of each returned cursor, and continue until `hasNextPage = false`.
4. Record page number, node count, first/last opaque ID hashes, cursor hash, and query-response UTC. Do not retain access headers.
5. Product child connections must also reach `hasNextPage = false`; a complete Product page with truncated Variants/media/metafields is a failed gate.
6. At completion, re-read counts/update watermarks. If material changes occurred during extraction, repeat affected pages or execute an approved consistent export.

No Shopify GraphQL mutation is permitted even if Shopify names a read-oriented bulk operation as a mutation. Use paginated queries or a merchant-generated read-only export.

### 4.2 BMS

1. Verify the database principal has only `SELECT` and cannot write.
2. Use an approved replica or a repeatable-read, read-only transaction:

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION READ ONLY;
-- approved SELECT statements only
COMMIT;
```

3. Record database UTC/timezone, schema name, server version, transaction start/end, and query hashes.
4. If the database cannot guarantee a consistent read, stop and obtain a read-only snapshot/replica; do not infer consistency.

## 5. Shopify Read Queries and Required Data

The operator must validate field availability against the pinned API version before execution. Removing an unavailable field requires an explicit gate exception; silently omitting it is prohibited.

### 5.1 Complete Product identity/content page

```graphql
query VastriqoValidationProducts($after: String) {
  products(first: 100, after: $after, sortKey: ID) {
    pageInfo { hasNextPage endCursor }
    nodes {
      id
      handle
      title
      description
      descriptionHtml
      vendor
      productType
      tags
      status
      createdAt
      updatedAt
      publishedAt
      totalInventory
      seo { title description }
      featuredImage { id url altText width height }
      options { id name position values }
    }
  }
}
```

Required outputs: total/unique Product IDs; status/published counts; handle/title/content completeness; distinct vendor/productType/tag values; update range; option names/order/values; SEO population; featured-media identity.

### 5.2 Complete per-Product children

Run every connection to completion for every Product. If the pinned API supports a complete top-level ProductVariant connection, it may replace per-Product Variant paging only when Product ownership is included and counts reconcile.

```graphql
query VastriqoValidationProductChildren(
  $id: ID!
  $variantAfter: String
  $mediaAfter: String
  $metafieldAfter: String
) {
  product(id: $id) {
    id
    variants(first: 100, after: $variantAfter) {
      pageInfo { hasNextPage endCursor }
      nodes {
        id
        title
        sku
        barcode
        price
        compareAtPrice
        availableForSale
        inventoryQuantity
        selectedOptions { name value }
        inventoryPolicy
        updatedAt
        image { id url altText width height }
        inventoryItem { id tracked }
      }
    }
    media(first: 100, after: $mediaAfter) {
      pageInfo { hasNextPage endCursor }
      nodes {
        id
        mediaContentType
        status
        preview { image { id url altText width height } }
      }
    }
    metafields(first: 100, after: $metafieldAfter) {
      pageInfo { hasNextPage endCursor }
      nodes {
        id
        namespace
        key
        type
        value
        updatedAt
        reference {
          ... on Metaobject { id type handle updatedAt fields { key type value } }
        }
        references(first: 100) {
          pageInfo { hasNextPage endCursor }
          nodes {
            ... on Metaobject { id type handle updatedAt fields { key type value } }
          }
        }
      }
    }
  }
}
```

If a metafield `references` connection itself paginates, execute a dedicated read query for that metafield until complete. Required outputs include exact Variant/media/metafield counts, Variant selections, SKU/price/policy/tracking, media order/alt/status, and every configured metafield/reference shape.

### 5.3 Locations and inventory levels

```graphql
query VastriqoValidationLocations($after: String) {
  locations(first: 100, after: $after, includeInactive: true) {
    pageInfo { hasNextPage endCursor }
    nodes { id name isActive fulfillsOnlineOrders address { countryCodeV2 } }
  }
}
```

Use the pinned API's read-only InventoryItem/InventoryLevel query to extract every relevant InventoryItem by Variant, every Location, tracked state, named available/on-hand quantities exposed by that API version, and source update token. The exact quantity field names must be recorded in `mapping-proposal.json`; never equate different Shopify quantity names without evidence.

### 5.4 Source checks produced from Shopify extraction

The sanitized summarizer must compute:

- total Products, published Products, Variants, media, and each child maximum;
- duplicate Product/Variant/media IDs;
- blank/invalid/duplicate normalized handles;
- Products with no Variants; active/published Products with no valid Variant;
- option names/order/value normalization and per-Variant completeness;
- duplicate normalized Variant signatures per Product;
- blank/duplicate case-normalized SKU and BMS style-number candidates;
- distinct currencies, amount precision, zero/negative/invalid/missing price, mixed-currency Products;
- publication/status/timestamp combinations;
- classification/productType/vendor/tag/audience vocabularies and counts;
- media count/order/featured mismatches, missing alt, non-image/failed media, duplicate URLs after signed query-string removal;
- all metafield `(namespace,key,type)` combinations, scalar/list/reference cardinalities, missing references, and readable value shapes;
- Color/Size from Variants versus product metafields/options;
- locations, tracked/untracked Variants, policies, negative/zero quantities, and multi-location use; and
- source `updatedAt` coverage usable as delta tokens.

## 6. BMS Read Queries and Required Data

The following statements are templates against the inspected BMS schema. The operator must first confirm column existence/types read-only and record any drift.

### 6.1 Counts, identities, and mirror freshness

```sql
SELECT COUNT(*) AS product_count,
       COUNT(DISTINCT shopify_product_id) AS unique_shopify_product_count,
       MIN(shopify_updated_at) AS min_shopify_updated_at,
       MAX(shopify_updated_at) AS max_shopify_updated_at,
       MIN(synced_at) AS min_synced_at,
       MAX(synced_at) AS max_synced_at
FROM shopify_products;

SELECT shopify_product_id, COUNT(*) AS duplicate_count
FROM shopify_products
GROUP BY shopify_product_id
HAVING COUNT(*) > 1;

SELECT id, shopify_product_id, title, price, url, image_url, gender,
       seo_title, seo_description, shopify_created_at,
       shopify_updated_at, synced_at, created_at, updated_at,
       SHA2(COALESCE(CAST(meta AS CHAR), ''), 256) AS meta_sha256
FROM shopify_products
ORDER BY shopify_product_id, id;
```

The retained summary uses IDs/hashes and classified differences; raw `meta` is processed in the controlled workspace and is not committed.

### 6.2 Collection and membership inventory

```sql
SELECT id, title, slug, description, status, sort_order, created_at, updated_at
FROM shopify_categories
ORDER BY sort_order, id;

SELECT shopify_category_id, shopify_product_id, sort_order, id, created_at
FROM shopify_category_products
ORDER BY shopify_category_id, sort_order, id;

SELECT scp.shopify_category_id, sc.slug, scp.shopify_product_id, scp.sort_order, scp.id
FROM shopify_category_products scp
LEFT JOIN shopify_categories sc ON sc.id = scp.shopify_category_id
WHERE sc.id IS NULL;

SELECT scp.shopify_category_id, scp.shopify_product_id, scp.sort_order, scp.id
FROM shopify_category_products scp
LEFT JOIN shopify_products sp ON sp.shopify_product_id = scp.shopify_product_id
WHERE sp.id IS NULL;

SELECT shopify_category_id, shopify_product_id, COUNT(*) AS duplicate_count
FROM shopify_category_products
GROUP BY shopify_category_id, shopify_product_id
HAVING COUNT(*) > 1;

SELECT shopify_category_id, sort_order, COUNT(*) AS position_count
FROM shopify_category_products
GROUP BY shopify_category_id, sort_order
HAVING COUNT(*) > 1;
```

Required outputs: active/inactive Collections, normalized slug collisions, expected current placement slugs, order gaps/duplicates, member counts, orphans, stale/unpublished Shopify correlations, and related-product ordering inputs.

### 6.3 Product-wide image behavior

```sql
SELECT id, shopify_product_id, title, image_url,
       COALESCE(
         JSON_UNQUOTE(JSON_EXTRACT(meta, '$.featured_image.url')),
         JSON_UNQUOTE(JSON_EXTRACT(meta, '$.featured_image.src')),
         JSON_UNQUOTE(JSON_EXTRACT(meta, '$.featuredImage.url'))
       ) AS mirrored_featured_url,
       updated_at, synced_at
FROM shopify_products
WHERE COALESCE(image_url, '') <>
      COALESCE(
        JSON_UNQUOTE(JSON_EXTRACT(meta, '$.featured_image.url')),
        JSON_UNQUOTE(JSON_EXTRACT(meta, '$.featured_image.src')),
        JSON_UNQUOTE(JSON_EXTRACT(meta, '$.featuredImage.url')),
        ''
      )
ORDER BY shopify_product_id;
```

Compare canonicalized URL scheme/host/path after removing query strings, but retain neither signed query parameters nor full URLs in committed evidence. Each difference must be classified as deliberate current card override, stale/accidental, same asset transformation, missing source media, or unresolved.

### 6.4 BMS-to-Shopify correlation

Join every BMS `shopify_product_id` to the complete Shopify extraction after normalizing numeric ID versus GID. Report:

- missing Shopify Product;
- duplicate/conflicting normalized identity;
- title/handle/price/currency/publication/media/Variant count mismatch;
- BMS mirror newer than extracted Shopify upper bound;
- stale BMS record or membership; and
- BMS `gender`, meta classification, tags, SKU/style number evidence and inconsistencies.

Do not choose BMS title/price/meta as Product authority merely because it differs. Shopify remains the preferred Product source; BMS supplies curation/reconciliation evidence.

## 7. Normalization and Mapping Proposal Output

`mapping-proposal.json` must be versioned and deterministic. Each mapping entry includes:

```text
sourceSystem
sourceEntityType
sourcePath
normalizationAlgorithm
validationRule
nativeTarget/table concept
fieldGroup
conflictBehavior
excludedBehavior
evidenceSummaryId
validationState: approved | quarantined | unresolved
```

Required mappings:

- Shopify Product/Variant/Media identity to typed external mapping;
- handle normalization/collisions/aliases;
- title/descriptions and sanitization outcome;
- Product lifecycle and merchandising timestamp;
- Product type to native classification;
- approved tags/vendor/style terms to search terms or explicit exclusion;
- Options/Values/Selections and signatures;
- SKU and price/currency exact conversion;
- inventory mode/policy, Location and Position initialization;
- gallery/featured media and any validated BMS card override;
- each approved AttributeDefinition mode/cardinality/value normalization;
- BMS Collection/ordered membership through Shopify Product mapping; and
- source versions/watermarks/hashes.

Unknown production facts remain `unresolved` and block only their affected freeze. No entry may fall back to raw Product JSON or a generic metafield copy.

## 8. Exception Categories and Required Disposition

| Category | Examples | Required disposition |
| --- | --- | --- |
| `EXTRACTION_INCOMPLETE` | Pagination not exhausted, child cap, mid-run drift | Rerun/obtain consistent export; cannot quarantine extraction itself. |
| `IDENTITY_CONFLICT` | Duplicate Product/Variant IDs, BMS correlation ambiguity | Resolve exact authority or quarantine record; zero unresolved for apply. |
| `HANDLE_CONFLICT` | Blank/invalid/normalized duplicate | Assign deterministic approved native handle/alias plan or quarantine. |
| `VARIANT_TOPOLOGY_INVALID` | Missing Option, duplicate signature, foreign value | Correct mapping with evidence or quarantine Product. |
| `PRICE_CURRENCY_INVALID` | Missing/zero published price, precision, mixed currency | Correct source/map deliberately or quarantine Product. |
| `ATTRIBUTE_UNKNOWN` | Unknown type/reference/cardinality/value | Approve native Definition mapping or quarantine affected value/Product. |
| `MEDIA_UNAVAILABLE` | Failed URL, invalid type, no copy permission | Quarantine media; Product may remain publishable with warning only if other rules pass. |
| `CARD_OVERRIDE_UNRESOLVED` | BMS/Shopify image difference without intent | Do not import override until classified. |
| `COLLECTION_ORPHAN_ORDER` | Missing Product, duplicate/gapped position | Resolve mapping and deterministic normalized order or exclude with owner sign-off. |
| `INVENTORY_AUTHORITY_UNKNOWN` | Multiple locations/quantities/policy ambiguity | Resolve source-of-truth before initialization/cutover. |
| `SOURCE_DRIFT` | Record changes during extraction | Re-read affected record/page and update manifest. |

Quarantine means the record/value is excluded from apply with a stable reason and remains in reconciliation totals. It never means silently dropping it.

## 9. Measurable Acceptance Thresholds

The gate passes only when:

1. Every source and nested connection reaches `hasNextPage = false`; extracted counts reconcile to recorded source counts.
2. Product/Variant/Media source IDs are unique or every conflict is quarantined with owner approval.
3. There are zero unresolved normalized handle collisions.
4. There are zero unquarantined active Variant missing selections or duplicate signatures.
5. There are zero unquarantined published Products without an active valid positive-price Variant.
6. Every publishable Product uses exactly one validated currency and every amount converts losslessly to minor units.
7. Every required displayed Attribute has a confirmed source path, native Definition, mode, cardinality, normalization, and unknown-value policy.
8. Every BMS/Shopify image difference is classified; only confirmed deliberate product-wide overrides are enabled.
9. Every included Collection membership resolves to exactly one Product mapping; positions have one deterministic normalized order; no orphan is silently included.
10. Every tracked Variant has a validated inventory policy, authoritative Location mapping, and quantity snapshot rule. Untracked/continue behavior is explicitly evidenced.
11. Every source watermark/version token used for delta is populated and monotonic enough for the documented use, or delta remains disabled.
12. `mapping-proposal.json`, exception files, manifest, and all hashes are complete and signed off.

Counts may be nonzero for warnings such as missing media/alt text, but each must be enumerated and its publication/import behavior must match the Phase 2B contract. There is no percentage-based allowance for unresolved identity, handle, topology, price/currency, or inventory-authority failures.

## 10. Redaction and Retention

- Remove authorization headers, tokens, cookies, shop secrets, connection strings, user/contact/order data, and URL query strings before hashing/retention.
- Replace external IDs in broadly shared summaries with stable salted hashes; the restricted mapping package may retain exact Product/Catalog source IDs.
- Do not retain complete rich HTML/raw metafield JSON when a shape/hash/short redacted sample proves the point.
- Limit diagnostic samples to the minimum necessary and mark their access classification.
- Retention duration/location requires project security approval before execution; this runbook does not invent it.

## 11. Gate Summary and Sign-Off Template

`gate-summary.md` must contain:

```text
Validation run ID:
Environment/source versions:
Started/completed UTC:
Manifest SHA-256:
Mapping proposal version/SHA-256:

Extraction complete: PASS|FAIL
Identity/handle gate: PASS|FAIL
Variant topology gate: PASS|FAIL
Price/currency gate: PASS|FAIL
Attribute gate: PASS|FAIL
Media/card-override gate: PASS|FAIL
Collection gate: PASS|FAIL
Inventory gate: PASS|FAIL
Delta-token gate: PASS|DISABLED|FAIL

Quarantined Product count:
Quarantined Variant count:
Quarantined membership/media/value counts:
Open exceptions (must be zero for affected freeze):

Architecture/API owner: name/date/decision
Migration owner: name/date/decision
Product/operations owner: name/date/decision

Final gate: PASS|FAIL
Authorized next action: mapping freeze only | fixture freeze | repository initialization consideration | none
```

A `PASS` validates evidence and permits the specifically signed next action; it does not itself initialize repositories or authorize production writes/imports.

## 12. Current Gate Status

**NOT EXECUTED / NOT PASSED.** No production source was queried by Phase 2B. Mapping/fixture freeze, repository initialization, physical migrations, and implementation remain blocked pending separately authorized execution and sign-off.
