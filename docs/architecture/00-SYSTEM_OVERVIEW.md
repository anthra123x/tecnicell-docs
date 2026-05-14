# System Overview — Tecnicell ERP ↔ Storefront

## What is Tecnicell?

Tecnicell is a mobile repair shop management system evolving into a full ERP/backoffice with an integrated online store. It consists of **two independent Next.js projects** sharing a single PostgreSQL database.

---

## Projects

### Project A: Tecnicell ERP (Backoffice)

| Attribute | Value |
|-----------|-------|
| Repo | `anthra123x/inventario-tecnicell` |
| Stack | Next.js 16, App Router, Prisma 5, PostgreSQL (Supabase), shadcn/ui, Tailwind v4 |
| Role | **Source of truth** for products, stock, orders, sales, repairs, clients, inventory |
| Auth | Supabase Auth (session-based) |
| Deployment | Vercel |

**Modules:**
- Inventory management (products, stock movements)
- POS Sales (point of sale)
- Repair management (diagnosis, parts, tracking)
- Order management (online orders lifecycle)
- Client registry
- Reports & analytics
- Ecommerce admin panel (product catalog, images, pricing, badges)

### Project B: Tecnicell Storefront (Online Store)

| Attribute | Value |
|-----------|-------|
| Repo | `anthra123x/tienda_online_tecnicell` |
| Stack | Next.js 16, App Router, Tailwind v4, shadcn/ui |
| Role | **Public-facing** online store — product catalog, cart, checkout, order tracking |
| Auth | None (public) |
| Deployment | Vercel |

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   Shared PostgreSQL Database                  │
│                                                              │
│  ┌──────────┐  ┌─────────────────┐  ┌──────────┐            │
│  │ products │  │ product_ecommerce│  │ orders   │            │
│  │ (stock)  │◄→│ (prices, badges) │  │ (status) │            │
│  └──────────┘  └─────────┬───────┘  └──────────┘            │
│                          │                                    │
│                   ┌──────┴──────┐                             │
│                   │product_media│                             │
│                   │ (images)    │                             │
│                   └─────────────┘                             │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ clients  │  │ sales    │  │ repairs  │  │ inventory_   │ │
│  │          │  │ sale_items│  │ repair_  │  │ movements   │ │
│  │          │  │          │  │ parts    │  │              │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │
│              ↑ Internal ERP tables (not shared)              │
└──────────────────────────┬──────────────────────────────────┘
                           │
            ┌──────────────┴──────────────┐
            │                              │
┌───────────▼──────────┐     ┌────────────▼───────────┐
│   Tecnicell ERP       │     │  Tecnicell Storefront   │
│   (Backoffice)        │     │  (Online Store)         │
│                       │     │                          │
│  ▪ Reads/Writes all   │     │  ▪ Reads products via    │
│    tables             │     │    GET /api/ecommerce/*  │
│  ▪ Manages catalog    │     │    (ERP) or direct DB    │
│  ▪ Processes orders   │     │  ▪ Creates orders via    │
│  ▪ POS, repairs, etc  │     │    POST /api/orders      │
│                       │     │    (local or ERP)       │
│  API endpoints:       │     │  ▪ Tracks orders via     │
│  GET /api/ecommerce/* │     │    GET /api/orders/[id]  │
│  POST /api/orders     │     │    (local or ERP)       │
│  GET /api/orders/[id] │     │                          │
└───────────────────────┘     └──────────────────────────┘
```

---

## Communication Patterns

| Pattern | Direction | Protocol |
|---------|-----------|----------|
| Product catalog read | Storefront → DB | HTTP API (ERP) or direct DB read (local) |
| Order creation | Storefront → DB | `POST /api/orders` (local Prisma) or ERP API |
| Order tracking | Storefront → DB | `GET /api/orders/[id]` (local Prisma) or ERP API |
| Stock updates | ERP → DB | Prisma writes (internal) |
| Catalog management | ERP → DB | Prisma writes (internal) |
| Order processing | ERP → DB | Prisma writes (internal) |

**Key constraint:** The storefront can write orders to the shared DB directly via its own API endpoints. Stock mutations are EXCLUSIVE to the ERP — the storefront never touches `products.stock`.

---

## Responsibilities Matrix

| Capability | Owner | Storefront Access |
|------------|-------|-------------------|
| Product base data (name, stock, salePrice) | ERP (Inventory module) | Read via ERP API or direct DB |
| Ecommerce settings (price, images, badges) | ERP (Ecommerce module) | Read via ERP API or direct DB |
| Product images | ERP (Ecommerce module) | Read via ERP API or direct DB |
| Order creation | Storefront (local) + ERP (admin) | Write via local `POST /api/orders` or ERP API |
| Order processing (status changes) | ERP (Orders module) | Read tracking via local API or ERP API |
| Order cancellation | ERP (via status update) | ERP handles; storefront reads result |
| Stock management | ERP (Inventory + Orders + Sales + Repairs) | None (read-only) — never write |
| Cart | Storefront (client-side) | 100% frontend |
| Checkout | Storefront → local API | Submit via local `POST /api/orders` |
| Payment | Storefront (future) | Future |

---

## Tech Stack (both projects)

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16 (App Router) |
| Language | TypeScript (strict mode) |
| Database | PostgreSQL (Supabase) |
| ORM | Prisma 5 |
| CSS | Tailwind CSS v4 |
| UI Library | shadcn/ui |
| Auth (ERP) | Supabase Auth (SSR) |
| Validation | Zod |
| Deployment | Vercel |

---

## Key Constraints

1. **Stock is 1:1** — `Product.stock` is the single source of truth. The storefront has NO stock field.
2. **Price is decoupled** — `ecommercePrice` can differ from `salePrice`. The API returns `price` which falls back automatically.
3. **No shared Prisma Client** — Each project defines its own Prisma schema. Shared tables are mapped via `@@map()`.
4. **Storefront can write orders** — The storefront writes `orders` and `order_items` via its local `POST /api/orders`. It NEVER writes to `products.stock` or `product_ecommerce`.
5. **Storefront is public** — All `/api/ecommerce/*` and `/api/orders/*` endpoints have NO authentication.
6. **Enums are UPPERCASE** — DB has legacy lowercase data; any direct query must normalize.
7. **Cascade deletes** — Deleting `EcommerceProduct` cascades to `ProductMedia`. Deleting `Product` cascades to `EcommerceProduct`.
8. **Soft deletes** — `Product.deletedAt` and `Client.deletedAt` are used instead of hard deletes. API queries always filter `deletedAt: null`.

---

## Current Architecture Risks

| Risk | Severity | Description |
|------|----------|-------------|
| Stock reservation window | High | `POST /api/orders` checks stock but does NOT reserve it. Stock is decremented only when `PENDING→CONFIRMED`. Between creation and confirmation, another order could oversell. |
| Trigger conflicts | High | A PostgreSQL trigger `trigger_product_ecommerce_updated_at` on `product_ecommerce` may conflict with Prisma's `@updatedAt` directive on writes. |
| Naming convention split | Medium | Old tables use camelCase columns, new ecommerce tables use snake_case. Raw SQL queries must account for this. |
| Two catalog endpoints | Low | `GET /api/products?ecommerce=true` is legacy and duplicates `GET /api/ecommerce/products`. The storefront should only use the latter. |
| No API rate limiting | Low | All API endpoints are public with no throttling. |
