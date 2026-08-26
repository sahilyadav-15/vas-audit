# Phase 1 — Vastriqo Functional Dependency Audit

Audit date: 2026-08-24 (Asia/Kolkata)  
Scope: the complete local `vastriqo` application, with relevant call paths verified in `bms-api`  
Method: static source trace only; no live Shopify, BMS, database, or deployed-environment access

Status note (2026-08-26): **Audit complete.** The questions in §14 were discovery inputs and have now been reclassified or bounded by `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`. This audit remains authoritative for current source behavior, not for the latest phase gate.

## 1. Executive Summary

### Status legend

- **Verified** — directly established from the audited source.
- **Inferred** — strongly suggested by source relationships but not directly established.
- **Undetermined** — requires runtime data, business input, or external-system evidence.

### Main findings

1. **Verified:** Vastriqo does not currently have one commerce backend. Direct Shopify Storefront GraphQL owns full-catalog listing, product detail, variants, prices, cart, buyer attachment, and checkout redirection. BMS owns custom merchandising and auxiliary records, while also calling Shopify Admin/OAuth APIs for identity, addresses, orders, customer details, and product mirroring.
2. **Verified:** the future platform must replace more than `vastriqo/lib/shopify/index.ts`. It must also replace Shopify behavior hidden behind Vastriqo route handlers and `bms-api`, including customer OAuth/token exchange, live address CRUD, live order history, checkout identity, and the upstream source of the BMS product mirror.
3. **Verified:** the BMS `shopify_products` mirror is not an independent catalog. It stores a small set of columns plus Shopify-shaped JSON, is refreshed one-way from Shopify, and lacks an independent publication, inventory-reservation, cart, checkout, retail-order, payment, or fulfillment system. It is useful as a migration/reconciliation source, not as the target model.
4. **Verified:** the catalog screens have two materially different product shapes. `/products`, `/collections`, and `/search` receive up to 100 rich Shopify products. Home and gender collections receive BMS-mirrored cards containing mainly ID, handle, title, one image, minimum price, currency, and color/size values. Product detail always returns to Shopify by product handle or ID.
5. **Verified:** search, filtering, and sorting are browser-side operations over the already-fetched list. “Recommended/Best Sellers” performs no calculated sort; it preserves upstream order. This is not a server-side catalog/search capability.
6. **Verified:** the current checkout requires a signed-in Shopify-backed customer. Shopify owns the cart and hosted checkout/payment boundary. BMS validates local name/phone data and returns a stored Shopify customer token; Vastriqo attaches that token to the Shopify cart and redirects to Shopify's `checkoutUrl`.
7. **Verified:** orders and addresses are not stored in BMS as Vastriqo commerce entities. They are queried or mutated live through Shopify Admin GraphQL. BMS `orders`/`order_items` are CCA production models and must not be copied as Vastriqo retail models.
8. **Verified:** current order “tracking” does not open tracking information; it only scrolls the existing order card. “Cancel Order” redirects to the contact page and performs no cancellation. Return/exchange, shipping-fee, COD, notification, and refund statements on content pages are not implemented commerce workflows in the storefront source.
9. **Verified:** current menus, footer, home/editorial copy, FAQ, story, policies, size charts, and shipping/returns content are source-controlled. BMS storefront-page/menu modules exist but have no Vastriqo call sites.
10. **Verified:** no application or physical schema design follows from this audit. The future responsibilities and conceptual data below are requirements derived from current behavior, not approved endpoint/table designs.

## 2. Current Vastriqo Runtime Dependency Map

```text
Browser
  |
  |-- local/source-controlled presentation
  |     header, footer, home/editorial content, FAQ, policies, size charts,
  |     navigation, filters/sorts, SEO composition, account/cart UI
  |
  |-- Vastriqo server components / server actions
  |     `lib/shopify/index.ts`
  |          `-- Shopify Storefront GraphQL
  |                product list/detail, variants, media, metafields,
  |                cart create/read/update/remove
  |
  |-- Vastriqo route handlers
  |     |-- `/api/cart/checkout` ----------> BMS checkout validation/token
  |     |                                  + Shopify cartBuyerIdentityUpdate
  |     |-- `/api/auth/*` ----------------> BMS customer bridge
  |     |                                      `-- Shopify OAuth/Admin GraphQL
  |     |-- `/api/wishlist/*` ------------> BMS `wishlists`
  |     |-- `/api/contact/submit` --------> BMS `inquiries` (+ optional S3)
  |     |-- `/api/newsletter/subscribe` --> BMS `newsletter_subscribers`
  |     `-- related-products proxy ------> BMS collections/product mirror
  |
  `-- external browser services
        Shopify Customer Account OAuth, Shopify hosted checkout,
        Google Analytics, Meta Pixel, Google Fonts, WhatsApp/mail/tel/social links
```

### Current sources of truth

| Domain | Current authority | Local/BMS role |
| --- | --- | --- |
| Product/variant/price/availability | Shopify | BMS keeps a partial mirror for custom collections and related products. |
| Collection merchandising | BMS custom categories and ordered membership | Membership points to Shopify product IDs. |
| Customer identity | Shopify Customer Accounts | BMS creates a local `customers` bridge and 24-hour app JWT. |
| Profile | Split | Normal profile edits are BMS-local; checkout detail edits write Shopify then BMS. |
| Addresses | Shopify | BMS is a live Admin GraphQL proxy; there is no local address table. |
| Cart/checkout/payment | Shopify | Browser keeps only a Shopify cart ID; BMS supplies customer validity/token. |
| Retail orders/fulfillment | Shopify | BMS queries live Shopify orders; no local Vastriqo order copy exists. |
| Wishlist/inquiry/newsletter | BMS | Records are Shopify-ID/customer-bridge coupled where applicable. |
| Storefront content/navigation | Vastriqo source | BMS page/menu modules are disconnected from the current renderer. |

## 3. Feature-by-Feature Dependency Matrix

| Feature | Implementation and call path | Data actually used / business behavior | Future responsibility and conceptual data | Disposition |
| --- | --- | --- | --- | --- |
| Global shell, header, announcement, footer | `app/layout.tsx`; `components/common/site-header/index.tsx`; `site-footer/index.tsx`; `content/site-content.json` | Local labels, links, contact/social data, announcement copy; header validates `front_token` through `/api/auth/me` and shows live Shopify cart count. | Storefront may keep source content; API needs session check and cart summary. Persist navigation/content only if admin management is approved. | Required behavior; content storage decision open. |
| Home | `app/page.tsx` -> `components/home/home-page/index.tsx`; featured cards call `getShopifyApiCollectionProducts("home-page")` | Hero, category tiles, trust copy, banners are local. Featured cards use BMS ordered membership, product ID/handle/title/image/minimum price/currency/color/size. Newsletter is separate. | Catalog/merchandising capability for an ordered home collection; product and media records; optional content model. | Required; BMS collection data must transform. |
| All products | `/products` -> `getProducts()` -> Shopify `products(first: 100)` | Product cards plus browser filters/sorts. Only the first 100 returned products participate. | Paginated published catalog query with stable sorting/filtering and card projection. | Required for independence. |
| Collections landing | `/collections` -> `CollectionsGridSection` -> `getProducts()` | Despite its name, it renders the same all-product grid; it does not list collection records. | Decide whether to preserve this alias or make it a true collection landing page. | Current behavior verified; target meaning needs decision. |
| Men/Women/Kids | route -> `CategoryCollectionPage` -> BMS `/admin/shopify-apis/collections/{slug}` | Fixed slugs (`mens-collection`, `womens-collection`, `kids-collection`); ordered BMS membership. Failure returns an empty grid with no Shopify fallback. Browser gender heuristic additionally checks title because mapped BMS cards omit most gender fields. | Active collection lookup and ordered product membership using Vastriqo IDs; explicit audience/category attributes if required. | Required; transform, do not copy Shopify IDs as primary keys. |
| Search | `/search?q=` -> `getProducts()` -> `CollectionsProductsClient` | Browser substring match over title, description, vendor, product type, tags, color, size within first 100 products. | Server-capable full-catalog search or an explicitly accepted client strategy; searchable field requirements are verified. | Required; current implementation must be replaced for complete catalog. |
| Filtering | `products-client.tsx` | Category=`productType`; size/color=`options`; price=minimum variant price; audience heuristic uses text/tags/options. | Filterable product type/category, option values, price and audience attributes. | Required behavior; current heuristics should be reviewed. |
| Sorting | `products-client.tsx` | Price and `createdAt` sort locally. “Recommended/Best Sellers” returns `0`, preserving input order; no sales/popularity metric exists. BMS cards omit `createdAt`. | Stable default merchandising order; price/new-arrival sorts; “best seller” only if a real rule is approved. | Price/new-arrival required; best-seller meaning undetermined. |
| Product cards | `ProductCard` in `products-client.tsx` | ID/handle, title, featured image/alt, min price/currency, product type badge, color/size, optional first variant ID, wishlist snapshot, analytics fields. | Catalog card projection and canonical product URL. | Required. |
| Product detail | `/[product-slug]` -> `getProductByHandle()`; legacy `/details?id=` -> `getProduct()` then redirect | Shopify product title, descriptions, images/media, variants/options, variant price, metafields; related products come from BMS. | Product-detail aggregate with product, media, variants, attributes/specifications, price and availability. | Required. |
| Variants/options | Detail derives option names/values from every variant's `selectedOptions`, selects a matching variant, and sends its Shopify variant GID to cart | Variant ID, selected option pairs, variant price and nominal availability. SKU is queried but not rendered. | Stable Vastriqo variant identity, option/value relationships, variant price and sellability. | Required; transform. |
| Images/media | Lists use featured image; detail deduplicates `images`, `media.previewImage`, and featured image; cart uses product featured image; order/wishlist use snapshots | URL and alt text are displayed. Detail does not associate variant selection with variant image. | Ordered product media with alt text and optional variant association; durable object storage/CDN. | Required; migrate selected media, not every Shopify media field automatically. |
| Pricing | Cards use minimum variant price; detail uses selected variant price; cart uses line price and Shopify total; orders use money sets | Amount/currency; no compare-at price is displayed even though listing queries it. Shipping is “calculated at checkout.” | Authoritative variant prices, currencies, cart price snapshots/revalidation and order money snapshots. | Required. Compare-at/discount model needs decision. |
| Inventory/availability | Shopify queries `availableForSale`/`currentlyNotInStock`; UI helper treats any variant with an ID as purchasable because `quantityAvailable` is typed but not queried | Shopify cart/checkout is the effective enforcement point. No local stock/reservation logic exists. | Inventory source of truth, availability calculation, reservation/release and checkout revalidation. | Required for independence; current UI logic is insufficient. |
| Descriptions/specifications | `DetailsPage` renders `descriptionHtml` unsanitized or plain `description`; displays non-hidden metafields | Product narrative and readable specification values/metaobject labels. | Product descriptions plus structured product attributes/custom content. | Required for preserved detail experience. |
| Tags/product type/vendor | Shopify list query; `matchesQuery`, `matchesGender`, category filter, card badge and analytics | Tags/type/vendor affect discovery; vendor/type may appear on related cards. Vendor is not visibly rendered on detail. | Store only attributes required for discovery/admin/merchandising; exact brand/vendor role needs decision. | Product type/tags currently required; vendor target role needs decision. |
| Product SEO | `getProducts()` requests Shopify `seo`, but no list consumer uses it. Dynamic product metadata in `/[product-slug]` uses local templates plus product title/featured image, not Shopify SEO fields. | Canonical URL is handle-based; title/description are synthesized. | Product handle, title, image and SEO override capability if approved. | Shopify SEO fields are not proven migration requirements; SEO behavior is required. |
| Related products | Detail -> Vastriqo proxy -> BMS `/admin/shopify-apis/get-related-products` -> shared collection membership | Up to 8 shown (BMS returns 8–12), deduplicated products sharing any active custom category; consumes ID/title/handle/image/price/currency/type/vendor/color/size. | Related-product selection derived from Vastriqo collections or an approved rule. | Required if preserving current experience; transform BMS relationships. |
| Cart | `CartProvider` -> direct Shopify cart mutations in `lib/shopify/index.ts`; `/cart` renders lines | Cart ID, line ID, variant ID/title, product title/handle/featured image, unit price/currency, quantity, total, checkout URL. Add/update/remove supported. | Cart lifecycle, line operations, authoritative pricing/availability, cart totals. | Required. |
| Cart persistence | `localStorage.cartId`; provider reloads Shopify cart on mount | Anonymous device-local pointer to Shopify-hosted cart; invalid cart clears local state. No customer cart merge. | Persistent cart ownership/session strategy, expiry and optional login merge. | Required behavior; policy needs decision. |
| Checkout / Buy now | Cart/detail -> `/api/cart/checkout` -> BMS checkout validity/details -> Shopify `cartBuyerIdentityUpdate` -> Shopify checkout URL | Login required; first name, last name, 10-digit phone required; country fixed to `IN`; Shopify customer access token attaches buyer. Buy Now creates a separate Shopify cart. | Checkout orchestration, customer/contact requirements, addresses, shipping, tax, promotions, payment handoff, idempotent order creation. | Required; Shopify may remain temporary until replacements exist. |
| Payment | No Vastriqo payment integration; Shopify hosted checkout owns method collection and payment result | Storefront only displays “secure checkout.” Actual providers/methods are not established from source. | Payment-provider boundary, attempts/transactions, webhook/reconciliation and order state. | Required capability; provider/rules undetermined. |
| Login/signup/OAuth | `/login` and `/signup` both run `LoginPage`; browser PKCE -> Shopify authorize URL -> `/api/auth/callback` -> BMS token exchange/Admin customer fetch | OAuth code/verifier/state/nonce; BMS local customer and app JWT. Signup is not a distinct registration flow. | Vastriqo customer registration/login/session/recovery/verification; migration/account-link strategy. | Required; Shopify identity is temporary. |
| Session/logout | `front_token` and normalized `front_user` in localStorage; header calls `/api/auth/me`; logout calls missing BMS `/auth/logout`, suppresses errors, then clears local state | 24-hour BMS JWT; no verified active revocation/logout endpoint for this path. | Secure session/token storage, rotation/revocation and logout. | Required; recreate. |
| Profile | `/profile` -> `/api/auth/profile` -> BMS local `customers`; POST updates local name/email/phone only | Local customer ID, name, email, phone, note/timestamps. Profile can drift from Shopify. | Vastriqo-owned customer profile. | Required; migrate/transform selected customer fields. |
| Addresses | `/address` and `/addresses` -> `/api/auth/addresses` -> BMS -> Shopify Admin GraphQL | CRUD/default; first/last name, company (read only in current form), address lines, city, province, country, zip, phone. UI writes country `IN` implicitly. | Customer address book and default-address relationship. | Required; migrate later or recreate per approved customer migration. |
| Order history/details | `/orders` -> `/api/auth/orders?limit=10` -> BMS -> Shopify Admin GraphQL | Latest 10 only; order ID/name/date, financial/fulfillment status, totals/subtotal/shipping/tax, first transaction gateway, lines/title/variant/qty/price/image, shipping address. | Customer order list/detail, status, amount/address snapshots, payment and fulfillment summaries. | Required for new orders; historical migration needs decision. |
| Tracking | “Track Order” in `orders-page.tsx` scrolls its own order card. BMS query requests fulfillment/tracking data but the UI type/renderer ignores it. | No tracking URL/number is shown or opened. | Shipment/tracking capability only if approved as required behavior; current marketing copy promises it. | Functional gap; needs explicit requirement. |
| Cancellation/returns | “Cancel Order” redirects to `/contact-us?subject=...`; contact component does not read `subject`. Shipping/returns page is static copy. | No cancel, return, exchange, reverse pickup or refund mutation exists. | Decide workflows/states and admin/customer responsibilities. | Not implemented; requirements undetermined. |
| Wishlist | Product grid/detail login modal and `/wishlist`; Vastriqo proxies BMS `/front/wishlist/*` | Owner customer ID; Shopify product ID/handle; title/image snapshots; soft-delete toggle; pending item stored in localStorage through OAuth. | Wishlist keyed to Vastriqo customer/product identities; pending-login behavior. | Required current feature; transform BMS records. |
| Contact/inquiry submit | `/contact-us` -> `/api/contact/submit` -> BMS `/front/inquiry/submit` | Name, email, phone, constant `company_brand_name="Contact Support"`, message; optional auth. Current form has no file input. | Public/authenticated inquiry creation and admin handling. | Required current feature; BMS records are migratable after identity correction. |
| Inquiry history | `/inquiries` -> native `/front/inquiry/list`; route is absent from account navigation | ID, brand, requirements, status, created/updated. Current customer JWT ID is placed into `inquiries.user_id`, which belongs to BMS `users`, creating an identity mismatch. | If retained, customer-linked inquiry history with correct owner type. | Transitional/broken; needs decision and transformation. |
| Newsletter | Home form -> Vastriqo proxy -> BMS `/newsletter/subscribe` | Normalized email and source (`homepage`); duplicate subscription is success. | Subscription capture, consent/status/source and unsubscribe policy if required. | Required current feature; migrate BMS subscribers subject to consent rules. |
| Story/About/Why/FAQ/Size chart/Shipping/Privacy | Source-controlled route/component arrays and local/public assets; `/about-us` redirects to `/our-story` | Static copy, local FAQ search/category filter, measurement tables, external contact links. | Can remain storefront-local; CMS API/data only if admin editing is approved. | Required content experience; recreate/retain, not database migration by default. |
| Sitemap | XML `app/sitemap.ts` and human `/sitemap` call `getSitemapProducts()` -> `${NEXT_PUBLIC_API_BASE_URL}/shopify-products/sitemap` | Expects handle/title/updatedAt. No matching audited BMS route exists, so dynamic product sitemap has no verified runtime source. Static sitemap entries remain defined. | Published-product sitemap projection from the Vastriqo catalog. | Required SEO capability; current product path is disconnected. |
| Analytics | `app/layout.tsx`, `lib/analytics.ts`, auto/page trackers | GA4 page views and UI/commerce events with product IDs/names/prices, cart/order interactions and UTM/referrer data; Meta Pixel tracks PageView only. | Preserve or deliberately redefine analytics events/consent. No Vastriqo DB entity is proven necessary. | External/optional to core commerce, currently active. |
| Canonical host and assets | `proxy.ts`; `lib/assets.ts`; Next/public files | Apex GET/HEAD redirects to `www`; many local paths use checked-in assets, while `imageAsset()` defaults to the CCA S3 asset base. Google Fonts is external. | Stable domain/canonical policy and independent media/CDN ownership. | Required operational behavior; remove CCA asset coupling. |

## 4. Shopify Dependency Matrix

| Shopify dependency | Exact need and data | Evidence and use | Vastriqo-owned replacement | Status |
| --- | --- | --- | --- | --- |
| Storefront product list | First 100 products with broad product/variant fields | `getProducts()` in `lib/shopify/index.ts`; consumed by products, collections, search grids | Published catalog query and field projections from Vastriqo DB | Required final replacement. |
| Storefront product by handle/ID | Rich detail, variants, media and selected metafields/metaobjects | `getProductByHandle()` / `getProduct()` -> `DetailsPage` and product metadata | Product-detail aggregate | Required final replacement. |
| Storefront cart mutations | Cart ID/URL, lines, merchandise/price/product snapshot, totals | `createCart`, `getCart`, `addToCart`, `updateCart`, `removeFromCart` | Vastriqo cart service/data | Required final replacement. |
| Storefront buyer identity and checkout | `customerAccessToken`, country `IN`, checkout URL | `app/api/cart/checkout/route.ts` mutation `cartBuyerIdentityUpdate` | Vastriqo checkout/payment/order orchestration | Required; temporary Shopify boundary possible. |
| Customer Account OAuth | Authorization endpoint, PKCE code exchange and Shopify identity token | `customer-auth-client.ts`; BMS `loginCustomerByShopifyToken` | Vastriqo customer identity/session system | Required final replacement. |
| Admin customer read | ID, name, email, phone, state/tags/verification/tax/note, addresses | BMS `fetchAdminCustomerById`; login and `/customer-details` | Customer/profile/address records | Required fields must be selected; Shopify-only metadata needs decision. |
| Admin customer update | Checkout name/phone synchronization | BMS `updateCheckoutCustomerDetails` -> `customerUpdate` | Vastriqo-owned profile/contact update | Required behavior; dual-write is temporary. |
| Admin address CRUD | Mailing address fields and default address | BMS `create/update/delete/setDefaultCustomerAddress` | Address book | Required final replacement. |
| Admin order query | Order/money/status/line/address/payment/fulfillment data | BMS `fetchCustomerOrders`; UI consumes a subset | Retail order/payment/fulfillment records | Required for new platform; historical scope open. |
| Admin product synchronization | Products, variants, options, images, metafields, SEO, total inventory into BMS mirror | `SharedShopifyCollectionService.SHOPIFY_ADMIN_PRODUCT_FIELDS` and sync methods | One-time/repeatable importer into Vastriqo domain plus later native admin writes | Temporary migration source. |
| Shopify IDs/GIDs and handles | Product, variant, customer, cart, address and order identity; handle-based URLs | Passed through product cards, cart, wishlist, BMS mirror and customer bridge | Vastriqo IDs/handles; retain external-ID mapping only for migration/reconciliation | Transform; do not use as permanent primary identity. |
| Shopify publication/availability semantics | Storefront visibility and cart acceptance implicitly gate products | Product queries and cart errors; BMS custom collection does not independently filter product publication | Explicit publication and sellability rules | Required final replacement. |

## 5. BMS Dependency Matrix

| BMS capability | Runtime use and persistence | Replacement responsibility | Assessment |
| --- | --- | --- | --- |
| Custom collections | Used by home/men/women/kids and related products; `shopify_categories`, `shopify_category_products`, ordered by `sort_order` | Vastriqo collections and ordered membership | Conceptually reusable; data must transform to Vastriqo product IDs. |
| Shopify product mirror | Cards read `shopify_products` columns/JSON populated by Shopify Admin sync | Native catalog/admin plus migration importer | Useful source/reconciliation snapshot; raw schema/JSON should not be copied. |
| Related products | Active shared-category lookup in `SharedShopifyCollectionService.getRelatedProducts` | Related-products rule | Conceptually reusable. |
| Customer login bridge | Used: token exchange, Admin customer fetch, local `customers`, `customer_tokens`, app JWT | Customer identity/session | Replace; migrate selected profiles and external mappings. Shopify tokens should not survive final cutover. |
| `/customer-details` | Used by header session check; calls Shopify live | Session/current-customer read | Replace. |
| `/customer-profile` | Used by primary profile page; reads/writes local `customers` | Customer profile | Migratable concept/data; remove profile split. |
| Checkout validity/details | Used; validates local name/phone, exposes stored Shopify token, optionally dual-writes Shopify/BMS | Checkout eligibility/customer contact | Replace. |
| Addresses | Used; pure Shopify Admin proxy, no BMS address persistence | Address book | Replace; no BMS table to copy. |
| Orders | Used; pure Shopify Admin query for retail orders | Retail order history | Replace; BMS CCA `orders` is unrelated. |
| Wishlist | Used; BMS `wishlists` supports `customer_id` and stores Shopify product snapshots | Wishlist | Migrate/transform product and customer references. |
| Inquiries | Contact submit used; `/inquiries` history exists but identity linkage is unsafe | Inquiry/support | Conceptually reusable after ownership redesign. |
| Newsletter | Used; `newsletter_subscribers` stores normalized email/source | Subscription management | Migratable subject to consent/retention decision. |
| Wallet/referrals | No Vastriqo call site. BMS login creates a referral code as a side effect, but storefront sends no referral code and renders no wallet UI | None until product scope confirms it | Disconnected/undetermined; do not migrate automatically. |
| Storefront pages/menus | Public BMS routes and admin modules exist; no Vastriqo call site | Optional CMS/navigation | Unused by current storefront; do not migrate automatically. |
| Product sitemap | Vastriqo expects `/shopify-products/sitemap`; no audited BMS route matches | Published catalog sitemap | Current dependency is disconnected/obsolete. |
| Files/S3 | BMS inquiry API accepts optional `reference`, uploads to S3; current contact UI never sends a file and history UI never renders URL | Optional inquiry attachment/media service | Backend capability unused by current UI; target need is undetermined. |
| Native front auth/profile | `frontLogin`, `frontSignup`, `frontLogout` are uncalled; `/profile/update` uses native `users` profile API | None for active Shopify-backed flow | Legacy/transitional. |

## 6. Product Data Flow

### Direct listing flow

```text
Shopify Storefront `products(first: 100)`
  -> getProducts()
  -> /products, /collections, /search
  -> CollectionsProductsClient
  -> browser search/filter/sort
  -> ProductCard
  -> handle route or legacy Shopify ID route
```

### BMS merchandising flow

```text
Shopify Admin GraphQL sync
  -> BMS `shopify_products` + JSON `meta`
  -> BMS custom category membership/order
  -> home / men / women / kids / related-product cards
  -> handle route
  -> Shopify Storefront product detail
```

Consequences:

- **Verified:** BMS card success does not imply independent product detail or purchasability; the next step still needs a Shopify handle/product/variant.
- **Verified:** BMS sync limits differ from detail queries (for example images 20 vs detail images 100; references 10 vs detail references 100), and BMS metaobject references retain mainly IDs/handles rather than the fields rendered by detail. The mirror cannot be assumed complete.
- **Verified:** BMS custom collections filter collection status but do not independently enforce mirrored product publication/status in the storefront read path. A stale/unpublished card can lead to a missing Shopify detail.

## 7. Product Field Usage Matrix

| Field/group | Source/query | Actual consumer and behavior | Future DB requirement | Migration note |
| --- | --- | --- | --- | --- |
| Product ID | Shopify GID; BMS numeric/external ID | React keys, analytics, wishlist, related lookup, fallback detail URL | Yes: Vastriqo product identity | Transform; retain Shopify ID only as external mapping. |
| Handle | Shopify; BMS meta/url | Canonical product path and wishlist/order links | Yes: unique slug/handle | Migrate with redirect/collision policy. |
| Title | Both | Cards, detail, search, links, SEO template, analytics, order/wishlist snapshots | Yes | Migrate. |
| Description / HTML | Shopify detail/list | Search uses plain description; detail prefers HTML | Yes | Migrate required content; define sanitization. |
| Vendor | Shopify/list and BMS meta | Search and related-card shape; not visibly rendered on detail | Needs decision | Migrate only if brand/vendor discovery/admin requires it. |
| Product type | Shopify/list and BMS meta | Category filter, badge, search, gender heuristic, analytics | Yes as a product classification concept | Transform to deliberate category/type semantics. |
| Tags | Shopify list | Search and audience heuristic only | Conditional | Transform only tags with continuing business meaning. |
| Created timestamp | Shopify list | “New Arrivals” client sort | Yes if this sort remains | Use a deliberate merchandising/publish timestamp, not blindly Shopify creation time. |
| Updated/published/status/online URL | Queried by listing or BMS sync | Not directly rendered; Storefront visibility is implicit | Publication/status required; exact timestamps conditional | Recreate explicit publication lifecycle. |
| Featured image URL/alt | Both | Cards, product SEO image, cart, wishlist snapshot | Yes | Migrate chosen media and alt text. |
| Images/media previews | Shopify detail | Gallery, deduplicated by URL | Yes: ordered media | Migrate selected media; preserve ordering/alt. |
| Variant image | Listing query only | Not consumed by current card/detail renderer | Needs decision | Do not migrate solely because Shopify returns it. |
| Minimum price/currency | Both | Cards, filter/sort, detail fallback, analytics | Yes | Derive from authoritative variant prices or persist a projection. |
| Maximum/compare-at price | Shopify listing query | Not consumed by current UI | Needs decision | Do not migrate automatically; required only if pricing/promotion scope approves it. |
| Product options name/values | Shopify listing | Size/color filters and card swatches | Yes where they define selectable variants | Transform to option/value model. |
| Variant ID | Shopify | Selected merchandise sent to cart/checkout | Yes: Vastriqo variant identity | Transform; retain Shopify GID temporarily for migration. |
| Variant selected options | Shopify detail/list | Builds detail selectors, matching, card color fallback | Yes | Migrate normalized option selections. |
| Variant price/currency | Shopify detail/cart | Selected detail price and cart totals | Yes | Migrate/current-price ownership required. |
| Variant availability | `availableForSale`, `currentlyNotInStock` queried; `quantityAvailable` typed but not queried | Fallback variant choice; helper effectively permits any variant with ID; Shopify rejects invalid cart operations | Yes: computed sellability | Recreate with inventory rules; current UI is not authoritative. |
| SKU | Queried on list/detail and BMS sync | No current storefront display/use | Likely operational, but target admin requirement not proven here | Needs decision; do not infer from query alone. |
| Barcode/weight/shipping/tax flags | Shopify listing query/BMS sync subset | No current storefront consumer | Undetermined | Do not migrate solely due query presence. |
| Shopify category/collections | Rich detail/list queries | Direct category/collection fields are not consumed; Vastriqo uses BMS custom collections instead | Collection concept yes; Shopify relationship no | Recreate from approved BMS merchandising. |
| SEO title/description | Shopify list and BMS mirror | Not consumed; dynamic metadata uses local template/title/image | SEO behavior yes; Shopify values not proven | Needs decision/quality review before migration. |
| Metafield namespace/key/type/value | Shopify detail; BMS meta | Detail labels/values and BMS color/size extraction | Yes for approved structured attributes | Transform into Vastriqo attributes/content, not opaque Shopify blobs. |
| Metaobject reference fields/handle | Shopify detail fragment | Display chooses `name`, `label`, `title`, `value`, first readable field, then handle | Yes for referenced attributes that remain | Flatten or model references deliberately. |
| Metaobject ID/image | Queried in fragment | ID and referenced MediaImage are not rendered by current specification logic | Not proven | Do not migrate automatically. |
| BMS collection sort order | BMS membership | Determines home/gender/related order | Yes: ordered membership/merchandising rank | Transform. |
| BMS image override | Admin input updates mirrored product `image_url`; cards consume it | Can change card image independently of Shopify featured image | Conditional merchandising requirement | Decide whether future collection-specific or product-wide override is intended. |

### Metafield/custom-data usage

| Key | Current use | Classification |
| --- | --- | --- |
| `color-pattern` | Hidden from detail specifications; BMS mirror reference handles/values become card Color options. Direct detail option selection comes from variant selected options, not this field. | Transform if needed for card/filter merchandising. |
| `size` | Same pattern as color: hidden specification; BMS card Size options; detail variants remain authoritative. | Transform if needed for filters/card display. |
| `fabric` | Displayed as a detail specification when value is present. | Required current custom content. |
| `age-group` | Queried but explicitly hidden from specifications; not mapped into direct list filters. | Not proven by current UI; needs decision. |
| `target-gender` | Queried/hidden; BMS sync derives a `gender` column, but collection mapper does not expose it to the grid. | Current storefront use is not established; needs decision. |
| `care-instructions` | Displayed as specification. | Required current custom content. |
| `clothing-features` | Displayed as specification. | Required current custom content. |
| `neckline` | Displayed as specification. | Required current custom content. |
| `top-length-type` | Displayed as specification. | Required current custom content. |
| `material` | Displayed as specification. | Required current custom content. |
| `material-composition` | Displayed as specification. | Required current custom content. |
| `sleeve-length-type` | Displayed as specification. | Required current custom content. |
| `pattern` | Displayed as specification. | Required current custom content. |
| `fit` | Displayed as specification. | Required current custom content. |
| `styling_tips`, `about_this_item` | No explicit default identifier or named consumer exists. They would be queried/displayed only if production `SHOPIFY_PRODUCT_METAFIELD_IDENTIFIERS` includes them. That runtime value is not committed. | **Undetermined**; not established from audited source. |
| Other configured identifiers | Server environment can add `namespace.key` identifiers; all non-hidden, non-empty fields render generically. | **Undetermined** until production configuration/data is inventoried. |

## 8. Data Ownership Classification

### A. Must become Vastriqo-owned

- Products, handles/publication, variants, option selections, required attributes, media, prices and sellability.
- Collections and ordered product membership.
- Inventory availability/reservations sufficient to accept orders independently.
- Customer identity/session and profile.
- Addresses, carts, checkout state, retail orders/order items, payment state and fulfillment/shipment state required by the chosen workflows.
- Wishlists, inquiries and newsletter subscription state if current features are retained.

### B. Must be migrated into Vastriqo

- Required Shopify catalog fields/media/variant relationships and approved product attributes.
- BMS custom collection definitions, membership and order after mapping to Vastriqo product identities.
- Current wishlists if customer/product identities are migrated and retention is approved.
- BMS inquiries/newsletter records if retention, consent and identity rules approve them.
- Customer profiles/addresses/order history only to the extent decided in the migration requirements; source establishes their existence, not mandatory historical scope.

### C. Must be recreated/restructured rather than copied

- BMS `shopify_products.meta` JSON and Shopify GIDs as primary identities.
- Customer authentication and `customer_tokens`.
- Product/variant/inventory/cart/order/payment/fulfillment models.
- Inquiry ownership, because `inquiries.user_id` is incompatible with the active customer identity path.
- Product publication, search/filtering and availability logic.
- Any CMS model, if approved, because current BMS pages/menus are not proven against the renderer.

### D. Can remain external temporarily

- Shopify catalog/detail/cart/checkout/payment/customer/address/order operations during controlled module migration.
- BMS collections, customer bridge, wishlist, inquiry and newsletter until Vastriqo equivalents and data migration are validated.
- GA4, Meta Pixel, email/SMS, payment, shipping, storage/CDN and other explicitly chosen external services; “external” does not mean Shopify remains the commerce owner.

### E. Can be discarded

- Shopify access/refresh tokens and Shopify-specific session keys after the relevant cutover and retention window.
- Raw Shopify response fields with no approved storefront/admin/operational requirement.
- Transient carts that are intentionally allowed to expire rather than migrated, if cutover policy approves this.
- Missing-route clients and obsolete aliases after compatibility/redirect decisions.

### F. Legacy/unused

- Native BMS `frontLogin`, `frontSignup`, `frontLogout` helpers and current `/signup` as a distinct registration implementation.
- Cookie-based Shopify customer token/refresh/customer GraphQL helper path in `lib/shopify/customer-auth.ts`; the active callback does not set those cookies.
- `getCustomAuthMe()` calls missing BMS `/auth/me` and has no caller.
- `/api/cart` route is not called by the audited storefront; cart context uses server actions in `lib/shopify/index.ts`.
- BMS storefront pages/menus and Vastriqo generic `/products?gender=` fallback have no matching/current runtime use for configured category pages.

## 9. Future Vastriqo API Responsibilities

These are capabilities, not endpoint designs.

### Catalog and merchandising

- Published product list/detail projections, handles and redirects.
- Variant/options, media, required attributes/specifications, pricing and availability.
- Collections with ordered membership, home/category merchandising and related-product selection.
- Full-catalog search, filters and stable sorts/pagination.
- Product sitemap data and SEO inputs.

### Inventory

- Stock source of truth, adjustments, availability computation, reservations/releases and checkout/order reconciliation.

### Customer and identity

- Registration/login, verification/recovery, secure sessions, logout/revocation and current-customer read.
- Profile and address-book CRUD/default address.
- Migration/account-linking support for Shopify-backed customers if historical accounts move.

### Cart, checkout and orders

- Cart create/read/line mutation/persistence/expiry/merge behavior.
- Server-side price and inventory validation.
- Checkout contact/address/shipping/tax/discount/payment orchestration.
- Idempotent retail order creation; customer order list/detail.
- Payment, cancellation/refund/return and fulfillment/shipment/tracking state as approved.

### Customer engagement and content

- Wishlist, inquiries/support and newsletter subscription behavior.
- Media/attachment handling where requirements approve it.
- Storefront content/navigation only if runtime admin management is selected.
- Wallet/referrals only if a separate requirements decision retains them.

### Platform concerns revealed by current behavior

- Consistent validation/errors, authorization, rate/abuse protection and privacy/consent operations.
- External-service webhooks, idempotency, retries and reconciliation.
- Audit/observability sufficient for checkout, payment, inventory and order support.

## 10. Future Vastriqo Database Requirements

Conceptual data supported by current requirements:

- **Product** — identity, handle, title, descriptions, classification/tags as approved, publication and SEO inputs.
- **ProductVariant** — identity, selected option values, current price/currency, sellability and optional operational identifiers.
- **ProductOption / ProductOptionValue** — selectable dimensions such as size/color and variant relationships.
- **ProductMedia** — ordered URL/object reference, alt text, role and optional variant association.
- **ProductAttribute / AttributeValue** — approved fabric/care/material/fit and other specifications without Shopify metaobject coupling.
- **Collection / CollectionProduct** — active collection and ordered product membership/merchandising overrides.
- **Inventory concept** — stock and reservations at the granularity/locations later approved.
- **Customer / CustomerCredential-or-Identity / Session** — profile, authentication linkage and session lifecycle.
- **Address** — customer addresses and default selection.
- **Cart / CartItem** — owner/session, variant, quantity, price context and lifecycle.
- **Checkout or CheckoutAttempt** — only if needed to persist orchestration/idempotency across external calls.
- **Order / OrderItem** — immutable purchase snapshots, totals, status and customer/address snapshots.
- **Payment/PaymentAttempt** — provider references and transaction/status history required for reconciliation.
- **Fulfillment/Shipment/Tracking** — only to the extent approved, but necessary to replace promised tracking independently.
- **WishlistItem** — customer/product relationship.
- **Inquiry / InquiryAttachment** — submitter/contact, customer link, message/status and optional media.
- **NewsletterSubscription** — normalized email, source, consent/status/timestamps.
- **ExternalIdMapping / MigrationRecord** — temporary or retained provenance for Shopify/BMS reconciliation.
- **StorefrontContent/Menu** and **Wallet/Referral** — not yet justified as mandatory database concepts; require explicit scope decisions.

This is not a table list. Several concepts may be embedded, projected, event-backed, or split later.

## 11. Migration Classification

| Current data/capability | Classification | Reason |
| --- | --- | --- |
| Required Shopify products/variants/options/prices/media/attributes | **Transform** before catalog cutover | Core catalog data is required, but Shopify's shape/IDs are not the target model. |
| Inventory/availability | **Recreate + reconcile** | BMS snapshot is not an inventory system; live Shopify remains authority. |
| BMS custom collections/order | **Transform** | Current merchandising is actively used and should map to Vastriqo product IDs. |
| BMS raw product mirror JSON | **Do not migrate as target data** | Use as import comparison/reconciliation evidence only. |
| Shopify customer profiles | **Migrate later / needs decision** | Identity strategy, verification, consent and account linking must precede import. |
| Shopify addresses | **Migrate later / needs decision** | Live-only data; scope and customer-account migration determine need. |
| Shopify historical orders | **Needs decision** | Current history is used, but retention horizon/import-versus-archive is not established. |
| Active Shopify carts | **Needs decision; normally expire or bridge temporarily** | They are transient Shopify objects referenced only by local cart ID. |
| BMS wishlists | **Transform later** | Valuable current feature, but both customer/product references change. |
| BMS inquiries | **Transform** if retained | Correct customer ownership and validate attachments/status history. |
| BMS newsletter subscribers | **Migrate** if consent/retention permits | Current active feature with simple records. |
| Source-controlled content/navigation/assets | **Recreate/retain** | Already Vastriqo-owned; move to DB only if CMS requirement is approved. |
| BMS storefront pages/menus | **Do not migrate by default** | No current storefront consumer. |
| BMS wallet/referral data | **Needs decision** | No current UI/runtime call; login side effect alone is insufficient evidence. |
| Shopify/BMS access tokens and secrets | **Do not migrate** | Temporary integration credentials, not Vastriqo customer/domain data. |
| Analytics IDs/event setup | **Recreate/configure** | External integration; consent and final measurement plan need review. |

## 12. Temporary Dependencies

| Temporary dependency | Why it may remain | Replacement/removal condition |
| --- | --- | --- |
| Shopify product/detail | Preserve live catalog while native catalog/admin/import validation is built | Native products/variants/media/prices/publication are reconciled and storefront reads Vastriqo API. |
| Shopify inventory/cart | Avoid split-brain selling before stock/reservation/cart are proven | Inventory authority, cart lifecycle and concurrency/reconciliation pass cutover tests. |
| Shopify checkout/payment | Continue taking orders before payment/tax/shipping/order orchestration exists | Provider integrations, webhooks, idempotent order creation and reconciliation are production-ready. |
| Shopify Customer Accounts | Preserve customer access while Vastriqo identity/migration is designed | Account migration/linking, verification, sessions and recovery are validated. |
| Shopify addresses/orders | Preserve account experience during identity/order migration | Address ownership and historical/new-order access are available through Vastriqo API. |
| BMS custom collections/product mirror | Preserve home/gender merchandising | Collections map to native products and admin can manage them. |
| BMS customer bridge/wishlist | Preserve sessions and saved products | Vastriqo identity is active and wishlist records are remapped/reconciled. |
| BMS inquiry/newsletter | Preserve support/subscription writes | Native modules and approved data imports are validated. |

Every temporary read/write needs an explicit owner and exit criterion. A permanent `vastriqo-api -> Shopify` core-commerce path would not satisfy the project goal.

## 13. Legacy/Unused Functionality

- **Verified:** `frontLogin`, `frontSignup`, and `frontLogout` in `lib/front-auth.ts` have no call sites. `/signup` renders the Shopify login component.
- **Verified:** `/profile/update` uses the native BMS `users` profile API and is not linked from the primary account navigation, while `/profile` uses BMS `customers`.
- **Verified:** `/inquiries` uses native front JWT conventions and is not linked from `AccountLayout`; active customer IDs can be written/read as `inquiries.user_id` despite that column belonging to BMS users.
- **Verified:** `lib/shopify/customer-auth.ts` contains cookie token exchange/refresh/customer GraphQL functions, but active `/api/auth/callback` delegates exchange to BMS and does not set those cookies. `/api/auth/refresh` therefore belongs to an overlapping path.
- **Verified:** `getCustomAuthMe()` and `logoutCustomAuth()` target root `/auth/me` and `/auth/logout`; no audited BMS routes match. Logout errors are intentionally suppressed.
- **Verified:** `app/api/cart/route.ts` creates a Shopify cart but has no storefront caller.
- **Verified:** generic `getProductsFromGenericApi()` calls `/products?gender=...`; configured men/women/kids routes always use BMS collection slugs, and audited BMS has no root `/products` route.
- **Verified:** `getSitemapProducts()` calls a missing `/shopify-products/sitemap` route.
- **Verified:** BMS storefront page/menu and wallet APIs have no Vastriqo call sites.
- **Verified:** `/about-us`, `/dashboard`, `/addresses`, and `/profile/cart` are compatibility aliases/redirects. `/details?id=` is a legacy ID route that redirects to the handle when possible.
- **Verified:** order line fallback links use `/products?search=...`, but `/products` does not read a `search` parameter; the intended filtered fallback is ineffective.
- **Verified:** BMS order query requests billing address, fulfillment/tracking, discounts and many transaction fields that the current orders UI does not consume.

No item above was deleted or changed.

## 14. Unresolved Questions

Closure note: these questions are retained as the original audit record. They are not a current questionnaire. The final A–E classification, production-validation checklist, genuine business decisions and Phase 2 readiness are in `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md` §§4–23.

1. What product/metaobject identifiers and actual values are configured in production through `SHOPIFY_PRODUCT_METAFIELD_IDENTIFIERS`? `styling_tips` and `about_this_item` cannot be confirmed from committed source.
2. How many live products, variants, media items, customers, addresses, wishlists, inquiries and orders exist? The source limits may truncate live data.
3. Which BMS collection records and overrides are actively curated in production, and are any products stale/unpublished/deleted?
4. Which current Shopify checkout behaviors are real requirements: payment methods, COD, discounts, tax, shipping rates, fraud checks, notifications and order confirmation? They are not established from repository source.
5. Must historical Shopify orders remain fully interactive, be imported read-only, or remain in an archive during/after cutover?
6. Is guest checkout intentionally prohibited, or is the current mandatory login only a Shopify/customer-token constraint?
7. Are customer account migration, password creation, email/phone verification and consent requirements defined?
8. Are cancellation, return, exchange, refund and real tracking promised final features? Current content describes them, but code does not implement them.
9. Should product vendor, SKU, barcode, compare-at price, weight, tax/shipping flags and Shopify SEO overrides exist in the future admin/domain? Queries alone do not prove requirement.
10. Should source-controlled content remain deployed with the storefront or become Vastriqo-admin managed? Current BMS CMS records are not evidence of current use.
11. Should wallet/referral functionality be retained? No current storefront UI consumes it.
12. Does the deployed BMS expose routes or behavior absent from this local revision, particularly the sitemap and logout paths? Not established from audited source.
13. What permanent, explicit integrations—if any—must connect Vastriqo with CCA/BMS after separation?

## 15. Recommended Next Phase 1 Steps

Historical note: these were the audit's recommended next steps. They were completed through the requirements, product, conceptual-architecture and final closure documents. The current next phase is Phase 2 detailed design; no implementation is authorized by this note.

1. Validate this source-derived matrix with product, operations, support and finance stakeholders; distinguish implemented behavior from policy-page promises.
2. Export non-mutating inventories/samples from Shopify and BMS to validate field usage, custom metafields/metaobjects, counts, truncation risks and data quality.
3. Produce a signed feature disposition register: preserve, change, retire or defer for every matrix row, including wallet/CMS/returns/tracking.
4. Define the catalog vocabulary and operational rules at the requirement level: product/variant/options, publication, pricing, inventory, collection merchandising and search.
5. Define customer/account and historical-data requirements before choosing an authentication or migration implementation.
6. Document checkout-to-order business requirements currently hidden inside Shopify: shipping, tax, discounts, payment, cancellation/refund/return and fulfillment.
7. Create a migration source-to-concept mapping and reconciliation plan using the conceptual entities in this audit; do not create the physical schema yet.
8. Only after those requirements are approved, proceed to detailed domain/data/API/admin design.

## 16. Evidence / Source References

### Vastriqo catalog and commerce

- `vastriqo/lib/shopify/index.ts` — Storefront GraphQL product and cart queries/mutations; metafield identifiers and query limits.
- `vastriqo/components/collections/sections/collections-grid-section/products-client.tsx` — product field consumers, browser search/filter/sort, cards, wishlist and quick-add behavior.
- `vastriqo/components/collections/details-page/index.tsx` — variant selection, gallery, specifications/metaobject display, related products, add/buy-now behavior.
- `vastriqo/lib/shopify-api-collection-products.ts` and `lib/product-metafield-options.ts` — BMS mirror-to-card transformation and color/size extraction.
- `vastriqo/components/cart/cart-context.tsx`, `components/cart/cart-page.tsx`, and `app/api/cart/checkout/route.ts` — cart persistence, checkout validation, buyer attachment and redirect.

### Vastriqo identity and account

- `vastriqo/lib/shopify/customer-auth-client.ts` and `components/account/account-pages/shopify-callback-page.tsx` — active browser PKCE/OAuth and local session creation.
- `vastriqo/app/api/auth/*` and `lib/api/custom-auth-route.ts` — BMS proxy boundaries.
- `vastriqo/components/account/account-pages/profile-page.tsx`, `address-page.tsx`, `orders-page.tsx`, `wishlist-page.tsx`, `inquiries-page.tsx` — actual account data consumers and actions.
- `vastriqo/lib/front-auth.ts` — current localStorage session, wishlist/inquiry helpers and overlapping native-BMS helpers.

### Vastriqo content, SEO and external services

- `vastriqo/components/home/home-page/index.tsx` and `content/site-content.json` — local home/navigation/footer content plus BMS home collection.
- `vastriqo/components/common/site-header/index.tsx`, `site-footer/index.tsx`, and `app/layout.tsx` — global navigation, account/cart checks and external analytics/fonts.
- `vastriqo/app/sitemap.ts`, `app/sitemap/page.tsx`, `lib/internal-products.ts`, and `lib/seo.ts` — sitemap and metadata behavior.
- Static/editorial implementation: `components/about-us`, `components/why-vastriqo`, `components/faq`, `components/size-chart`, `app/shipping-returns/page.tsx`, and `app/privacy-policy/page.tsx`.

### BMS verification

- `bms-api/src/api/admin/modules/shopify-apis/route.ts`, `controller.ts`, `service.ts`, `checkout.ts`, and `customer-auth.ts` — OAuth/customer bridge, Shopify Admin customer/address/order calls and checkout token behavior.
- `bms-api/src/shared-services/shopifyCollection.ts` and `db-migrations/20260527130000_shopify_product_collections.ts` — product mirror, sync fields, collections, membership/order and related products.
- `bms-api/src/api/front/modules/wishlist/*` and `db-migrations/20260523110000_customer_wishlists.ts` — active customer/user wishlist ownership.
- `bms-api/src/api/front/modules/inquiry/*`, `shared-services/inquiry.ts`, and `api/front/middlewares/jwt-auth.ts` — inquiry persistence, files and identity mismatch evidence.
- `bms-api/src/api/front/modules/newsletter/*` and `shared-services/newsletter.ts` — subscription persistence.
- `bms-api/src/api/front/modules/pages`, `modules/menus`, `modules/wallet`, and corresponding shared services — present BMS capabilities with no verified Vastriqo consumer.

### Whole-application coverage check

All page routes under `vastriqo/app` were inventoried, including customer/account routes, aliases/redirects, API route handlers, catalog/product/cart paths, content/legal/help pages, sitemap, global layout, analytics and host proxy. All imports/call sites for Shopify, BMS, fetch, local storage and external URLs were searched outside generated/dependency directories. Important BMS calls were followed through routes/controllers/services and relevant migrations. No live-data conclusions were invented where source could not establish them.
