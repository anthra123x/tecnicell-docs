# Storefront API Contract

Base URL (production): `https://tecnicell.vercel.app/api`

All endpoints are **public** (no authentication required). They are consumed by the Tecnicell Storefront to display products, create orders, and track order status.

---

## Endpoints Overview

| Method | Path | Host | Purpose |
|--------|------|------|---------|
| `GET` | `/api/ecommerce/products` | **ERP only** | List visible products with pagination, filtering, search |
| `GET` | `/api/ecommerce/products/[id]` | **ERP only** | Single product detail (by ID or slug) |
| `POST` | `/api/orders` | **Both** (ERP + Storefront) | Create a new order |
| `GET` | `/api/orders/[id]` | **Both** (ERP + Storefront) | Order tracking/lookup (by ID or externalReference) |

> **Note:** Both projects expose `POST /api/orders` and `GET /api/orders/[id]`. The storefront has local implementations that read/write the shared DB directly via Prisma. When `ECOMMERCE_API_URL` is configured, the storefront can delegate these calls to the ERP instead.

---

## `GET /api/ecommerce/products`

Returns paginated list of visible ecommerce products.

### Query Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `category` | string | — | Filter by `ProductCategory`. Pass `ALL` for no filter. |
| `search` | string | — | Case-insensitive text search on product name |
| `page` | number | 1 | Page number |
| `limit` | number | 50 | Items per page (max 100, capped server-side) |
| `featured` | boolean | — | Set to `"true"` to show only featured products |
| `slugs` | string | — | Comma-separated list of slugs to fetch specific products |

### Response (200)

```json
{
  "products": [
    {
      "id": "clx...",
      "slug": "cargador-rapido-20w",
      "name": "Cargador Rápido 20W",
      "description": "Cargador original 20W con carga rápida",
      "shortDescription": "Cargador original 20W USB-C",
      "longDescription": "<p>Larga descripción HTML</p>",
      "category": "ACCESSORY",
      "price": 45000,
      "compareAtPrice": 55000,
      "salePrice": 50000,
      "stock": 15,
      "showStock": true,
      "inStock": true,
      "featured": true,
      "badges": ["Nuevo", "Oferta"],
      "tags": ["cargador", "20w", "usb-c"],
      "image": {
        "url": "https://images.example.com/cargador.jpg",
        "alt": "Cargador Rápido 20W",
        "width": 800,
        "height": 800
      },
      "createdAt": "2026-05-13T00:00:00.000Z",
      "updatedAt": "2026-05-13T00:00:00.000Z"
    }
  ],
  "total": 42,
  "page": 1,
  "totalPages": 5
}
```

### Business Rules

- Only returns products where `visible = true` AND `product.stock > 0` AND `product.deletedAt IS NULL`
- `price` = `ecommercePrice` if set, otherwise falls back to `product.salePrice`
- `image` is the primary image (first with `isPrimary = true`), or `null` if no images exist
- `description` = `shortDescription` if set, otherwise `product.description`
- Products are ordered by `sortOrder` ascending, then `product.createdAt` descending
- When `slugs` parameter is used, all other filters (except category and search) still apply

### Error Responses

| Status | Body |
|--------|------|
| 500 | `{ "error": "Error al obtener productos" }` |

---

## `GET /api/ecommerce/products/[id]`

Single product detail. The `id` can be a **product ID** (cuid) or a **slug**.

### Response (200)

```json
{
  "product": {
    "id": "clx...",
    "slug": "cargador-rapido-20w",
    "name": "Cargador Rápido 20W",
    "description": "Cargador original 20W con carga rápida",
    "shortDescription": "Cargador original 20W USB-C",
    "longDescription": "<p>Larga descripción HTML</p>",
    "category": "ACCESSORY",
    "price": 45000,
    "compareAtPrice": 55000,
    "salePrice": 50000,
    "stock": 15,
    "showStock": true,
    "inStock": true,
    "featured": true,
    "badges": ["Nuevo", "Oferta"],
    "tags": ["cargador", "20w", "usb-c"],
    "metaTitle": "Cargador Rápido 20W | Tecnicell",
    "metaDescription": "Cargador original 20W con carga rápida USB-C, compatible con iPhone y Android",
    "images": [
      {
        "id": "cm...",
        "url": "https://images.example.com/cargador.jpg",
        "alt": "Cargador Rápido 20W",
        "width": 800,
        "height": 800,
        "sortOrder": 0,
        "isPrimary": true
      }
    ],
    "primaryImage": {
      "url": "https://images.example.com/cargador.jpg",
      "alt": "Cargador Rápido 20W",
      "width": 800,
      "height": 800
    },
    "createdAt": "2026-05-13T00:00:00.000Z",
    "updatedAt": "2026-05-13T00:00:00.000Z"
  }
}
```

### Error Responses

| Status | Body |
|--------|------|
| 404 | `{ "error": "Producto no encontrado" }` |
| 500 | `{ "error": "Error al obtener producto" }` |

---

## `POST /api/orders`

Create a new order from the storefront.

### Request Body

```json
{
  "clientName": "Juan Pérez",
  "clientPhone": "3001234567",
  "clientEmail": "juan@example.com",
  "clientCity": "Bogotá",
  "clientAddress": "Calle 123 #45-67",
  "clientNotes": "Entregar en portería",
  "subtotal": 90000,
  "shipping": 5000,
  "total": 95000,
  "externalReference": "ORD-001",
  "items": [
    {
      "productId": "clx...",
      "quantity": 2,
      "unitPrice": 45000
    }
  ]
}
```

### Field Validation (applied via Zod `CreateOrderSchema`)

| Field | Required | Type | Constraints |
|-------|----------|------|-------------|
| `clientName` | Yes | string | min 2 characters |
| `clientPhone` | Yes | string | min 8 characters |
| `clientEmail` | No | string | Valid email if provided |
| `clientCity` | No | string | — |
| `clientAddress` | No | string | — |
| `clientNotes` | No | string | — |
| `subtotal` | No | number | Default 0, min 0 |
| `shipping` | No | number | Default 0, min 0 |
| `total` | Yes | number | min 0 |
| `externalReference` | No | string | Must be unique across all orders |
| `items` | Yes | array | min 1 item |

**Each item:**

| Field | Required | Type | Constraints |
|-------|----------|------|-------------|
| `productId` | Yes | string | Must reference an existing product |
| `quantity` | Yes | number | Integer, min 1 |
| `unitPrice` | Yes | number | min 0 |

### Success Response (201)

```json
{
  "order": {
    "id": "clx...",
    "clientName": "Juan Pérez",
    "clientPhone": "3001234567",
    "clientCity": "Bogotá",
    "clientEmail": "juan@example.com",
    "clientAddress": "Calle 123 #45-67",
    "clientNotes": null,
    "status": "PENDING",
    "subtotal": 90000,
    "shipping": 5000,
    "total": 95000,
    "externalReference": "ORD-001",
    "createdAt": "2026-05-13T12:00:00.000Z",
    "updatedAt": "2026-05-13T12:00:00.000Z",
    "items": [
      {
        "id": "cm...",
        "orderId": "clx...",
        "productId": "clx...",
        "quantity": 2,
        "unitPrice": 45000,
        "total": 90000
      }
    ]
  }
}
```

### Error Responses

| Status | Body | When |
|--------|------|------|
| 400 | `{ "error": "El nombre debe tener al menos 2 caracteres" }` | Zod validation failed — message varies by field |
| 409 | `{ "error": "Stock insuficiente para Cargador Rápido 20W" }` | A product doesn't have enough stock |
| 409 | `{ "error": "Ya existe un pedido con la referencia: ORD-001" }` | Duplicate `externalReference` |
| 500 | `{ "error": "<error message>" }` | Unexpected server error |

### Order Creation Flow

```
Storefront → POST /api/orders
  │
  ├─ 1. Zod validation (400 if fails)
  │
  ├─ 2. Stock check per item (409 if insufficient)
  │     └─ prisma.product.findUnique({ id: item.productId })
  │
  ├─ 3. Duplicate externalReference check (409 if exists)
  │     └─ prisma.order.findUnique({ externalReference })
  │
  └─ 4. Transaction: Create order + order items
        └─ prisma.$transaction()
              ├─ tx.order.create({ data, include: { items } })
              └─ Returns new order
```

> ⚠️ **IMPORTANT:** Stock is NOT decremented at this stage. It is only decremented when an admin confirms the order (`PENDING → CONFIRMED` in the ERP). See `03-ORDER_FLOW.md`.

---

## `GET /api/orders/[id]`

Order tracking / status lookup. The `id` can be an **order ID** (cuid) or an **externalReference**.

### Response (200)

```json
{
  "order": {
    "id": "clx...",
    "clientName": "Juan Pérez",
    "clientPhone": "3001234567",
    "clientEmail": "juan@example.com",
    "clientCity": "Bogotá",
    "clientAddress": "Calle 123 #45-67",
    "clientNotes": "Entregar en portería",
    "status": "CONFIRMED",
    "subtotal": 90000,
    "shipping": 5000,
    "total": 95000,
    "externalReference": "ORD-001",
    "internalNotes": null,
    "createdAt": "2026-05-13T12:00:00.000Z",
    "updatedAt": "2026-05-13T12:30:00.000Z",
    "items": [
      {
        "id": "cm...",
        "orderId": "clx...",
        "productId": "clx...",
        "quantity": 2,
        "unitPrice": 45000,
        "total": 90000,
        "product": {
          "id": "clx...",
          "name": "Cargador Rápido 20W",
          "description": "Cargador original 20W con carga rápida",
          "category": "ACCESSORY",
          "salePrice": 50000
        }
      }
    ]
  }
}
```

### Error Responses

| Status | Body |
|--------|------|
| 404 | `{ "error": "Pedido no encontrado" }` |
| 500 | `{ "error": "Error al obtener el pedido" }` |

---

## Legacy Endpoint (DO NOT USE for storefront)

### `GET /api/products?ecommerce=true`

This endpoint exists for backward compatibility but is **deprecated**. It returns a different payload structure. The storefront MUST use `/api/ecommerce/products` instead.

**Differences from `/api/ecommerce/products`:**
- Different response field names
- No `shortDescription`, `longDescription`, `tags`, `metaTitle`, `metaDescription`
- Primary image returns as `image` only (no `images` array)
- Missing `createdAt`/`updatedAt` timestamps
- No `slugs` filter parameter
- No `featured` filter

---

## API Security Model

| Endpoint | Auth Required | Notes |
|----------|--------------|-------|
| `GET /api/ecommerce/products` | No | Public catalog |
| `GET /api/ecommerce/products/[id]` | No | Public product detail |
| `POST /api/orders` | No | Public order creation |
| `GET /api/orders/[id]` | No | Public order tracking |

The ERP's Supabase auth middleware only protects page routes (`/dashboard`, `/inventory`, `/sales`, etc.). All `/api/*` routes are explicitly excluded from auth checks.

**Recommendation:** If the storefront needs admin-only API access in the future, add API key authentication via headers (e.g., `x-api-key`) or JWT tokens.

---

## Storefront Local Endpoints

The storefront project (`tienda_online_tecnicell`) has its own implementations of some endpoints that read/write the shared DB directly. These serve as fallback when the ERP API is unavailable or in development mode.

### `POST /api/orders` (Storefront)

Located at `src/app/api/orders/route.ts` in the storefront project.

**Functionally identical to the ERP version** — validates input, checks stock, creates order with items via Prisma directly on the shared DB.

**Differences from ERP version:**
- Uses the storefront's own Prisma Client (pointing to the same DB)
- No Zod validation (manual validation)
- Does NOT check duplicate `externalReference`
- Stock check uses the same logic: verifies `product.stock >= item.quantity`

### `GET /api/orders/[id]` (Storefront)

Located at `src/app/api/orders/[id]/route.ts` in the storefront project.

**Same contract as ERP version** — looks up by `id` or `externalReference`, returns order with items and product details.

### When to Use Which

| Mode | Config | Product Catalog | Orders |
|------|--------|----------------|--------|
| **API mode** | `ECOMMERCE_API_URL` set | Reads from ERP API | Delegates to ERP API (future) |
| **DB mode** (default) | `ECOMMERCE_API_URL` empty | Reads via Prisma join on `product_ecommerce` + `products` + `product_media` | Creates/reads via local Prisma |
| **Fallback** | No `product_ecommerce` table | Falls back to `products` table only | N/A (always available) |
