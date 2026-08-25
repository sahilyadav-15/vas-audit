# Vastriqo Platform

## Project Overview

This repository root is a working directory containing three independent applications:

- `bms-admin`: the existing browser-based administration application.
- `bms-api`: the existing BMS/CCA backend and MySQL database layer, including several storefront and Shopify bridge capabilities.
- `vastriqo`: the customer-facing Vastriqo e-commerce storefront.

This document is a read-only audit baseline of the local source as it existed on 2026-08-23. It describes implemented behavior, current ownership, integration boundaries, and source-established technical debt. It is not a migration plan and does not describe future work as if it already exists.

The long-term direction supplied with the audit is for Vastriqo to stop using Shopify as its core commerce backend and eventually use a dedicated `vastriqo-api` and a separate Vastriqo database. Neither of those exists in this workspace today.

## Current System

The current system has two overlapping concerns:

1. CCA business operations are implemented in `bms-admin` and `bms-api`: internal users, CCA products, production-style orders, expenses/payments/salaries, GST invoices, seller details, and inquiries.
2. Vastriqo commerce is implemented by the Next.js storefront, Shopify, and selected BMS APIs: Shopify owns the live catalog/cart/checkout/order/address capabilities; BMS mirrors Shopify products and stores custom collection membership, customer bridge records, wishlists, referrals/wallet data, inquiries, newsletter subscriptions, and configurable storefront records.

There is no root package manifest, workspace definition, Compose file, shared build configuration, or root orchestration script. Each child directory has its own `package.json`, lockfile, Dockerfile, Git history, dependencies, and deployment posture. The root itself is not a Git repository, so this root README is not currently covered by any of the three child repositories.

## Applications

### bms-admin

#### Architecture and technology

| Concern | Verified implementation |
| --- | --- |
| Framework | React 19 in React Router 7 framework mode |
| Language | TypeScript with strict checking |
| Rendering/routing | React Router file routes configured in `app/routes.ts`; `ssr: false`, so the application is built as a client-side SPA |
| Build tool | Vite 6 with the React Router, Tailwind, and TypeScript-path plugins |
| Server state | TanStack React Query 5 |
| Local UI state | React component state plus a small loader context |
| HTTP client | A shared Axios instance with request/response interceptors |
| UI | Ant Design 5 and its theme JSON; Ant Design icons are consumed transitively through `antd` |
| Styling | Tailwind CSS 4 utility classes plus a global `app/app.css` stylesheet |
| Authentication | Email/password login to BMS; JWT and serialized user are kept in browser `localStorage`; protected routes test only for token presence; Axios adds `Authorization: Bearer ...` and clears the session on HTTP 401 |
| API configuration | The deployed BMS API URL is hard-coded in `app/lib/axios.ts`; a localhost alternative is commented out. No admin environment template or runtime environment abstraction is present |
| Deployment | Multi-stage Node 20 Alpine Docker build; no admin GitHub Actions workflow is present in the audited source |
| PWA-related behavior | Manifest, icons, service worker, and production-only service-worker registration are present |

The shared Axios base URL points to the deployed BMS API. There is no direct browser-to-Shopify client in the admin: Shopify actions go through `bms-api`.

#### Important directories

- `app/api-hooks/`: React Query query/mutation hooks. This is the main admin-to-API boundary.
- `app/components/`: domain UI for lists, forms, previews, collection/product assignment, and shared loading/protection behavior.
- `app/constants/`: central API path constants.
- `app/contexts/`: the global module-loading overlay state.
- `app/lib/`: the Axios instance and browser authentication helpers.
- `app/routes/`: thin route entry points that compose domain components. Some routes exist but are hidden from the main navigation.
- `app/types/`: frontend API contracts and domain types.

#### Implemented admin modules

| Module | Main UI and state | Backing API/data | Current character |
| --- | --- | --- | --- |
| Authentication/profile | `/login`, `/profile`, `/change-password`; `api-hooks/auth.ts` and user mutation hooks | BMS `users` and admin JWT endpoints | Shared infrastructure, currently shaped around BMS user roles |
| Users | `/user`; list/create/edit/delete forms; `api-hooks/user.ts` | BMS `users` | CCA-oriented: employee/admin/vendor/fabric-supplier/checker/sales roles and company-oriented fields |
| Seller details | `/seller-user-details`; list and large seller/bank/tax/signature form | BMS `user_seller_details`, users, and S3 file uploads | CCA-specific invoice/seller identity |
| CCA products | `/product`; list and form for name, HSN, price, sizes, colors, and images | BMS `products`, `product_sizes`, `product_colors`, `product_images` | CCA-specific internal product model; distinct from Shopify products |
| Transaction categories | `/category`; list/create/edit | BMS `categories` | Used for transaction categorization; the route is implemented but the main navigation item is commented out |
| Orders | `/order`; searchable list and modal create/edit/details flows | BMS `orders` and `order_items` tied to internal products and users | CCA production order workflow, not Shopify customer orders |
| Invoices | `/invoice`, create/detail/edit routes; GST form and printable preview | BMS `invoices`, `invoice_items`, users, orders, products, and seller details | CCA-specific invoicing, bank, GST, PAN, tax, and signature workflow |
| Transactions | `/expense`, `/payment`, and legacy `/transaction`; shared transaction list/form | BMS `transactions`; receipt upload via BMS/S3 | CCA finance. Expense is in navigation; payment navigation is commented out; payment is not a separate persistence model. Types include expense, payment, and salary |
| Inquiries | `/inquiry`; filters, detail drawer/modal, reference-file link, status update | BMS `inquiries` | Shared in concept; submitted by the Vastriqo contact flow, but identity linkage has a verified mismatch described below |
| Shopify product mirror | Shopify Collections -> Products tab; sync all/single, filter, inspect metafields/SEO/gender, open Shopify Admin | BMS `/admin/shopify/*`; BMS calls Shopify Admin GraphQL and persists `shopify_products` | Vastriqo/storefront-related Shopify bridge |
| Custom storefront collections | Shopify Collections -> Categories tab and detail page; CRUD, assign/remove/reorder products, override display image | BMS `shopify_categories` and `shopify_category_products` | Vastriqo-oriented collection curation over mirrored Shopify products |
| Storefront pages | `/storefront-pages`; CRUD and ordered product/category/image items | BMS `storefront_pages` and `storefront_page_items` | Implemented in admin/API, but the current Vastriqo source does not read these endpoints |
| Storefront menus | `/storefront-menus`; CRUD and ordered page assignments | BMS `storefront_menus` and `storefront_menu_pages` | Implemented in admin/API, but the current Vastriqo source does not read these endpoints |
| Wallet settings | `/wallet-settings`; coin-to-rupee and referral reward settings | BMS `wallet_settings` | Customer/referral infrastructure; no current Vastriqo wallet screen was found |
| File uploads | Used by products, transactions, storefront pages, and seller details | BMS multipart endpoint, then S3 | Shared infrastructure |

No separate payments UI/data service, Shopify order administration screen, customer list screen, role-based navigation, or business/tenant switcher was found. Route protection establishes authentication but does not authorize individual modules by role.

### bms-api

#### Technology

| Concern | Verified implementation |
| --- | --- |
| Runtime/framework | Node.js (Node 20 Docker image) and Express 5 |
| Language/build | Strict TypeScript compiled to CommonJS in `dist/` |
| Database | MySQL through `mysql2` |
| Query/migration layer | Knex 3; query-builder services rather than an ORM/model instance layer |
| Authentication | Bcrypt passwords and JSON Web Tokens; separate middleware conventions for admin/front users and Shopify-backed customers |
| Validation | `express-validator` middleware on many save/auth routes; some modules validate inline or only in controllers |
| Transactions | Per-request Knex transactions on write routes; response helpers also commit or roll back unfinished request transactions |
| Files | Multer memory uploads and an AWS SDK S3 service; `/uploads` and static directories also exist |
| External integrations | Shopify OAuth/token endpoints, Shopify Admin GraphQL, Shopify customer/account data, and AWS S3. Redis packages and a Redis loader exist, but the loader is disabled |
| Logging | Morgan request logging, console logging, and Winston daily rotating file logs |
| Testing | No test runner script, test dependency, test configuration, or test files were found |
| Deployment | Docker/PM2; GitHub Actions builds to ECR, replaces an EC2 container, and runs Knex migrations after start |

Environment loading checks `.env.${NODE_ENV}` and otherwise calls default `dotenv.config()`. There is no committed backend `.env.example`. The default port in source is 3000; deployment supplies port 4000. Database migration is not performed at application startup: it is a manual npm command locally and an explicit deployment workflow step.

#### Actual layers

```text
src/app.ts
  -> loaders (database, Express, scheduler; Redis currently disabled)
     -> /admin router
     -> /front router
     -> /newsletter router
        -> route modules
           -> auth / transaction / upload / validation middleware
           -> controllers
              -> shared-services (primary query-builder data layer)
                 -> global Knex connection / request transaction
              -> Shopify and S3 integrations where required
```

Controllers coordinate HTTP input, response formatting, transactions, and service calls. Most database behavior is in `src/shared-services`, although some controllers query Knex directly (notably authentication, wishlist, and parts of Shopify/customer behavior). Types are structural TypeScript interfaces, not active-record models.

#### API modules and protection

The main response shape is generally `{ success, message, code, data, extra?, qdata? }`. Shopify customer middleware returns a smaller `{ success: false, error }` error shape in some paths, so response formatting is not fully uniform.

| Domain | Routes/controller/service | Persistence or external owner | Protection/validation |
| --- | --- | --- | --- |
| Admin auth | `POST /admin/auth/login` | BMS `users` | Public login; bcrypt/JWT; required-field checks in controller |
| Admin users | `/admin/user/*` | `users`, sometimes `user_seller_details` | Admin JWT; express-validator on writes |
| CCA products | `/admin/product/*` | `products` and child tables | Admin JWT; product validators; transactional writes |
| CCA categories | `/admin/category/*` | `categories` | Admin JWT; validator on save |
| CCA orders | `/admin/order/*` | `orders`, `order_items`, internal products/users; order completion can call wallet reward logic | Admin JWT; detailed order/item validation |
| Invoices | `/admin/invoice/*` | `invoices`, `invoice_items`, users/orders/products/seller data | Admin JWT; invoice validation on save |
| Transactions | `/admin/transaction/*` | `transactions`, users | Admin JWT; transaction validation; soft deletion |
| Seller details | `/admin/user-seller-details/*` | `user_seller_details`, users | Admin JWT; extensive field validation |
| Inquiries | `/admin/inquiry/*` | `inquiries`; reference files in S3 | Admin JWT; ID/status validation |
| Files | `POST /admin/file/upload` | S3 object URL returned to caller | Admin JWT; Multer, but no route-level file type or size restriction |
| IoT | `POST /admin/iot/save` | `iot_payloads` | Mounted before the global admin JWT and has no module auth middleware |
| Shopify bridge | `/admin/shopify-apis/*` | Shopify plus local customers/tokens | Related products, collection lookup, and customer login are public; individual customer endpoints use customer JWT; token-storage endpoint uses admin JWT |
| Shopify sync | `/admin/shopify/sync*`, `/admin/shopify/products` | Shopify Admin GraphQL -> `shopify_products` mirror | Admin JWT inherited from the main router |
| Custom collections | `/admin/categories/*` and `/front/collections/:slug` | BMS collection tables over Shopify IDs | Admin writes require admin JWT; front reads are public |
| Storefront pages/menus | admin CRUD plus `/front/pages/*`, `/front/menus` | BMS page/menu tables | Admin JWT for management; public active-record reads |
| Wallet/referrals | `/admin/wallet/settings`, `/front/wallet/*` | wallet settings/ledger/rewards plus users/customers | Admin JWT for settings; Shopify-customer JWT for current front routes |
| Front native auth/profile | `/front/auth/*`, `/front/user/*` | BMS `users` | Public login/signup then front JWT; validators on login/signup/profile writes |
| Front wishlist | `/front/wishlist/*` | `wishlists`, keyed to either `user_id` or `customer_id` | Generic front JWT middleware; controller distinguishes a token with `type: CUSTOMER` |
| Front inquiry | `/front/inquiry/*` | `inquiries.user_id` and S3 | Submit allows optional front JWT; list/detail require front JWT |
| Newsletter | `POST /newsletter/subscribe` | `newsletter_subscribers` | Public; controller/service email validation, no route-level authentication |

#### Database architecture

Knex migrations are the schema record. The application does not introspect or auto-synchronize TypeScript types into MySQL. The main conceptual groups are:

```text
CCA operations
users
  |-- orders -- order_items -- products -- product_sizes/colors/images
  |-- transactions
  |-- invoices -- invoice_items
  `-- user_seller_details

Storefront/customer bridge
customers (Shopify customer ID)
  |-- customer_tokens (keyed by a hash of the app JWT)
  |-- wishlists (also supports users)
  |-- referrals / wallet ledger
  `-- no local address or commerce-order tables

Shopify catalog curation
shopify_products (local mirror)
  `-- shopify_category_products -- shopify_categories
                                    |-- storefront_page_items -- storefront_pages
                                    `-- storefront_menu_pages -- storefront_menus

Other
inquiries -> optionally users
newsletter_subscribers
iot_payloads
wallet_settings / wallet_ledger / referral_rewards
fabrics / fabric_styles / fabric_images (schema only; no routed module found)
shopify_admin_tokens
```

Important distinctions:

- `products` is the CCA internal product model; `shopify_products` is a separate local mirror used by Vastriqo collection and related-product features.
- `orders` is a CCA production order model. Vastriqo customer orders are read live from Shopify and are not persisted into this table.
- `users` backs BMS staff/partners and an older native front-auth path. `customers` backs the current Shopify customer-account bridge.
- Customer addresses, Shopify cart state, Shopify checkout, payments, fulfillment, and Shopify orders do not have equivalent local BMS entities.
- Wallet ledger ownership is polymorphic (`CUSTOMER` or `USER`) rather than enforced by foreign keys.

The migration set is chronological and sometimes adapts existing tables conditionally. It therefore records compatibility with older schemas as well as the intended current schema.

#### Shopify boundary inside bms-api

`src/api/admin/modules/shopify-apis` and `src/shared-services/shopifyCollection.ts` are the integration boundary.

Shopify Admin access:

- A scheduled task requests a client-credentials Admin token at startup and every four hours when configuration is present.
- Admin tokens are encrypted and stored in `shopify_admin_tokens` by shop domain.
- Admin GraphQL reads customers, orders, addresses, products, images, variants, inventory fields, options, metafields, SEO, tags, vendor/type, and timestamps.
- Admin GraphQL writes customer profile fields and creates/updates/deletes/defaults customer addresses.

Catalog synchronization:

- Full and single-product sync fetch Shopify products through Admin GraphQL.
- BMS normalizes the Shopify numeric ID and persists title, minimum price, handle-derived URL, image, gender, SEO, Shopify timestamps, sync timestamp, and a JSON `meta` document containing variants/options/images/metafields and other Shopify attributes.
- BMS owns custom category records, category slugs/status/order, product membership/order, and optional display-image overrides.
- BMS does not write product/catalog changes back to Shopify. Shopify is authoritative for source product, variant, option, image, inventory, metafield, and price data; BMS is a mirror plus curation layer.

Customer/account bridge:

- Vastriqo performs Shopify customer-account OAuth with PKCE. BMS exchanges the code with Shopify, decodes the returned identity token, reads the customer through Shopify Admin GraphQL, upserts a local `customers` row, stores the Shopify customer access token, and returns its own 24-hour `type: CUSTOMER` JWT.
- BMS customer profile reads/updates normally use the local `customers` row. The checkout-details path first writes name/phone to Shopify and then updates the local row.
- Address CRUD and order history are live Shopify Admin GraphQL operations; no local address/order copy is created.
- Checkout validity returns the stored Shopify customer access token to the Vastriqo server when required customer fields exist. Vastriqo uses it to attach the buyer identity to the Shopify cart.
- Wishlist rows, wallet/referral rows, and the local customer bridge are BMS-owned additions around a Shopify-owned customer identity.

### vastriqo

#### Architecture and technology

| Concern | Verified implementation |
| --- | --- |
| Framework | Next.js 16 App Router, with server components, client components, dynamic metadata, and route handlers |
| Language | Strict TypeScript / React 19 |
| Routing | Files under `app/`; a dynamic top-level `app/[product-slug]` product route; `proxy.ts` redirects apex GET/HEAD traffic to `www.vastriqo.com` |
| Styling/UI | Tailwind CSS 4, global CSS/custom properties, styled-components 6, Ant Design 6, and the Ant Design Next registry |
| State management | React state/context. `CartProvider` is the only global application context found; auth and cart identifiers are persisted in `localStorage` |
| API architecture | Server-side Shopify Storefront GraphQL helper; Next route handlers as BMS/Shopify proxies; direct browser calls to BMS remain in legacy `front-auth` helpers |
| Authentication | Shopify Customer Account OAuth/PKCE in the browser, exchanged by BMS; BMS customer JWT/user summary stored in `localStorage` |
| Configuration | Server-only and public environment helper modules with built-in defaults; `.env.example` exists but is incomplete/inconsistent with current reads |
| Deployment | Node 20 Alpine multi-stage Docker image on port 3003; GitHub Actions deploys an ECR image to EC2 |
| Analytics | Google Analytics 4 plus Meta Pixel, with page-view and custom commerce/navigation event helpers |
| SEO | Per-route Next metadata, canonical/Open Graph/Twitter data, JSON-LD on home, and a dynamic sitemap implementation |

#### Important directories

- `app/`: page routes and server-side route handlers. The route handlers form a browser-facing proxy/BFF layer for auth, checkout, addresses, orders, wishlist, inquiries, newsletter, and related products.
- `components/collections/`: catalog grid, client-side search/filter/sort, collection presentation, and product-detail commerce behavior.
- `components/cart/`: the Shopify cart context and cart/checkout UI.
- `components/account/`: Shopify sign-in/callback and account pages.
- `components/common/`: header/footer, analytics trackers, shared promotional UI, and the wishlist login modal.
- `content/site-content.json`: checked-in navigation, footer, home-page copy, links, and content configuration.
- `lib/shopify/`: Storefront GraphQL catalog/cart client plus Shopify customer-account OAuth helpers.
- `lib/custom-auth-api.ts` and `lib/api/`: server-side BMS bridge and generic authenticated proxy behavior.
- `lib/front-auth.ts`: browser-local JWT/session helpers and older BMS-native user/profile/inquiry APIs; only part of this module is used by the current Shopify-backed customer flow.
- `public/`: checked-in Vastriqo banners, images, icons, and static illustrations.

#### Commerce and content source map

| Area | Current implementation | Current source/owner |
| --- | --- | --- |
| Home | Static hero/category/trust/newsletter content plus a featured product grid | Copy/navigation/assets are local JSON/files; featured products come from the BMS custom `home-page` collection, whose product records mirror Shopify |
| All products | Up to the first 100 products, then client-side filter/search/sort | Direct Shopify Storefront GraphQL |
| Collections landing | Product grid | Direct Shopify Storefront GraphQL; the page does not list BMS page/menu entities |
| Men/women/kids pages | Product grid for named collection slugs | BMS custom collection membership and ordering over locally mirrored Shopify products; no fallback to direct Shopify when that call fails |
| Product detail | Product by handle/ID, images, variants, options, price, description, selected metafields | Direct Shopify Storefront GraphQL |
| Related products | Up to eight product cards associated through BMS custom category memberships | BMS `shopify_products` plus `shopify_category_products`; source product identity remains Shopify |
| Search/filtering | Browser-side search, category, size, color, price, audience heuristics, and sorting over the already-fetched list | Vastriqo-owned UI logic; source fields are Shopify/BMS-mirrored Shopify data |
| Cart | Create/get/add/update/remove GraphQL cart operations; cart ID in `localStorage` | Shopify Storefront Cart API |
| Checkout | Requires BMS customer JWT/details; attaches the stored Shopify customer token as cart buyer identity; redirects to returned checkout URL | Shopify checkout; BMS supplies customer bridge/validity and may synchronize checkout name/phone |
| Payment | No payment processor is implemented inside Vastriqo | Shopify-hosted checkout/payment boundary |
| Sign in/sign up | `/login` starts Shopify OAuth PKCE; `/signup` renders the same login component | Shopify Customer Accounts for identity, BMS for the app JWT/local customer bridge |
| Profile | View/edit current local customer name/email/phone | BMS `customers`; normal edits do not write Shopify. Checkout-specific name/phone updates do write Shopify then BMS |
| Addresses | List/create/update/delete/set default | Live Shopify Admin GraphQL through Vastriqo route handler -> BMS |
| Orders | Last ten customer orders, totals, line items, fulfillment/tracking, addresses, and transaction status | Live Shopify Admin GraphQL through Vastriqo -> BMS; action-menu track/cancel items are UI links/actions, not implemented order mutations |
| Wishlist | List, toggle, pending-after-login behavior | BMS `wishlists` keyed to local Shopify-backed customer ID; product references are Shopify IDs/handles |
| Contact/inquiries | Public or authenticated multipart form; authenticated inquiry history page | BMS `inquiries` and S3, but current customer-to-inquiry identity linkage is inconsistent as documented below |
| Newsletter | Home form through Next proxy | BMS `newsletter_subscribers` |
| Banners/editorial pages | Home/about/why/FAQ/size/shipping/privacy/contact content | Checked-in JSON, TypeScript/JSX, and public assets |
| Storefront menus/pages | BMS provides public page/menu endpoints and admin screens | Not consumed by current Vastriqo source; current header/footer/pages remain checked-in content |
| SEO | Route metadata, product metadata fetched from Shopify, static route sitemap, intended internal-product sitemap data | Mostly Vastriqo/Shopify; the sitemap's configured BMS endpoint is not present in audited `bms-api` |
| Analytics | GA page views/events and Meta Pixel page views | Client scripts with IDs in source |

#### Shopify dependency map

```text
Vastriqo-owned
  presentation, routes, responsive UI, content files, filters/sorts/search,
  SEO composition, analytics events, cart UI, account UI

BMS-owned around Shopify
  local customer bridge + app JWT, wishlists, custom collection membership/order,
  related-product selection, inquiries/files, newsletter, wallet/referral data,
  mirrored product snapshot, optional storefront records

Shopify-owned today
  customer identity/OAuth, source catalog, products, handles, descriptions,
  prices, variants, availability, options, media, metafields, product SEO,
  carts, checkout URL/payment, buyer identity, addresses, orders,
  fulfillment/tracking, and order transaction state
```

Direct Vastriqo-to-Shopify calls are concentrated in `lib/shopify/index.ts` and `app/api/cart/*`. BMS-mediated Shopify calls cover login, customer data, order history, addresses, checkout identity, product synchronization, custom collection presentation, and related products.

## Current Architecture

```text
Browser
  |
  |-- bms-admin SPA -----------------------> deployed bms-api
  |                                              |
  |                                              |-- MySQL BMS database
  |                                              |-- AWS S3
  |                                              `-- Shopify Admin/OAuth APIs
  |
  `-- Vastriqo Next.js
         |-- server components/route handlers --> Shopify Storefront GraphQL
         |                                        (catalog, product, cart, checkout URL)
         |
         `-- server route handlers/direct legacy calls --> bms-api
                                                        |-- BMS MySQL storefront/customer records
                                                        `-- Shopify Admin/OAuth APIs
                                                            (customer, order, address, sync)
```

This is a verified three-party commerce path, not the proposed future architecture. Vastriqo frequently bypasses BMS for catalog detail and cart operations, while other storefront capabilities pass through BMS and may then call Shopify.

## Application Relationships

- `bms-admin` calls only `bms-api` through its shared Axios client.
- `bms-api` owns the current MySQL schema and S3 upload integration and acts as the Shopify Admin/customer bridge.
- `vastriqo` calls Shopify Storefront GraphQL directly from server functions/route handlers for products and carts.
- `vastriqo` calls `bms-api` for custom collections, related products, OAuth exchange/session bridging, customer profile, orders, addresses, wishlist, inquiries, and newsletter.
- Both the admin and storefront use the same BMS deployment/database endpoints today. No separate Vastriqo API or Vastriqo database exists.
- BMS's `storefront_pages` and `storefront_menus` are administrable and publicly readable, but there is no verified call from the Vastriqo source to those endpoints.

## Domain Ownership

| Domain | Current owner/authority | Current persistence/source | Used by | Notes |
| --- | --- | --- | --- | --- |
| CCA products | BMS | `products` plus size/color/image tables | bms-admin, CCA orders/invoices | Separate from the Shopify catalog |
| Vastriqo products/variants | Shopify | Shopify; partial mirror in `shopify_products` | Vastriqo, bms-admin collection tools | Shopify is authoritative; mirror is incomplete for independent commerce |
| Vastriqo categories/collections | Split | BMS custom category/membership/order plus Shopify product IDs | Vastriqo category/home/related sections, bms-admin | These are custom storefront groupings, not Shopify collection sync |
| CCA orders | BMS | `orders`, `order_items` | bms-admin | Production statuses and internal product/user references |
| Vastriqo carts/checkout | Shopify | Shopify Cart/Checkout | Vastriqo | Browser stores only cart ID; no BMS cart/checkout persistence |
| Vastriqo orders | Shopify | Queried live from Shopify | Vastriqo account | Not mapped into BMS `orders` |
| Payments | Shopify checkout for Vastriqo; BMS transaction ledger for CCA | Shopify vs `transactions` | Vastriqo vs bms-admin | The two meanings are unrelated |
| Admin/staff users | BMS | `users` | bms-admin and older front-auth paths | CCA-oriented roles |
| Vastriqo customers | Shopify identity + BMS bridge | Shopify plus `customers`/`customer_tokens` | Vastriqo, BMS | Two-system record with possible profile drift |
| Customer addresses | Shopify | Shopify only | Vastriqo account/checkout | CRUD is proxied through BMS |
| Authentication | Split | BMS JWT for admin; Shopify OAuth + BMS customer JWT for current storefront; older BMS-native user JWT also exists | All applications | Three overlapping token/identity paths |
| Wishlists | BMS | `wishlists`, referencing Shopify product IDs | Vastriqo | BMS-owned feature coupled to Shopify identifiers |
| Inquiries | BMS | `inquiries` and S3 | Vastriqo, bms-admin | Current Shopify customer identity is not correctly represented by `inquiries.user_id` |
| Invoices | BMS | `invoices`, `invoice_items` | bms-admin | CCA GST/seller workflow, not Vastriqo receipts |
| Expenses/payments/salary | BMS | `transactions` | bms-admin | CCA business ledger |
| Wallet/referrals | BMS | customer/user referral fields, settings, ledger, rewards | BMS APIs/admin | Backend implemented; no current Vastriqo wallet screen found |
| Storefront copy/navigation | Vastriqo source today; BMS has unused configurable records | JSON/JSX/assets; separately BMS page/menu tables | Vastriqo; bms-admin | No runtime connection between current Vastriqo content and BMS page/menu records |
| Newsletter | BMS | `newsletter_subscribers` | Vastriqo | Public subscribe endpoint |
| Product SEO | Shopify for product fields; Vastriqo for page composition | Shopify product SEO and Next metadata | Vastriqo | BMS mirror also stores SEO snapshots |
| Analytics | Vastriqo | Browser events to GA/Meta | Vastriqo | IDs and scripts are in source |

## Technology Stack

| Application | Frontend/backend | Core stack | Persistence/integrations |
| --- | --- | --- | --- |
| bms-admin | Frontend SPA | React 19, React Router 7, TypeScript, Vite, React Query, Axios, Ant Design 5, Tailwind 4 | Browser localStorage; BMS API |
| bms-api | Backend API | Node 20, Express 5, TypeScript, Knex, express-validator, JWT/bcrypt, Multer, Winston | MySQL, S3, Shopify Admin/OAuth; Redis code present but disabled |
| vastriqo | Full-stack storefront/BFF | Next.js 16 App Router, React 19, TypeScript, Tailwind 4, styled-components 6, Ant Design 6 | Shopify Storefront API, BMS API, Shopify customer/Admin bridge, browser localStorage, GA4, Meta Pixel |

## Authentication

### Admin

Admin login validates a BMS `users.password` bcrypt hash and returns a 24-hour JWT containing user ID, email, and user type. The admin stores it in `localStorage`. The API's global admin middleware validates the signature but does not enforce module permissions or user-role policies.

### Current Vastriqo customer flow

1. The browser creates PKCE verifier/state/nonce values and redirects to the hard-coded Shopify store-account authorize URL.
2. The callback validates state and sends code/verifier to a Vastriqo route handler.
3. The route handler calls BMS, which exchanges the code with Shopify, fetches the Shopify customer, upserts `customers`, stores the Shopify access token, and returns a BMS `type: CUSTOMER` JWT.
4. Vastriqo stores that BMS JWT and a normalized customer summary in `localStorage`.
5. Vastriqo sends the JWT to its own proxy route handlers or directly to BMS. BMS identifies the local `customers.id` from the token.

An older native BMS storefront flow also exists (`/front/auth/login`, `/front/auth/signup`, `/front/user/profile`) and uses `users`. The current `/signup` page does not use it; it renders the Shopify sign-in page. A separate cookie-based Shopify token/refresh helper also remains, but the current callback does not set those cookies.

## Data and Database Architecture

- There is one verified BMS MySQL connection. A separate Vastriqo database is not present.
- Knex services use a global connection and optional request transactions. There is no ORM schema generation.
- Migrations are versioned in `bms-api/src/db-migrations` and are run explicitly, including in the BMS deployment workflow.
- CCA commerce and Shopify/storefront bridge tables coexist in the same schema.
- The current BMS catalog mirror stores Shopify product snapshots and opaque JSON metadata, but it is not an independent product/variant/inventory system.
- The current BMS customer bridge stores Shopify IDs and access tokens but has no independent password/identity, address, cart, checkout, payment, fulfillment, or customer-order schema.

## Shopify Integration

Shopify is currently authoritative for the minimum set required to complete a Vastriqo purchase:

- product and variant identity;
- handles, product descriptions, images, prices, options, availability, and metafields;
- customer account authorization;
- customer addresses;
- cart state and checkout URL;
- checkout buyer identity and payment handoff;
- customer order, fulfillment, tracking, and transaction status.

BMS adds a local product mirror and custom merchandising relationships, but Shopify synchronization is one-way into BMS. The source contains no BMS-to-Shopify product writer, order importer, webhook consumer, inventory reconciliation, or deletion reconciliation. A sync queries products by Shopify update time and upserts returned records; no verified process removes local products deleted or unpublished in Shopify.

## Current Features

Verified end-user features include responsive home/editorial pages, catalog browsing, product search/filter/sort, product detail/metafields/variants, cart management, authenticated Shopify checkout redirect, Shopify customer sign-in, profile view/edit, address management, order history, wishlist, contact/inquiry submission, newsletter subscription, SEO metadata/sitemap code, and GA/Meta analytics.

Verified admin features include BMS user/product/order/invoice/transaction/seller/inquiry management; Shopify product synchronization and inspection; custom collection curation; storefront page/menu records; and wallet settings.

“Track Order” redirects to the Shopify order status URL when available. The displayed “Cancel Order” action has no mutation handler. Storefront page/menu records are not current Vastriqo features despite having admin and API implementations.

## Existing bms-api Design Conventions

These are current conventions that a future API may evaluate for reuse; listing them is not a recommendation to copy defects.

- Domain route/controller folders under separate `api/admin` and `api/front` trees.
- Thin route files that compose authentication, transaction creation, validation, and controller handlers.
- Controllers as static classes and shared services as static query-builder classes.
- Shared services accept an optional transaction for reads and a required transaction for writes.
- Knex migrations are the explicit schema source.
- Central configuration loaded from environment variables.
- Central success/failure helpers and a standard pagination helper.
- JWT middleware attaches decoded identity to the Express request.
- `express-validator` arrays end with a shared validation-result middleware.
- Soft deletion via `deleted_at` in several domains.
- Snake_case database/API fields and mostly singular module/service filenames.
- S3 uploads use memory-backed Multer input and return a public object URL.
- Shopify calls are isolated mostly in Shopify service modules and use normalized legacy IDs/GIDs.
- Docker produces compiled JavaScript and PM2 runs `dist/app.js`.

## Current Known Issues / Technical Debt

Only source-established, migration-relevant issues are listed.

### Architecture and domain boundaries

- BMS contains two customer-capable identities: `users` for native front auth and `customers` for Shopify-backed auth. Vastriqo currently uses `customers`, while some front modules still assume `users`.
- Authenticated inquiry submission/history always writes/filters `inquiries.user_id`, even when the token is a Shopify customer token whose ID belongs to `customers`. This can associate the wrong record or violate the users foreign key.
- CCA products/orders and mirrored Shopify products/customer orders are unrelated models with similar names. Reusing them without an explicit boundary would conflate production operations with retail commerce.
- Storefront pages and menus have complete admin/API paths but are disconnected from the current Vastriqo rendering/content system.
- The Shopify collection read is exposed under both a front route and a public path named `/admin/shopify-apis/collections/:slug`; Vastriqo uses the latter.

### API/configuration drift

- `vastriqo/lib/internal-products.ts` calls `/shopify-products/sitemap`, which does not exist in the audited BMS router. Dynamic product sitemap generation therefore has no verified backend.
- A generic `/products?gender=...` fallback client exists in Vastriqo, but the audited BMS API has no root `/products` route. Current gender pages bypass it because they always use configured BMS collection slugs.
- Vastriqo logout calls BMS `/auth/logout`, which is not an audited BMS route. The Next handler suppresses the error and the browser clears only its local session.
- `getCustomAuthMe()` calls another missing `/auth/me` route, although no current caller was found.
- Vastriqo `.env.example` names `SHOPIFY_CUSTOMER_ACCOUNT_SHOP_DOMAIN`, while runtime catalog code reads `SHOPIFY_STORE_DOMAIN`; it also omits several variables consumed by the source. One example redirect URI still references the CCA domain.
- `bms-admin` has no environment-based API setting and requires a source edit to switch away from its hard-coded production endpoint.

### Authentication and security

- Admin and Vastriqo app JWTs are stored in browser `localStorage`, increasing exposure to any successful script injection.
- BMS falls back to a known default JWT secret when configuration is absent; an encryption key is also hard-coded in source.
- The disabled Redis loader contains a credential-bearing connection URL in source.
- Shopify login code logs the PKCE verifier, token exchange payload, and customer payload. These logs can expose credentials or personal data.
- BMS enables CORS for every origin.
- The local Vastriqo Git remote URL contains an embedded credential. Its value is intentionally not reproduced here; it should be treated as exposed and removed/rotated.
- Admin file upload accepts arbitrary MIME types and has no explicit size limit. Inquiry uploads do restrict MIME type and size.
- BMS query logging prints full SQL with bindings, which can expose business/customer values in logs.

### Data/schema consistency

- The latest `users.user_type` migration allows `EMPLOYEE`, `ADMIN`, `VENDOR`, `FABRIC_SUPPLIER`, `CHECKER`, and `SALES_MAN`. Native front signup writes `CUSTOMER`, and the admin UI still offers `COMPANY`; both conflict with that migration-defined enum.
- Local BMS customer profile edits do not normally update Shopify, while checkout detail edits update both. The two records can drift.
- Shopify product sync has no verified webhook, deletion, unpublish, or inventory reconciliation path.
- Shopify access tokens are retained in `customer_tokens` using a hash of the BMS JWT as the lookup key; there is no verified cleanup, revocation, or refresh process for this active flow.

### Frontend and maintainability

- `bms-admin` fails a no-emit TypeScript check with six errors across invoice utilities, product list/form typing, Shopify selected-product state, and seller-detail query typing. `bms-api` and `vastriqo` pass the same check.
- `bms-admin` commits generated `build/` and `dist/` artifacts, increasing drift/noise risk.
- The Vastriqo source contains overlapping current and legacy auth/profile flows: Shopify-backed customers, native BMS front users, and unused cookie-refresh helpers with CCA-prefixed names.
- `/profile/update` and `/inquiries` use the native `users` front API, but they are not linked from the current account navigation and are incompatible with the Shopify-backed customer ID model. The primary `/profile` route uses `customers` instead.
- Product list/search/filter operations fetch at most 100 Shopify products and filter entirely in the browser; no pagination/search query is implemented for the full catalog.
- No automated tests were found in any application.
- Standalone tracked utilities/assets such as Adminer files, a referral Ruby script, and a collection HTML file have no verified runtime reference.

### Deployment/operations

- The BMS API Dockerfile and Vastriqo Dockerfile use `npm install` rather than lockfile-enforced `npm ci`.
- Both EC2 deployment workflows run broad Docker prune commands including volumes. The BMS command contains `-f2`, which does not match the standard force flag used by the Vastriqo workflow.
- The BMS deployment script hard-codes an AWS account/region registry path in its EC2 commands even though earlier workflow steps use secrets.
- There are no health checks, rollback steps, or test gates in the audited deployment workflows.

## Findings Relevant to the Future Vastriqo Platform

### Verified migration boundary

An independent Vastriqo backend would need to replace capabilities on both sides of the present split, not only the direct Storefront GraphQL calls. The current Shopify boundary includes catalog/variant/inventory data, customer identity, address book, cart, checkout/payment, order/fulfillment, and account order history. BMS currently fills supporting but Shopify-coupled roles through Shopify IDs, access tokens, mirrored JSON, and Admin GraphQL.

The minimum relationships visible in current code include:

- product -> variants/options/images/metafields/SEO/availability;
- collection -> ordered product membership and optional display override;
- customer -> sessions/tokens, addresses, wishlist, referral/wallet history;
- cart -> line -> product variant;
- order -> line items, amounts/discount/tax/shipping, addresses, fulfillment/tracking, and payment transaction status;
- storefront page/menu -> ordered content/product/category references.

### BMS capabilities with potentially reusable shape

The route/controller/service split, validation middleware, response/pagination helpers, Knex transaction pattern, migrations, S3 upload wrapper, logging abstraction, and Docker/runtime shape are existing conventions that a separate API could follow after deliberate review. Inquiry/newsletter, custom collection ordering, wishlist behavior, storefront page/menu records, and wallet/referral concepts are closer to storefront concerns than the core CCA production modules.

This is an observation about functional similarity, not a decision to share their code or database.

### CCA coupling that matters

The BMS `users` roles, seller bank/tax/signature data, HSN/GST invoices, expense/payment/salary transactions, fabric schema, internal products, and cutting/stitching/packing order states are explicitly CCA/production oriented. The existing `orders` table is not a starting representation of Shopify/Vastriqo customer orders without a domain redesign.

### Evidence for the future admin decision

Evidence making a unified admin easier:

- The current admin already contains Vastriqo-adjacent Shopify sync, custom collection, page/menu, inquiry, and wallet screens.
- React Query hooks and route components are separated by domain.
- Admin calls are centralized through one Axios client and endpoint constants.

Evidence making a unified admin harder:

- There is one hard-coded API base URL, one flat navigation, one undifferentiated admin JWT, and no business/tenant context or per-module authorization.
- Existing CCA and storefront modules write the same BMS database even though the intended Vastriqo database is separate.
- Labels, roles, order states, invoicing, seller details, and data contracts are strongly CCA-specific.
- The current storefront page/menu modules are not proven against the actual Vastriqo renderer.

Evidence still needed before deciding:

- administrator/persona overlap between CCA and Vastriqo;
- role and permission matrix for each business;
- whether cross-business users require one identity/session;
- API gateway/session strategy for two databases;
- operational ownership and deployment independence requirements;
- final Vastriqo admin domain list and whether current storefront admin screens satisfy it;
- acceptable failure/isolation behavior when one business API is unavailable.

## Verified / Inferred / Undecided

### Verified

- The three applications are independent projects and repositories.
- `bms-admin` calls BMS, not Shopify directly.
- Vastriqo calls both Shopify and BMS.
- Shopify is authoritative for current Vastriqo catalog details, customer identity, addresses, cart, checkout/payment handoff, and customer orders.
- BMS has one MySQL schema containing CCA operations and Shopify/storefront bridge records.
- BMS mirrors Shopify products and owns custom collection membership/order, but not independent variants/inventory/checkout/orders for Vastriqo.
- Current Vastriqo content is mostly source-controlled; BMS page/menu content is not consumed.
- There is no `vastriqo-api`, Vastriqo database, unified-admin switcher, or separate Vastriqo admin in this workspace.

### Inferred

- The custom BMS collections named `home-page`, `mens-collection`, `womens-collection`, and `kids-collection` are intended as Vastriqo merchandising collections because those exact slugs are requested by Vastriqo.
- The native BMS front-user/profile/inquiry screens are legacy or transitional relative to the current Shopify customer flow because current login/signup and primary profile routes do not use them and they are absent from current account navigation.
- Some committed standalone scripts, HTML, Adminer files, generated admin builds, and unused auth helpers are legacy/development artifacts because no runtime import or route reference was found.
- The BMS storefront page/menu models were intended for configurable storefront content, but their intended consumer cannot be established from current Vastriqo source.

### Undecided

- Whether Vastriqo should have a separate admin application.
- Whether `bms-admin` should become a multi-business admin with CCA/Vastriqo switching.
- The exact `vastriqo-api` module boundaries and schema.
- The final customer, administrator, and cross-business authentication model.
- Which BMS implementations should be shared, extracted, copied, or replaced.
- How Shopify data would be exported, reconciled, or cut over.
- Payment provider, checkout ownership, tax/invoice behavior, inventory rules, and fulfillment integrations for independent Vastriqo commerce.
- AWS topology, database hosting, networking, secrets, scaling, observability, and disaster recovery.

## Future Direction

The stated direction is conceptually:

```text
vastriqo -> vastriqo-api -> separate Vastriqo database
```

`vastriqo-api` is expected to consider the architectural/design conventions of `bms-api` while remaining a separate database boundary. This audit makes no choice about the admin application and proposes no migration sequence.

## Development Notes

### Repository/tooling

- Run npm commands inside the relevant child directory; there is no root npm command.
- Each project has its own `package-lock.json` and local `node_modules` in this working copy.
- All three child Git worktrees were clean at audit completion before this root README was added. Vastriqo's local `main` was nine commits behind its configured remote-tracking branch; this audit therefore describes the local checkout, not those unseen commits.

### bms-admin

- `npm run dev`: React Router/Vite development server, documented by its template README as port 5173.
- `npm run build`: React Router production build.
- `npm run start`: serves `build/server/index.js`.
- `npm run typecheck`: generates React Router types and runs TypeScript; an equivalent no-emit TypeScript audit currently fails as listed above.
- API selection is hard-coded in `app/lib/axios.ts`.

### bms-api

- `npm run build`: compiles `src` to `dist`.
- `npm run dev`: runs Nodemon against the already-compiled `dist/app.js`; it does not itself compile TypeScript.
- `npm run migrate:latest`: runs compiled migrations using `dist/config/knex.js`.
- Environment resolution is `.env.${NODE_ENV}`, then default dotenv lookup.
- Runtime requires MySQL configuration. Imported S3 upload code constructs its client at module load and requires AWS credentials even for a local process that does not intend to upload.

### vastriqo

- `npm run dev`, `npm run build`, `npm run start`, and `npm run lint` are defined.
- Next defaults to port 3000 locally; the Docker image starts on port 3003.
- Important current variables read by source include `CUSTOM_AUTH_API_BASE_URL`, `CUSTOM_AUTH_API_SECRET`, `NEXT_PUBLIC_API_BASE_URL`, `NEXT_PUBLIC_IMAGE_BASE_URL`, `NEXT_PUBLIC_SITE_URL`, `SHOPIFY_CLIENT_ID`, `SHOPIFY_REDIRECT_URI`, `SHOPIFY_STORE_DOMAIN`, `SHOPIFY_STOREFRONT_TOKEN`, and optional product-metafield identifiers.
- Built-in defaults point server-to-BMS traffic at `host.docker.internal:4000`, public API traffic at the same Docker host, images at the CCA BMS asset bucket, and products at the CCA Shopify store domain.

## Audit Metadata

- Audit date: 2026-08-23 (Asia/Kolkata).
- Scope: root plus all non-generated source/configuration in `bms-admin`, `bms-api`, and `vastriqo`.
- Local revisions: `bms-admin` `4003a39`; `bms-api` `422b28d`; `vastriqo` `7657a90`.
- Inspected: root/child Git metadata and status; package manifests/lock posture; Docker and deployment workflows; route configuration; page/route handlers; admin hooks/components/types; API routers/controllers/services/middleware/validation/types/migrations/loaders/integrations; storefront components/contexts/lib/config/content/assets references; environment examples; and runtime endpoint call sites.
- Generated/dependency directories (`node_modules`, `.next`, admin `build`/`dist`, API `dist`) were inventoried where relevant but not used as the source of architectural truth.
- Verification: `tsc --noEmit` passed for `bms-api` and `vastriqo`; it failed for `bms-admin` with the six errors summarized under technical debt. No dependencies were installed and no application source was modified.

## Final Audit Summary

### Current State

`bms-admin` and `bms-api` operate the CCA business system and also contain a growing Shopify/storefront bridge. `vastriqo` is a Next.js storefront whose presentation is owned locally but whose purchase-critical commerce remains primarily Shopify-backed, with BMS supplying selected persistence and mediation.

### Main Architectural Finding

Vastriqo currently has no single backend of record. It talks directly to Shopify for catalog/product/cart behavior and uses BMS for custom merchandising and customer-adjacent features, while BMS itself calls Shopify for identity, addresses, checkout support, orders, and product synchronization.

### Main Migration Boundary

The main future boundary is the Shopify-owned commerce aggregate: product/variant/inventory, customer identity/address, cart/checkout/payment, and order/fulfillment. The existing BMS mirror and CCA commerce tables do not yet constitute an independent Vastriqo commerce backend.

### Decisions Deferred

- Separate Vastriqo admin versus a unified multi-business admin.
- Exact Vastriqo API/schema and identity architecture.
- Which BMS conventions/modules to reuse.
- Shopify migration/cutover strategy.
- Independent checkout/payment/inventory/fulfillment design.
- AWS deployment architecture.
