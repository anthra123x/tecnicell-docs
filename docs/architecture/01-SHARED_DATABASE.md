# Shared Database — Tables, Models & Conventions

## Overview

Both projects share a single PostgreSQL database hosted on Supabase. Each project maintains its own Prisma schema file — they must NOT share a Prisma Client.

Tables are divided into two categories: **shared** (both projects read/write according to their responsibilities) and **ERP-internal** (only ERP accesses).

**Write rules:**
- ERP writes to ALL tables (products, orders, ecommerce, sales, repairs, etc.)
- Storefront writes ONLY to `orders` and `order_items` (via its local `POST /api/orders`)
- Storefront NEVER writes to `products`, `product_ecommerce`, or `product_media`

---

## Database Connection

Supabase provides two connection strings:

- `DATABASE_URL` — Primary connection (with connection pooling via Supavisor)
- `DIRECT_URL` — Direct connection (for migrations, long-running queries)

Both projects use the same credentials. Environment variables are documented in `05-DEVELOPMENT_GUIDE.md`.

---

## Shared Tables (Both Projects)

### `products` (Prisma: `Product`)

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT (cuid) | PK |
| `name` | TEXT | Product name |
| `description` | TEXT | Nullable |
| `category` | TEXT | Enum: `ACCESSORY`, `REPAIR_PART`, `DEVICE`, `OTHER` |
| `stock` | INTEGER | **Single source of truth** — default 0 |
| `minStock` | INTEGER | Alert threshold — default 5 |
| `purchasePrice` | DOUBLE PRECISION | Cost price |
| `salePrice` | DOUBLE PRECISION | Internal sale price |
| `supplier` | TEXT | Nullable |
| `createdAt` | TIMESTAMP(3) | |
| `updatedAt` | TIMESTAMP(3) | |
| `deletedAt` | TIMESTAMP(3) | Soft delete — API always filters `deletedAt IS NULL` |

**Indexes:** `category`, `stock`, `name`, `supplier`, `deletedAt`

### `product_ecommerce` (Prisma: `EcommerceProduct`)

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT (cuid) | PK |
| `product_id` | TEXT | FK → `products.id`, UNIQUE, CASCADE delete |
| `ecommerce_price` | DOUBLE PRECISION | Nullable — storefront price (falls back to `salePrice`) |
| `compare_at_price` | DOUBLE PRECISION | Nullable — strikethrough price |
| `visible` | BOOLEAN | Default true — if false, hidden from storefront |
| `featured` | BOOLEAN | Default false — highlight on storefront |
| `published_at` | TIMESTAMPTZ | Set when first made visible |
| `slug` | TEXT | UNIQUE — URL-friendly identifier |
| `meta_title` | TEXT | SEO title |
| `meta_description` | TEXT | SEO description |
| `short_description` | TEXT | Storefront description |
| `long_description` | TEXT | Rich HTML content |
| `badges` | TEXT[] | Array of badge strings: `["Nuevo", "Oferta"]` |
| `tags` | TEXT[] | Array of search tags |
| `sort_order` | INTEGER | Default 0 — display order |
| `show_stock` | BOOLEAN | Default true — show/hide stock count on storefront |
| `created_at` | TIMESTAMPTZ | |
| `updated_at` | TIMESTAMPTZ | Auto-updated via PostgreSQL trigger |

**Indexes:** `visible`, `featured`, `sort_order`, `published_at`
**Relations:** 1:1 with `products`, 1:N with `product_media`

> ⚠️ **Warning:** `product_ecommerce` has a PostgreSQL trigger `trigger_product_ecommerce_updated_at` that auto-sets `updated_at` on row update. This may conflict with Prisma's `@updatedAt` which also sets the field during writes. The trigger will override Prisma's value.

### `product_media` (Prisma: `ProductMedia`)

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT (cuid) | PK |
| `ecommerce_id` | TEXT | FK → `product_ecommerce.id`, CASCADE delete |
| `url` | TEXT | Image URL |
| `alt` | TEXT | Alt text |
| `width` | INTEGER | Image width |
| `height` | INTEGER | Image height |
| `sort_order` | INTEGER | Display order (0-indexed) |
| `is_primary` | BOOLEAN | Primary image for listings |
| `storage_provider` | TEXT | Default `"placeholder"` — future: `"supabase"`, `"cloudinary"` |
| `storage_id` | TEXT | Provider-specific storage ID |
| `created_at` | TIMESTAMPTZ | |

**Indexes:** `ecommerce_id`, `is_primary`

### `orders` (Prisma: `Order`)

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT (cuid) | PK |
| `client_name` | TEXT | |
| `client_phone` | TEXT | |
| `client_city` | TEXT | Nullable |
| `client_email` | TEXT | Nullable |
| `client_address` | TEXT | Nullable |
| `client_notes` | TEXT | Customer notes |
| `status` | TEXT | Enum: `PENDING`, `CONFIRMED`, `PREPARING`, `SHIPPED`, `DELIVERED`, `CANCELLED` |
| `subtotal` | DOUBLE PRECISION | Default 0 |
| `shipping` | DOUBLE PRECISION | Default 0 |
| `total` | DOUBLE PRECISION | Default 0 |
| `external_reference` | TEXT | Nullable, UNIQUE — storefront order ID |
| `internal_notes` | TEXT | Staff-only notes |
| `created_at` | TIMESTAMP(3) | |
| `updated_at` | TIMESTAMP(3) | |

**Indexes:** `status`, `created_at`, `client_phone`, `external_reference`

### `order_items` (Prisma: `OrderItem`)

| Column | Type | Notes |
|--------|------|-------|
| `id` | TEXT (cuid) | PK |
| `order_id` | TEXT | FK → `orders.id`, CASCADE delete |
| `product_id` | TEXT | FK → `products.id` |
| `quantity` | INTEGER | |
| `unit_price` | DOUBLE PRECISION | Price at time of order |
| `total` | DOUBLE PRECISION | `quantity * unit_price` |

**Index:** `order_id`

---

### Storefront Prisma Models

The storefront defines its own Prisma models for these shared tables (for direct DB reads/writes):

| Table | Storefront Model | Notes |
|-------|-----------------|-------|
| `products` | `products` | Read-only, maps via `@@map("products")` |
| `product_ecommerce` | `ecommerce_products` | Read-only, maps via `@@map("product_ecommerce")` |
| `product_media` | `product_media` | Read-only, maps via `@@map("product_media")` |
| `orders` | `orders` | Read/Write, maps via `@@map("orders")` |
| `order_items` | `order_items` | Read/Write, maps via `@@map("order_items")` |

The storefront also has its own analytics table (not shared with ERP):

| Table | Storefront Model | Purpose |
|-------|-----------------|---------|
| `product_views` | `product_views` | Product page view tracking (storefront analytics) |

---

## ERP-Internal Tables (NOT accessible to storefront)

| Table | Prisma Model | Purpose |
|-------|-------------|---------|
| `users` | `User` | Staff accounts, roles (ADMIN/EMPLOYEE) |
| `clients` | `Client` | Client registry (POS + repairs) |
| `sales` | `Sale` | POS point-of-sale transactions |
| `sale_items` | `SaleItem` | POS line items |
| `repairs` | `Repair` | Repair jobs |
| `repair_parts` | `RepairPart` | Parts used in repairs |
| `inventory_movements` | `InventoryMovement` | Stock entry/exit audit trail |
| `system_settings` | `SystemSettings` | App configuration |

---

## Naming Convention

**Critical:** The database has TWO coexisting naming conventions:

| Convention | Applies to | Example |
|-----------|------------|---------|
| **camelCase** | All tables created before 2026-05-13 | `products.createdAt`, `sales.total`, `clients.phone` |
| **snake_case** | Tables created on 2026-05-13+ | `product_ecommerce.ecommerce_price`, `product_media.is_primary` |

The ecommerce models use explicit `@map()` directives in Prisma to bridge this:

```prisma
model EcommerceProduct {
  productId      String   @map("product_id")
  ecommercePrice Float?   @map("ecommerce_price")
  createdAt      DateTime @map("created_at")
  // ...
  @@map("product_ecommerce")
}
```

**If you write raw SQL**, you must know which convention each table uses.

---

## Enums

| Enum | Values | DB Storage | Used In |
|------|--------|-----------|---------|
| `UserRole` | `ADMIN`, `EMPLOYEE` | TEXT | `users.role` |
| `ProductCategory` | `ACCESSORY`, `REPAIR_PART`, `DEVICE`, `OTHER` | TEXT | `products.category` |
| `RepairStatus` | `RECEIVED`, `IN_PROGRESS`, `READY`, `DELIVERED`, `CANCELLED` | TEXT | `repairs.status` |
| `PaymentMethod` | `CASH`, `CARD`, `TRANSFER` | TEXT | `sales.payment_method` |
| `MovementType` | `ENTRY`, `EXIT` | TEXT | `inventory_movements.type` |
| `OrderStatus` | `PENDING`, `CONFIRMED`, `PREPARING`, `SHIPPED`, `DELIVERED`, `CANCELLED` | TEXT | `orders.status` |

> ⚠️ **Legacy data warning:** The DB contains orders with lowercase status values (`preparing`, `shipped`, `delivered`) created by a previous version of the storefront. Prisma enum validation will fail on these rows. If you query `orders` directly with raw SQL, normalize with `UPPER(status)`.

---

## Migration History

| Migration | Date | Description |
|-----------|------|-------------|
| `20260411035645_init` | 2026-04-11 | Initial schema: all core tables, enums, FKs |
| `20260414044055_add_system_settings` | 2026-04-14 | `system_settings` table; additional indexes |
| `20260415031043_remove_mercado_pago` | 2026-04-15 | Removed `MERCADO_PAGO` from PaymentMethod enum |
| `20260416034452_add_soft_delete_and_indexes` | 2026-04-16 | `deletedAt` on clients/products; more indexes |
| `20260513000000_add_ecommerce_models` | 2026-05-13 | `product_ecommerce` + `product_media` tables (hand-written migration) |

### Migration Policy

- New migrations run via `prisma migrate deploy` in CI/CD
- For development: `prisma db push` is acceptable but creates schema drift
- The ecommerce migration (`20260513...`) is **hand-written** (not Prisma-generated) — be careful when regenerating

---

## Schema Drift (Known)

The following fields exist in `prisma/schema.prisma` but were **never added via migration** — they were applied directly to the DB via `prisma db push`:

| Model | Fields | Risk |
|-------|--------|------|
| `Sale` | `clientAddress`, `clientEmail`, `clientName`, `clientPhone`, `discountAmount`, `discountPercent` | `prisma migrate deploy` will fail or drop these |
| `SaleItem` | `profit`, `purchasePriceAtSale` | Same as above |
| `Repair` | `partsCost`, `profit` | Same as above |
| `RepairPart` | `purchasePriceAtPart` | Same as above |

**Recommendation:** Create a migration that snapshots the current DB state before running `prisma migrate deploy` in production.

---

## Indexes Summary

All tables have indexes on foreign keys and frequently queried columns. Full list:

- `Client`: `phone`, `name`, `deletedAt`
- `Product`: `category`, `stock`, `name`, `supplier`, `deletedAt`
- `EcommerceProduct`: `visible`, `featured`, `sortOrder`, `publishedAt`
- `ProductMedia`: `ecommerceId`, `isPrimary`
- `Sale`: `clientId`, `createdAt`, `paymentMethod`
- `Repair`: `clientId`, `status`, `createdAt`
- `InventoryMovement`: `productId`, `createdAt`
- `Order`: `status`, `createdAt`, `clientPhone`, `externalReference`
- `OrderItem`: `orderId`

---

## Timestamp Type Inconsistency

| Tables | Column Type |
|--------|------------|
| Old ERP tables (`products`, `sales`, `orders`, ...) | `TIMESTAMP(3)` — no timezone |
| New ecommerce tables (`product_ecommerce`, `product_media`) | `TIMESTAMPTZ` — with timezone |

Both are functional in Prisma (`DateTime` maps to both), but direct SQL comparisons across table types may produce unexpected results.
