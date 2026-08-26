# Vastriqo Platform — Project Goal & Phased Direction

## 1. Project Goal

The goal of this project is to transform Vastriqo from a Shopify-dependent e-commerce storefront into a **fully independent e-commerce platform owned and operated by us**.

Vastriqo should ultimately have its own:

- customer-facing storefront;
- backend API;
- database;
- administration application;
- customer authentication and account system;
- product/catalog system;
- inventory system;
- cart and checkout flow;
- order management system;
- customer/order history;
- wishlist and other customer features;
- content and merchandising management;
- integrations required to operate the business.

The resulting system should not depend on Shopify as the underlying commerce platform.

The target core architecture is:

```text
                         VASTRIQO PLATFORM

                    ┌──────────────────────┐
                    │      Vastriqo        │
                    │   Customer Store     │
                    │   vastriqo.in        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │     vastriqo-api     │
                    │    Backend / API     │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │     Vastriqo DB      │
                    │   Commerce Data      │
                    └──────────────────────┘
```

The administration system will be a separate application:

```text
                    ┌──────────────────────┐
                    │   vastriqo-admin     │
                    │ admin.vastriqo.in     │
                    └──────────┬───────────┘
                               │
                               ▼
                         vastriqo-api
```

Therefore, the fundamental Vastriqo architecture will eventually be:

```text
vastriqo.in
      │
      ▼
vastriqo-api
      │
      ▼
Vastriqo DB

admin.vastriqo.in
      │
      ▼
vastriqo-api
      │
      ▼
Vastriqo DB
```

Shopify will not sit between these systems.

---

# 2. Shopify Independence

The goal is **not** simply to hide Shopify behind `vastriqo-api`.

This would not be considered the completed architecture:

```text
Vastriqo
    ↓
vastriqo-api
    ↓
Shopify
```

Instead, the target is:

```text
Vastriqo
    ↓
vastriqo-api
    ↓
Vastriqo DB
```

Shopify may temporarily remain involved during the migration, but it must eventually cease to be the source of truth for Vastriqo's core commerce operations.

The future Vastriqo backend should therefore own the data and business logic necessary to operate the store independently.

This includes, subject to detailed design later:

- products;
- product variants;
- categories/collections;
- product media and merchandising information;
- pricing;
- inventory;
- customers;
- customer authentication;
- customer addresses;
- carts;
- cart items;
- checkout;
- orders;
- order items;
- payment state/integration;
- fulfillment state;
- shipping information;
- customer order history;
- wishlists;
- Vastriqo-specific storefront/content data;
- other commerce-related data currently delegated to Shopify.

The exact schema and module boundaries are intentionally **not defined yet**.

---

# 3. Separate Vastriqo Database

Vastriqo will have a dedicated database.

The existing BMS database must not become the long-term Vastriqo commerce database.

The target separation is:

```text
CCA
 │
 ▼
bms-api
 │
 ▼
BMS DB


Vastriqo
 │
 ▼
vastriqo-api
 │
 ▼
Vastriqo DB
```

These databases should represent separate business/domain boundaries.

The existing BMS database currently contains both CCA operations and several Vastriqo/Shopify bridge records. Those records will be evaluated individually during the migration.

We must not blindly copy the existing BMS schema into the new database.

Instead, the future Vastriqo database should be designed around **Vastriqo's actual commerce requirements**.

---

# 4. Separate Vastriqo API

A new backend application named:

```text
vastriqo-api
```

will be created.

It should follow the useful architectural conventions established by `bms-api`, where appropriate, rather than introducing an unrelated backend style.

However, `vastriqo-api` must remain an independent application with:

- its own codebase;
- its own configuration;
- its own database;
- its own migrations;
- its own API contracts;
- its own deployment;
- its own authentication/authorization model;
- its own domain boundaries.

The intention is to **reuse architectural knowledge**, not create a hidden dependency on `bms-api`.

The exact framework/module structure will be decided during the architecture phase.

---

# 5. Separate Vastriqo Admin

Vastriqo administration should eventually be handled by a dedicated application:

```text
vastriqo-admin
```

hosted at:

```text
admin.vastriqo.in
```

This application will be responsible for managing Vastriqo-specific operations through `vastriqo-api`.

It should eventually provide the administration capabilities required to operate Vastriqo without relying on Shopify Admin.

Examples may include:

- dashboard;
- products;
- variants;
- inventory;
- categories/collections;
- merchandising;
- orders;
- customers;
- customer accounts;
- discounts/promotions where required;
- content/storefront configuration;
- media;
- payments/order state;
- fulfillment/shipping;
- settings;
- other Vastriqo-specific operations.

This is a future capability list, not a finalized module list.

The exact admin scope will be defined later after the existing Shopify/BMS functionality is mapped against the desired Vastriqo platform.

---

# 6. Relationship With CCA / BMS

CCA and Vastriqo are related businesses/projects but should have **clear technical boundaries**.

CCA will continue to use:

```text
bms.customcraftapparel.in
        │
        ▼
    bms-admin
        │
        ▼
     bms-api
        │
        ▼
      BMS DB
```

Vastriqo will use:

```text
admin.vastriqo.in
        │
        ▼
  vastriqo-admin
        │
        ▼
   vastriqo-api
        │
        ▼
   Vastriqo DB
```

The two systems should not be merged simply because they are owned by the same organization.

In particular:

- Vastriqo should not use the BMS database as its primary commerce database.
- Vastriqo should not depend on BMS APIs for core commerce functionality in the final architecture.
- CCA-specific business logic should not be moved into `vastriqo-api`.
- Vastriqo-specific commerce logic should not unnecessarily be added to `bms-api`.

Where information genuinely needs to cross the boundary, an explicit integration can be designed later.

---

# 7. Connection Between the Two Admin Applications

Although the admin applications will be separate technically, they should provide convenient navigation between each other.

The intended user experience is approximately:

```text
bms.customcraftapparel.in
        │
        │ Admin selector
        ▼
admin.vastriqo.in
```

and:

```text
admin.vastriqo.in
        │
        │ Admin selector
        ▼
bms.customcraftapparel.in
```

For example, both applications may have a header selector:

```text
Current Admin: [ Custom Craft Apparel ▼ ]

Custom Craft Apparel
Vastriqo
```

and:

```text
Current Admin: [ Vastriqo ▼ ]

Vastriqo
Custom Craft Apparel
```

Selecting another system may simply redirect the administrator to the corresponding independently hosted application.

The selector is therefore a **navigation/UX connection**, not necessarily a shared application or shared backend.

---

# 8. Authentication Between Admin Applications

The two admin applications should remain independently secured.

Initially, it is acceptable for each application to have its own authentication system.

A seamless cross-application authentication/SSO mechanism may be introduced later if it provides meaningful operational value.

We should not introduce shared authentication merely to make the dropdown work.

If SSO is eventually required, it should be designed explicitly rather than sharing JWT secrets, database sessions, or authentication tables in an ad-hoc way.

The authentication architecture is therefore an **open design decision**.

---

# 9. Migration Rather Than Rewrite

This project should be treated as a **controlled migration of an existing production system**, not as an opportunity to blindly rewrite everything.

The current Vastriqo storefront already contains significant functionality and design work.

The goal is to preserve the working customer experience while replacing the underlying commerce infrastructure.

The migration should therefore identify:

```text
CURRENT
Shopify
BMS
Vastriqo
   ↓
Understand
   ↓
Design
   ↓
Build
   ↓
Migrate
   ↓
Validate
   ↓
Cut over
   ↓
Independent Vastriqo
```

Existing behavior should be preserved where appropriate, while architectural weaknesses discovered during the audit can be addressed when designing the new system.

We should not assume that every existing implementation deserves to be copied.

---

# 10. Phased Project Direction

The project will be executed in deliberate phases.

## Phase 0 — Current-System Audit

**Status: Complete**

Understand:

- `bms-admin`;
- `bms-api`;
- `vastriqo`;
- current databases;
- Shopify dependencies;
- authentication;
- data ownership;
- API relationships;
- current technical debt;
- existing CCA/Vastriqo boundaries.

The output is the current-state audit:

```text
PROJECT-AUDIT.md
```

This document describes what exists today and should not be treated as the implementation plan.

---

# Phase 1 — Target Architecture & Requirements

**Status: Complete (2026-08-26)**

The completed decision record is `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`. It preserves the existing storefront as the functional reference, closes the conceptual architecture and permits Phase 2 detailed design without authorizing implementation.

Define exactly what the independent Vastriqo platform must become.

This phase should establish:

- final system boundaries;
- `vastriqo-api` responsibilities;
- `vastriqo-admin` responsibilities;
- Vastriqo database domains;
- customer identity model;
- administrator identity model;
- product/catalog model;
- inventory model;
- cart/checkout model;
- order model;
- payment integration boundary;
- fulfillment/shipping boundary;
- content/merchandising model;
- required external services;
- relationship, if any, with CCA/BMS;
- temporary versus permanent Shopify dependencies;
- migration and cutover principles.

No major implementation should begin until this target architecture is sufficiently defined.

---

# Phase 2 — Detailed Data & API Design

**Status: Ready to start**

Design the new Vastriqo backend before building the complete application.

This includes:

- database schema;
- entities and relationships;
- migrations;
- API resource structure;
- authentication;
- authorization;
- validation;
- error/response conventions;
- file/media handling;
- integrations;
- transaction boundaries;
- logging/observability;
- testing strategy.

The design should be reviewed against the current audit before implementation.

---

# Phase 3 — Build Vastriqo API & Database Foundation

Create:

```text
vastriqo-api
```

and its dedicated database.

Establish the production-quality foundation first:

- project structure;
- configuration;
- database;
- migrations;
- authentication;
- authorization;
- common API infrastructure;
- validation;
- error handling;
- logging;
- file/media infrastructure;
- testing;
- local development environment.

Then implement commerce modules progressively.

---

# Phase 4 — Build Vastriqo Admin

Create:

```text
vastriqo-admin
```

and connect it to:

```text
vastriqo-api
```

The admin should be designed around the new Vastriqo domain model rather than copying the existing Shopify admin screens blindly.

The existing `bms-admin` UI/design system can be used as a reference where appropriate.

At the end of this phase, Vastriqo should have an operational internal administration system independent of Shopify Admin for the capabilities already migrated.

---

# Phase 5 — Migrate Storefront

Gradually move:

```text
vastriqo.in
```

from its current Shopify/BMS dependencies toward:

```text
vastriqo.in
      ↓
vastriqo-api
      ↓
Vastriqo DB
```

The customer-facing behavior should remain stable wherever possible.

Shopify dependencies should be removed progressively rather than all at once.

---

# Phase 6 — Data Migration

Determine what existing data needs to be migrated from:

- Shopify;
- BMS;
- existing Vastriqo-local sources.

Examples include:

- products;
- variants;
- categories;
- inventory;
- required product attributes/specifications;
- product media, pricing, publication and availability data;
- relevant collection membership/order and merchandising data; and
- only other data explicitly included by the authoritative Phase 1 migration register.

Approved Phase 1 clean-start decision: do **not** migrate current test-era customers, customer credentials, addresses, wishlists, historical orders, newsletter subscribers, or support/inquiry records. Do not create a dedicated historical-order archive for that data. New native customer and order records begin after Vastriqo takes ownership.

Existing legal/financial information may be carried forward only where already available and safely/straightforwardly reusable for an included requirement; it is not a dedicated historical migration stream.

Migration must include validation and reconciliation.

No data should be assumed to map one-to-one simply because similarly named entities exist.

---

# Phase 7 — External Commerce Integrations

Replace Shopify-owned external responsibilities with appropriate independent integrations.

Depending on the final requirements, this may include:

- payment gateway;
- shipping provider;
- email;
- SMS/notifications;
- tax/GST services;
- analytics;
- storage/CDN;
- other required services.

The external service architecture will be decided during the design phase.

The important principle is:

> Shopify should not remain necessary for Vastriqo to operate its core commerce system.

---

# Phase 8 — Validation, Parallel Operation & Cutover

Before removing Shopify dependency:

- test storefront functionality;
- test admin functionality;
- test authentication;
- test catalog;
- test inventory;
- test cart;
- test checkout;
- test payments;
- test orders;
- test customer accounts;
- test fulfillment;
- test notifications;
- validate migrated data;
- verify production performance;
- verify security;
- run controlled parallel/transition operation where appropriate.

Only after the new system is proven should Shopify be removed from the critical path.

---

# Phase 9 — Shopify Removal

Once the independent Vastriqo system is stable:

- remove Shopify runtime dependencies;
- remove Shopify-specific application code;
- remove obsolete BMS Shopify bridge dependencies;
- remove unused Shopify synchronization;
- remove obsolete credentials/secrets;
- verify that Vastriqo can operate independently;
- retain historical data/integration information only where legitimately required.

The final architecture should no longer require Shopify to run Vastriqo's core e-commerce operation.

---

# 11. Definition of Success

The project is successful when:

```text
                         VASTRIQO

                  ┌──────────────────┐
                  │  vastriqo.in     │
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │   vastriqo-api   │
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │   Vastriqo DB    │
                  └──────────────────┘


                  ┌──────────────────┐
                  │ admin.vastriqo.in│
                  └────────┬─────────┘
                           │
                  ┌────────▼─────────┐
                  │ vastriqo-admin   │
                  └────────┬─────────┘
                           │
                           ▼
                      vastriqo-api
```

and:

- Vastriqo's core commerce data is owned by the Vastriqo database.
- Vastriqo's storefront is powered by `vastriqo-api`.
- Vastriqo administration is powered by `vastriqo-api`.
- Shopify is no longer required for core commerce operations.
- CCA continues to operate independently through BMS.
- CCA and Vastriqo can be navigated between conveniently through their separate admin applications.
- The two systems can be deployed and evolved independently.
- The system is maintainable and production-ready rather than merely being a Shopify replacement prototype.

---

# 12. What This Document Does Not Decide Yet

Status note (2026-08-26): this was the original direction-stage list. Phase 1 has now decided or bounded the conceptual items in `PHASE-1-CLOSURE-AND-DECISION-REGISTER.md`, including the clean-start/no-history-import scope. Exact physical schema/API/framework/provider/deployment choices remain Phase 2 or later decisions as appropriate; they are no longer Phase 1 blockers.

The following remain deliberately open until the architecture phase:

- exact database schema;
- exact API modules;
- exact admin modules;
- framework details for `vastriqo-api`;
- framework/details for `vastriqo-admin`;
- shared design system strategy;
- authentication/SSO;
- payment provider;
- shipping provider;
- tax/GST architecture;
- inventory rules;
- order state machine;
- fulfillment model;
- Shopify cutover strategy;
- AWS architecture;
- CI/CD architecture;
- monitoring/observability;
- secrets management;
- whether any BMS functionality should communicate with Vastriqo;
- whether any data should be shared between CCA and Vastriqo.

These decisions should be made deliberately during subsequent phases rather than being assumed during implementation.

---

# 13. Project Principle

The central principle of the project is:

> **Build Vastriqo as an independent commerce platform, not as a new interface over Shopify.**

Existing systems and code should be treated as references and migration sources.

They should not automatically become dependencies of the new platform.

The final system should have clear ownership of its own data, business logic, APIs, administration, and customer experience.
