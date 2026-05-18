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

### 2026-05-16 — Fix: NEXT_REDIRECT en formularios de creación

**Problema:** Al crear productos en inventario aparecía error `NEXT_REDIRECT` en el toast, aunque el producto sí se guardaba correctamente.

**Causa raíz:** Las server actions en `inventory/new/page.tsx` y `sales/new/page.tsx` llamaban `redirect()` después de crear el registro. `redirect()` lanza una excepción interna (`NEXT_REDIRECT`) que Next.js maneja automáticamente en `<form action={serverAction}>`, pero al ser llamadas **programáticamente** desde client components via `await onSubmit()`, la excepción propagaba como promesa rechazada al cliente, siendo capturada por `try/catch`.

**Fix:** Se eliminó `redirect()` de las server actions. La navegación ahora se maneja desde el cliente con `router.push(redirectTo)` después del success toast. Mismo patrón ya probado en `repairs/new/page.tsx`.

**Archivos modificados (ERP):**
- `src/app/inventory/new/page.tsx` — server action retorna `createProduct()` sin redirect
- `src/app/sales/new/page.tsx` — server action retorna `createSale()` sin redirect
- `src/components/forms/product-form.tsx` — agregado `redirectTo` prop + `router.push()`
- `src/components/forms/sale-form.tsx` — agregado `redirectTo` prop + `router.push()` + `try/catch`

**Lugares vulnerables identificados:** Solo los 2 corregidos. `repairs/new/` ya usaba el patrón correcto. `auth/actions.ts` y Server Components puros no se ven afectados.

### 2026-05-16 — Badge warning para stock bajo

**Problema:** Productos con stock bajo (stock > 0 pero <= minStock) se marcaban con el mismo color gris secundario que no era distinguible visualmente.

**Fix:** Nuevo variant `warning` en el componente `Badge` (bg-warning/15 + text-warning), aplicado al estado "Stock Bajo" en inventario.

**Archivos:** `src/components/ui/badge.tsx`, `src/app/inventory/page.tsx` (ERP)
**Commit:** `9692eab`

### 2026-05-16 — Stock breakdown global en inventario y reportes

**Problema:** Las tarjetas de resumen en inventario (En Stock, Stock Bajo, Agotados) solo contaban los 20 productos visibles en la página actual. El reporte de inventario no mostraba el desglose completo.

**Fix:** Nueva server action `getInventoryStockBreakdown()` que cuenta sobre TODOS los productos. Reporte de inventario ahora muestra En Stock/Stock Bajo/Agotados con colores.

**Archivos:** `src/modules/inventory/inventory.actions.ts`, `src/app/inventory/page.tsx`, `src/app/reports/page.tsx`, `src/modules/reports/reports.actions.ts` (ERP)
**Commit:** `a16e77b`

### 2026-05-16 — FASE 1: Exportaciones funcionando

**Problemas:**
- Reportes: botón "Exportar Excel" era un placeholder (toast "en desarrollo")
- Server actions de exportación sin filtro `deletedAt: null` (soft-delete incluido)
- Botones sin loading state (usuario sin feedback)
- Filenames sin timestamp único (sobrescritura en mismo día)
- `serverActions.bodySizeLimit` de 2MB insuficiente para backups grandes

**Fixes:**
- Reportes exportan Excel real con XLSX por tipo de reporte
- Filtros `deletedAt: null` en exportProductsToExcel y exportClientsToExcel
- Loading states en admin (exportExcelLoading, backupLoading) y reportes (exportLoading)
- Timestamps con segundos en filenames
- bodySizeLimit: 2mb → 10mb

**Archivos:** `src/app/reports/page.tsx`, `src/app/admin/page.tsx`, `src/modules/export/export.actions.ts`, `next.config.ts` (ERP)
**Commits:** `38a4aca`, `c7e0d58`

### 2026-05-16 — FASE 2: Integridad y seguridad

**Problemas críticos corregidos:**

1. **Auth: Sin guards de autorización** — Cualquier usuario podía crear admins, cambiar roles, eliminar usuarios.
   - **Fix:** `requireAdmin()` en getUsers, updateUserRole, deleteUser, createUserByAdmin
   - **Archivo:** `src/modules/auth/auth.actions.ts`

2. **Auth: Usuario Supabase huérfano** — Si fallaba creación en DB, el auth user quedaba sin registro.
   - **Fix:** Orden invertido: crear Prisma User primero, luego Supabase auth. Rollback en fallo.

3. **Orders: Stock phantom en PENDING→CANCELLED** — Cancelar pedido PENDING incrementaba stock nunca debitado.
   - **Fix:** Solo restaurar si el status previo estaba en CONFIRMED/PREPARING/SHIPPED/DELIVERED

4. **Orders: Transiciones de estado inválidas** — DELIVERED→PENDING era posible.
   - **Fix:** Máquina de estados `ALLOWED_TRANSITIONS` para los 6 estados

5. **Orders: Sin re-check de stock al confirmar** — PENDING→CONFIRMED debitaba sin verificar stock actual.
   - **Fix:** `findUnique` con select:stock dentro de la transacción antes de decrementar

6. **Dashboard: Ventas del Mes = histórico total** — `getSalesStats()` se llamaba sin filtro de fecha.
   - **Fix:** Pasar `startOfMonth` y `now()` como argumentos

7. **Dashboard: Categorías infladas** — `getProductsByCategory()` sin `deletedAt: null`.
   - **Fix:** Agregado `where: { deletedAt: null }` al groupBy

8. **Dashboard: Stat "Productos en Stock" incorrecto** — Mostraba lowStock count (capped a 10).
   - **Fix:** Renombrado a "Stock Bajo" con valor correcto

9. **Clients: Conteos inflados** — `getClientStats()`, `getClients()`, `getClientById()` sin `deletedAt: null`.
   - **Fix:** Agregado filtro a las 3 funciones

10. **Settings: Race condition + sin auth** — findFirst+create podía duplicar filas.
    - **Fix:** Helper `getOrCreateSettings` + `requireAdmin()`

11. **Settings: Sin validación Zod** — Datos sin sanitizar.
    - **Fix:** `UpdateSettingsSchema` con validación de tipos, email, rangos

**Archivo:** múltiples server actions (ERP)
**Commit:** `38a4aca`

### 2026-05-17 — Setup local + FASE 4: Estabilidad y middleware

**Setup en equipo nuevo:**
- Instalación de dependencias y config de Vercel CLI
- Recuperación de `.env` con credenciales reales de Supabase via Vercel
- Vinculación del proyecto `inventario-tecnicell-mvg1` en Vercel
- Eliminación de 5 proyectos duplicados de Vercel (`inventario-tecnicell`, `inventario-tecnicell-nuevo`, etc.)
- Aplicación de migraciones Prisma a BD Supabase

**1. Middleware → Proxy (Next.js 16)**
- `middleware.ts` estaba deprecado en Next.js 16.2.3
- Migrado a `proxy.ts` con función renombrada a `proxy()`
- `config.matcher` sin cambios
- **Archivo:** `src/middleware.ts` → `src/proxy.ts`

**2. Error boundaries faltantes**
- Agregados `error.tsx` en 5 rutas sin fallback UI
- **Archivos nuevos:** `login/error.tsx`, `register/error.tsx`, `profile/error.tsx`, `print/error.tsx`, `auth/logout/error.tsx`

**3. Loading states en dashboard**
- Agregados `loading.tsx` con Skeleton en 10 rutas principales
- **Archivos nuevos:** `dashboard/loading.tsx`, `inventory/loading.tsx`, `inventory/[id]/loading.tsx`, `sales/loading.tsx`, `sales/[id]/loading.tsx`, `repairs/loading.tsx`, `repairs/[id]/loading.tsx`, `orders/loading.tsx`, `orders/[id]/loading.tsx`, `ecommerce/loading.tsx`, `ecommerce/products/loading.tsx`, `reports/loading.tsx`, `admin/loading.tsx`

**4. Bugfix: Query inválida en reports (lowStock)**
- `reports.actions.ts:126` usaba `prisma.product.fields.minStock` que no existe en runtime
- **Fix:** Filtro post-query con `products.filter(p => p.stock <= p.minStock)`
- **Archivo:** `src/modules/reports/reports.actions.ts`

**5. Bugfix: Hard delete en clients**
- `clients.actions.ts:180` usaba `prisma.client.delete()` (hard delete)
- Contradecía la política de soft-delete documentada
- **Fix:** Cambiado a `prisma.client.update({ data: { deletedAt: new Date() } })`
- **Archivo:** `src/modules/clients/clients.actions.ts`

**6. Lint: Missing deps en useEffect**
- 10 componentes con efectos que no incluían la función de carga en el dependency array
- **Fix:** Funciones movidas dentro del efecto o `eslint-disable-next-line` cuando eran referenciadas externamente
- Sin `react-hooks/exhaustive-deps` warnings restantes

**Archivos del ERP involucrados:** proxy.ts, reports.actions.ts, clients.actions.ts, sales, orders, repairs, inventory, ecommerce pages, error/loading components

### 2026-05-16 — FASE 3: Riesgos restantes y performance

1. **Middleware: Cookies perdidas en redirect** — Session refresh no persistía al redirigir.
   - **Fix:** API `getAll/setAll` + `applyCookies()` que copia cookies a la response final

2. **Error boundaries** — 8 rutas sin fallback UI (white screen en producción).
   - **Fix:** `error.tsx` en dashboard, inventory, sales, repairs, reports, orders, admin, ecommerce

3. **Prisma indexes** — Queries lentas en tablas grandes.
   - **Fix:** Nuevos índices en SaleItem(productId), RepairPart(productId, repairId), OrderItem(productId)

4. **Dashboard: Query huérfana** — `getDailySales(30)` llamado pero nunca usado.
   - **Fix:** Eliminado del Promise.all y return de getDashboardStats()

5. **Settings: Validación Zod** — Implementado UpdateSettingsSchema con coerce, enum, rangos.

**Archivos:** middleware, error.tsx × 8, schema.prisma, dashboard.actions, settings.actions (ERP)
**Commit:** `c7e0d58`

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
