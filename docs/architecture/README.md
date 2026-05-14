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
4. **Storefront never writes to DB directly** — All mutations through API
5. **Enums are UPPERCASE in schema** — Legacy lowercase data exists in DB
6. **Stock decrement happens ONLY on `PENDING → CONFIRMED`** — Not on order creation
7. **API endpoints are public** — No auth on any `/api/*` route

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

tecnicell-store/          ← Storefront project
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

## Tareas Pendientes — Storefront Agent

> Lee la documentación en el orden del Quick Start antes de ejecutar estas tareas. El catálogo online del ERP ya está listo. El storefront debe consumirlo.

### Tarea 1: Configurar modo API

El storefront tiene dos modos de obtener productos. Actualmente corre en **modo DB directa** (muestra todos los productos de la tabla `products` sin filtrar por visibilidad ecommerce).

**Qué hacer:**

```env
# En tu .env, agrega:
ECOMMERCE_API_URL="https://tecnicell.vercel.app"
```

Esto activa el modo API en `src/modules/products/service.ts`:

```
Modo DB directa (actual)              Modo API (target)
┌─────────────────────┐               ┌──────────────────────┐
│ products table       │               │ GET /api/ecommerce/  │
│ (todos los productos)│      →        │  products            │
│ Sin imágenes         │               │ (solo visibles,     │
│ Sin badges           │               │  con imágenes,      │
│ Sin ecommercePrice   │               │  badges, precios)   │
│ Sin slug             │               │  Con slug y metadatos│
└─────────────────────┘               └──────────────────────┘
```

### Tarea 2 (alternativa): Agregar modelos ecommerce a Prisma

Si no puedes usar el modo API, agrega estos modelos a `prisma/schema.prisma`:

```prisma
model ecommerce_products {
  id               String   @id
  productId        String   @unique @map("product_id")
  ecommercePrice   Float?   @map("ecommerce_price")
  compareAtPrice   Float?   @map("compare_at_price")
  visible          Boolean  @default(true)
  slug             String?  @unique
  shortDescription String?  @map("short_description")
  badges           String[]
  showStock        Boolean  @default(true) @map("show_stock")
  featured         Boolean  @default(false)
  media            product_media[]
  @@map("product_ecommerce")
}

model product_media {
  id          String  @id
  ecommerceId String  @map("ecommerce_id")
  url         String
  alt         String?
  isPrimary   Boolean @default(false) @map("is_primary")
  sortOrder   Int     @default(0) @map("sort_order")
  ecommerce   ecommerce_products @relation(fields: [ecommerceId], references: [id])
  @@map("product_media")
}
```

Luego modifica `getProducts()` en `src/modules/products/service.ts` para:
- Hacer join con `ecommerce_products`
- Filtrar `visible = true`
- Usar `ecommercePrice ?? salePrice` como `price`
- Incluir `media` (solo `isPrimary = true`) para la imagen
- Incluir `badges`, `tags`, `shortDescription`, `slug`

### Tarea 3: Verificar el catálogo

1. En el ERP, agrega un producto al catálogo online (Catálogo Online → Agregar producto)
2. Asígnale imagen, precio ecommerce, badges
3. Verifica que aparezca en el storefront
4. Verifica que los productos SIN `EcommerceProduct` o con `visible = false` NO aparezcan

### Tarea 4: Verificar tracking de pedidos

`GET /api/orders/[id]` ya existe en el ERP. Con `ECOMMERCE_API_URL` configurado, la página `src/app/(store)/order/[id]/page.tsx` debe funcionar automáticamente.

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
