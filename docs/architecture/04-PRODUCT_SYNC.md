# Product Sync — Catalog Lifecycle & Stock Synchronization

## Overview

Products have two representations:
- **`Product`** (table: `products`) — ERP base data: stock, prices, categories, supplier
- **`EcommerceProduct`** (table: `product_ecommerce`) — Storefront extensions: display prices, images, badges, SEO

A product becomes visible on the storefront only when BOTH conditions are met:
1. It has an `EcommerceProduct` record with `visible = true`
2. The ERP's `Product.stock > 0` (filtered in API)

---

## Product Lifecycle Diagram

```
ERP Inventory                              Ecommerce Admin                Storefront
    │                                           │                           │
    │  Create Product                           │                           │
    │  (name, stock, category,                  │                           │
    │   purchasePrice, salePrice)               │                           │
    │──────────────────────────► DB             │                           │
    │                                           │                           │
    │                                           │  Add to Catalog            │
    │                                           │  (creates EcommerceProduct │
    │                                           │   with auto-slug,         │
    │                                           │   visible based on        │
    │                                           │   category)               │
    │                                           │──────────► DB             │
    │                                           │                           │
    │                                           │  Edit Ecommerce Settings  │
    │                                           │  ├─ Prices, badges, tags  │
    │                                           │  ├─ Images, descriptions  │
    │                                           │  ├─ Visibility toggle     │
    │                                           │  ├─ Featured, sort order  │
    │                                           │  └─ SEO metadata          │
    │                                           │──────────► DB             │
    │                                           │                           │
    │  Update Stock                             │                 GET /api/ecommerce/products
    │  (via POS sale, inventory                  │                 ◄────────────│──────────
    │   movement, order confirm)                │                           │
    │──────────────────────────► DB             │  Read stock from Product  │
    │                                           │  ◄─────────────────────────│
    │                                           │                           │
    │                                           │                 Return product
    │                                           │                 ──────────│─────────►
```

---

## Adding a Product to the Catalog

### Step 1: ERP creates a Product

All products start in the ERP Inventory module. They have no ecommerce data yet.

```typescript
// Server action: inventory.actions.ts :: createProduct
// Creates Product with: name, description, category, stock, prices, supplier
```

### Step 2: Admin adds to ecommerce catalog

From the ecommerce admin panel ("Catálogo Online" → "Agregar producto"), the admin selects an unconverted product and creates an `EcommerceProduct` record.

```typescript
// Server action: ecommerce.actions.ts :: createEcommerceProduct(productId)
// - Auto-generates slug: `${product.name.toLowerCase().replace(/\s+/g, '-')}-${id.slice(-6)}`
// - Sets visible based on category:
//     REPAIR_PART → visible = false (hidden by default)
//     All others  → visible = true
```

**Slug generation rules:**
- Lowercase product name with spaces replaced by `-`
- Special characters removed
- Suffixed with last 6 characters of the ecommerce record ID for uniqueness
- Example: `"Cargador Rápido 20W"` → `cargador-rapido-20w-a1b2c3`

### Step 3: Admin configures ecommerce settings

From the product edit page at `/ecommerce/products/[id]`, the admin can modify:

| Setting | Field | Default | Storefront Effect |
|---------|-------|---------|-------------------|
| Display price | `ecommercePrice` | `null` → falls back to `salePrice` | `price` in API response |
| Compare price | `compareAtPrice` | `null` | Strikethrough price (for "was" display) |
| Visible | `visible` | `true` (false for REPAIR_PART) | Hidden from catalog if false |
| Featured | `featured` | `false` | Highlight on storefront |
| Show stock | `showStock` | `true` | Controls stock visibility on PDP |
| Badges | `badges` | `[]` | Array of strings: `["Nuevo", "Oferta"]` |
| Tags | `tags` | `[]` | Search/filter tags |
| Sort order | `sortOrder` | `0` | Display order in listings |
| Slug | `slug` | Auto-generated | URL for product detail page |
| Images | `media` | `[]` | Product gallery |

---

## Stock Synchronization

### Source of Truth

`Product.stock` is the **single source of truth** for all stock operations. The ecommerce system has NO stock field.

### How Stock Changes (ERP-side)

```
Operation                    │ Module              │ Effect on Product.stock
─────────────────────────────┼─────────────────────┼────────────────────────
Manual inventory entry       │ inventory.actions   │ +quantity
Manual inventory exit        │ inventory.actions   │ -quantity
POS sale                     │ sales.actions       │ -quantity (per item)
Order confirmed (PENDING→CNF)│ orders.actions      │ -quantity (per item)
Order cancelled (any→CANC)   │ orders.actions      │ +quantity (per item)
Repair created (parts used)  │ repairs.actions     │ -quantity (per part)
Repair edited (part removed) │ repairs.actions     │ +quantity (per part)
Repair edited (part added)   │ repairs.actions     │ -quantity (per part)
Repair deleted               │ repairs.actions     │ +quantity (per part)
```

### How the Storefront Reads Stock

The storefront NEVER writes to `Product.stock`. It reads stock via:

1. **API response**: `GET /api/ecommerce/products` returns `stock` and `inStock` fields
2. **Direct DB read** (read-only): Only if the storefront has its own Prisma schema pointing to the shared tables

### What the Storefront Shows

| Setting | `showStock=true` | `showStock=false` |
|---------|------------------|-------------------|
| In stock | Show "15 en stock" | Hide stock count |
| Out of stock | Filtered from catalog | Filtered from catalog |

> Products with `stock = 0` are **always excluded** from the catalog API response, regardless of the `showStock` setting. The `showStock` flag only controls whether the count is visible on the product detail page.

---

## Price Calculation

The API computes `price` as:

```
price = ecommercePrice ?? salePrice
```

| Scenario | `ecommercePrice` | `salePrice` | `price` (API) | Storefront Display |
|----------|-----------------|-------------|---------------|-------------------|
| No ecommerce price set | `null` | 50000 | 50000 | "$50.000" |
| Ecommerce price set | 45000 | 50000 | 45000 | "$45.000" |
| With compare-at price | 45000 | 50000 | 45000 | "$55.000 $45.000" (strikethrough) |
| Both null | `null` | 50000 | 50000 | "$50.000" |

---

## Image Management

Images are stored in `product_media` table with these rules:

- **Primary image**: The image with `isPrimary = true` is returned as `image` in listings and `primaryImage` in detail view
- **Sort order**: Images are ordered by `sortOrder` ascending (0, 1, 2...)
- **Multiple images**: All images are returned in the `images` array on the detail endpoint
- **First image**: If no primary is set, the first image (by sortOrder) is used as primary
- **Storage provider**: Currently defaults to `"placeholder"` — URLs are stored directly. Future: migrate to Supabase Storage or Cloudinary

```json
// Product detail response - images section
"images": [
  {
    "id": "cm...",
    "url": "https://images.example.com/photo-1.jpg",
    "alt": "Product photo 1",
    "width": 800,
    "height": 800,
    "sortOrder": 0,
    "isPrimary": true
  },
  {
    "id": "cm...",
    "url": "https://images.example.com/photo-2.jpg",
    "alt": "Product photo 2",
    "width": 800,
    "height": 800,
    "sortOrder": 1,
    "isPrimary": false
  }
]
```

---

## Visibility & Published State

| State | `visible` | `publishedAt` | API Includes? | Notes |
|-------|-----------|---------------|---------------|-------|
| Draft | `false` | `null` | ❌ No | After creation, before first publish |
| Published | `true` | timestamp | ✅ Yes | Set automatically when made visible |
| Hidden | `false` | `null` | ❌ No | `publishedAt` cleared on hide |

**Toggle behavior:**
- Making visible: `visible = true`, `publishedAt = new Date()`
- Making hidden: `visible = false`, `publishedAt = null`

---

## Badges & Tags

### Badges
- Array of plain strings: `["Nuevo", "Oferta", "Envío Gratis"]`
- No predefined list — admins type and add via Enter key
- Displayed as colored pills on storefront product cards

### Tags
- Array of plain strings: `["cargador", "20w", "usb-c"]`
- Used for internal search/filtering
- Not displayed to customers

---

## Storefront Product URL Structure

Each product has a unique `slug`. The storefront should use:

```
/products/:slug
```

The detail API accepts either slug or product ID:

```
GET /api/ecommerce/products/cargador-rapido-20w-a1b2c3
GET /api/ecommerce/products/clx123abc...
```

---

## Category Default for Storefront Visibility

When a product is first added to the ecommerce catalog:

```typescript
const isRepairPart = product.category === 'REPAIR_PART'
const data = {
  visible: !isRepairPart,  // REPAIR_PART hidden by default
  publishedAt: isRepairPart ? null : new Date(),
}
```

**Rationale:** Repair parts (screens, batteries, flex cables) are typically B2B/workshop items. They shouldn't appear on the public storefront unless explicitly enabled.

---

## Deletion Safety

- Deleting `EcommerceProduct` → CASCADE deletes `ProductMedia` (images are lost)
- Deleting `Product` → CASCADE deletes `EcommerceProduct` (ecommerce data is lost)
- Products are soft-deleted (`deletedAt`), and the API always filters them out
- Ecommerce products with soft-deleted base products are automatically excluded from API responses

---

## Contracts That Must NOT Be Broken

1. **`Product.stock` is immutable by the storefront** — No direct DB writes to stock column
2. **`price` in API always = `ecommercePrice ?? salePrice`** — Storefront must not implement its own price fallback logic
3. **`slug` / `productId` dual lookup** — Detail endpoint must support both
4. **API fields are the contract** — Any field added to the API response must be documented here
5. **Filtering always applies `deletedAt IS NULL` AND `stock > 0`** — Storefront cannot override this
