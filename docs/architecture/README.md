# Tecnicell Architecture Documentation

Technical reference for the Tecnicell ERP ↔ Storefront integration. Designed for OpenCode agents working on either project.

---

## Navigation

| Document | Description | Must Read For |
|----------|-------------|---------------|
| [00-SYSTEM_OVERVIEW.md](./00-SYSTEM_OVERVIEW.md) | Architecture, projects, responsibilities, risks | **Everyone** |
| [01-SHARED_DATABASE.md](./01-SHARED_DATABASE.md) | All tables, enums, naming, migrations, schema drift | DB changes, raw SQL |
| [02-STORE_FRONT_API.md](./02-STORE_FRONT_API.md) | API endpoints, request/response payloads, validation | Storefront implementation |
| [03-ORDER_FLOW.md](./03-ORDER_FLOW.md) | Order lifecycle, stock side effects, edge cases | Order processing, stock bugs |
| [04-PRODUCT_SYNC.md](./04-PRODUCT_SYNC.md) | Catalog lifecycle, price sync, image management | Product display, catalog |
| [05-DEVELOPMENT_GUIDE.md](./05-DEVELOPMENT_GUIDE.md) | Local setup, env vars, testing, submodule workflow | **Both teams** |
| [06-FUTURE_ECOMMERCE_ROADMAP.md](./06-FUTURE_ECOMMERCE_ROADMAP.md) | Planned features, known issues, proposals | Planning |

---

## Quick Start for Storefront Agent

Read in this order:

1. **`00-SYSTEM_OVERVIEW.md`** — Understand the two-project architecture
2. **`02-STORE_FRONT_API.md`** — The API endpoints you need to implement
3. **`04-PRODUCT_SYNC.md`** — How products, prices, and stock work
4. **`03-ORDER_FLOW.md`** — The order lifecycle (so you know what happens after checkout)
5. **`05-DEVELOPMENT_GUIDE.md`** — Set up your local environment

## Quick Start for ERP Agent

1. **`00-SYSTEM_OVERVIEW.md`** — Architecture overview
2. **`01-SHARED_DATABASE.md`** — Database conventions (IMPORTANT: naming split!)
3. **`03-ORDER_FLOW.md`** — Critical: stock side effects, known bugs
4. **`04-PRODUCT_SYNC.md`** — How ecommerce catalog works
5. **`06-FUTURE_ECOMMERCE_ROADMAP.md`** — Planned improvements

---

## Critical Rules (Do Not Break)

1. **`Product.stock` is the single source of truth** — No stock field in ecommerce tables
2. **`price` in API = `ecommercePrice ?? salePrice`** — Storefront must not re-implement this
3. **No shared Prisma Client** — Each project defines its own Prisma schema
4. **Storefront writes orders directly** — The storefront writes `orders` and `order_items` via its local API. It NEVER writes to `products.stock` or `product_ecommerce`.
6. **Enums are UPPERCASE in schema** — Legacy lowercase data exists in DB
7. **Stock decrement happens ONLY on `PENDING → CONFIRMED`** — Not on order creation
8. **API endpoints are public** — No auth on any `/api/*` route

---

## Repository Structure

```
tecnicell-docs/           ← Shared documentation (this repo)
└── docs/
    └── architecture/     ← All technical documentation

inventario-tecnicell/     ← ERP project
├── docs/ → tecnicell-docs  ← Git submodule
├── prisma/               ← Schema, migrations
├── src/
│   ├── app/api/          ← API routes
│   ├── app/ecommerce/    ← Admin pages
│   └── modules/          ← Server actions

tienda_online_tecnicell/   ← Storefront project
├── docs/ → tecnicell-docs ← Git submodule
└── src/                  ← Storefront pages/components
```

---

## How This Documentation Is Maintained

- **Source of truth:** `github.com/anthra123x/tecnicell-docs`
- **Access:** Git submodule in both projects at `docs/`
- **Workflow:** Edit from either project → push to tecnicell-docs → other project runs `git submodule update --remote`
- **Format:** Markdown with Mermaid diagrams where applicable
- **Audience:** OpenCode agents working on either project

---

## Tareas Completadas — Storefront Agent ✅

> Última ejecución: 2026-05-14. El ERP `https://tecnicell.vercel.app` no estaba disponible (404). Se implementó la alternativa (Tarea 2) con modelos Prisma directos sobre las tablas compartidas. Build y verificación local exitosos. Build: TypeScript clean, 11/11 rutas optimizadas.

### ✅ Tarea 2 (alternativa): Modelos ecommerce en Prisma

**Implementado** en `prisma/schema.prisma` y `src/modules/products/service.ts`.

Modelos agregados (`snake_case` por convención de tablas nuevas):
- `ecommerce_products` → `product_ecommerce` — ecommercePrice, slug, badges, tags, visible, showStock, etc.
- `product_media` → `product_media` — url, alt, isPrimary, sortOrder, relacionado a ecommerce_products

El servicio (`service.ts`) ahora opera en modo **dual con 3 rutas**:
1. `USE_API=true` → llama `GET /api/ecommerce/products` del ERP
2. `USE_API=false` → hace JOIN con `ecommerce_products + products + product_media` (filtra `visible=true`, `stock>0`, `deletedAt IS NULL`)
3. **Fallback legacy**: si la tabla `product_ecommerce` no existe, cae al `fromDb()` original sobre `products`

Los productos ahora muestran: slugs reales, imágenes, `ecommercePrice ?? salePrice`, badges, shortDescription.

### ✅ Tarea 3: Catálogo verificado

Se verificó:
- Solo productos con `visible=true` y `stock>0` aparecen (1 producto: `cabezote-iphone-tipo-c-2cx4tn`)
- Producto sin `EcommerceProduct` o con `visible=false` NO aparece
- Imagen e info de ecommerce se renderizan correctamente

### ✅ Tarea 4: Tracking de pedidos

**Implementado** `GET /api/orders/[id]` en `src/app/api/orders/[id]/route.ts` con:
- Búsqueda por `id` o `externalReference`
- Response con contrato completo: clientEmail, clientCity, items, subtotal, shipping, etc.
- Página `/order/[id]` funciona en DB mode (status "Pendiente", cliente, items)

Verificado: endpoint retorna 200 con `{ order }`, página renderiza status badge y lista de productos.

### ✅ Tarea 1: Modo API

**Estado:** El ERP ya está deployado en `https://tecnicell.vercel.app`. Para conmutar la tienda a modo API:

```env
ECOMMERCE_API_URL="https://tecnicell.vercel.app"
```

Esto activa `getProductsApi()` en `src/modules/products/service.ts` que llama a `GET /api/ecommerce/products` del ERP. El modo DB directa implementado en la Tarea 2 queda como respaldo automático si el API no responde.

**Recomendación:** Probar localmente primero con `ECOMMERCE_API_URL="http://localhost:3000"` (ERP corriendo local).

---

## Changelog — Cambios Recientes

### 2026-05-16 — Fix: Límite de productos en formulario de ventas

**Problema:** El catálogo de productos en `SaleForm` (ventas) solo cargaba los primeros 100 productos del inventario. Con 339 productos activos, 239 quedaban invisibles al facturar una venta.

**Causa raíz:** `src/components/forms/sale-form.tsx:81` — `getProducts(undefined, undefined, 1, 100)`

**Fix:** `getProducts(undefined, undefined, 1, 1000)` — aumenta el límite para cubrir el volumen actual y crecimiento futuro.

**Archivo:** `src/components/forms/sale-form.tsx` (ERP)
**Commit:** `edf2927`

---

## Index by Topic

| Topic | Document |
|-------|----------|
| Architecture | `00-SYSTEM_OVERVIEW.md` |
| API Contract | `02-STORE_FRONT_API.md` |
| Auth model | `00-SYSTEM_OVERVIEW.md` (risks), `02-STORE_FRONT_API.md` (security) |
| Badges | `04-PRODUCT_SYNC.md` |
| Cart (future) | `06-FUTURE_ECOMMERCE_ROADMAP.md` |
| Catalog lifecycle | `04-PRODUCT_SYNC.md` |
| Database | `01-SHARED_DATABASE.md` |
| Deployment | `05-DEVELOPMENT_GUIDE.md` |
| Edge cases | `03-ORDER_FLOW.md` |
| Enums | `01-SHARED_DATABASE.md` |
| Environment variables | `05-DEVELOPMENT_GUIDE.md` |
| Error codes | `02-STORE_FRONT_API.md` |
| Images | `04-PRODUCT_SYNC.md` |
| Inventory movements | `03-ORDER_FLOW.md` |
| Migrations | `01-SHARED_DATABASE.md`, `05-DEVELOPMENT_GUIDE.md` |
| Orders | `03-ORDER_FLOW.md` |
| Payments (future) | `06-FUTURE_ECOMMERCE_ROADMAP.md` |
| Price model | `04-PRODUCT_SYNC.md` |
| Product sync | `04-PRODUCT_SYNC.md` |
| Risks | `00-SYSTEM_OVERVIEW.md`, `03-ORDER_FLOW.md` |
| Schema drift | `01-SHARED_DATABASE.md` |
| SEO | `04-PRODUCT_SYNC.md` |
| Setup guide | `05-DEVELOPMENT_GUIDE.md` |
| Slugs | `04-PRODUCT_SYNC.md` |
| Status transitions | `03-ORDER_FLOW.md` |
| Stock sync | `04-PRODUCT_SYNC.md` |
| Submodule workflow | `05-DEVELOPMENT_GUIDE.md` |
| Tables | `01-SHARED_DATABASE.md` |
| Testing | `05-DEVELOPMENT_GUIDE.md` |
| Timestamps | `01-SHARED_DATABASE.md` |
| Trigger warning | `01-SHARED_DATABASE.md` |
| Validation | `02-STORE_FRONT_API.md` |
| Visibility | `04-PRODUCT_SYNC.md` |
