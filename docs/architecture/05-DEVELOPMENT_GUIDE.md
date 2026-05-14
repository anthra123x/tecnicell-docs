# Development Guide — Local Setup, Testing & Workflow

## Prerequisites

| Tool | Version | Purpose |
|------|---------|---------|
| Node.js | >= 20 | Runtime |
| npm | >= 10 | Package manager |
| Git | Latest | Version control |
| PostgreSQL | 15+ | Local database (or use Supabase remote) |

---

## Environment Variables

### ERP Project (`inventario-tecnicell`)

Create `.env` at the project root:

```env
# Database
DATABASE_URL="postgresql://postgres:password@localhost:5432/tecnicell?pgbouncer=true"
DIRECT_URL="postgresql://postgres:password@localhost:5432/tecnicell"

# Supabase Auth
NEXT_PUBLIC_SUPABASE_URL="https://your-project.supabase.co"
NEXT_PUBLIC_SUPABASE_ANON_KEY="your-anon-key"

# App
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

### Storefront Project (`tienda_online_tecnicell`)

```env
# Database (shared Supabase)
DATABASE_URL="postgresql://postgres:password@db.supabase.co:6543/postgres?pgbouncer=true"
DIRECT_URL="postgresql://postgres:password@db.supabase.co:5432/postgres"

# ERP API URL — set to use ERP endpoints instead of local DB mode
# When empty, the storefront reads from the shared DB directly via Prisma
# WARNING: This must point to the ERP's deployment URL (e.g. https://inventario-tecnicell.vercel.app),
# NOT to the storefront itself. Pointing it to the storefront URL will cause 404s on
# /api/ecommerce/products/* because those routes only exist in the ERP project.
ECOMMERCE_API_URL=""

# App
NEXT_PUBLIC_URL="http://localhost:3001"
NEXT_PUBLIC_WHATSAPP_PHONE="573001234567"
WHATSAPP_PHONE="573001234567"
```

---

## Local Setup

### 1. Clone both projects

```bash
git clone https://github.com/anthra123x/inventario-tecnicell.git
git clone https://github.com/anthra123x/tienda_online_tecnicell.git
```

### 2. Clone documentation (into each project)

```bash
# In each project root:
git submodule add https://github.com/anthra123x/tecnicell-docs.git docs
```

### 3. Install dependencies

```bash
# Each project:
npm install
```

### 4. Set up the database

```bash
# Option A: Use remote Supabase DB (recommended)
# Copy connection strings from Supabase dashboard to .env

# Option B: Local PostgreSQL
createdb tecnicell

# In the ERP project:
npx prisma db push --accept-data-loss  # For development
# OR
npx prisma migrate deploy               # For production-like setup
```

### 5. Run the projects

```bash
# Terminal 1 — ERP (port 3000)
cd inventario-tecnicell
npm run dev

# Terminal 2 — Storefront (port 3001)
cd tienda_online_tecnicell
npm run dev
```

---

## Testing Integration Locally

### With both projects running

1. **ERP** runs on `http://localhost:3000`
2. **Storefront** runs on `http://localhost:3001`
3. Storefront API calls go to `http://localhost:3000/api`

### Using a public tunnel (for webhook testing)

```bash
# Install ngrok or cloudflared
npx ngrok http 3000

# Update NEXT_PUBLIC_API_URL in storefront to the ngrok URL
```

### Testing API endpoints directly

```bash
# List products
curl http://localhost:3000/api/ecommerce/products?category=ACCESSORY&page=1&limit=5

# Create an order
curl -X POST http://localhost:3000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "clientName": "Test User",
    "clientPhone": "3000000000",
    "items": [{"productId": "clx...", "quantity": 1, "unitPrice": 50000}],
    "total": 50000
  }'

# Track an order
curl http://localhost:3000/api/orders/clx...
```

---

## Database Schema Management

### ERP Project (Prisma)

```bash
# After modifying prisma/schema.prisma:

# Development (fast, may create drift):
npx prisma db push

# Production (safe, creates migration):
npx prisma migrate dev --name description_of_change
npx prisma migrate deploy  # In CI/CD
```

### Storefront Project (if using Prisma for direct DB reads)

```bash
# Define models in prisma/schema.prisma with @@map() pointing to shared tables

# Example:
model EcommerceProduct {
  id             String   @id
  productId      String   @map("product_id")
  ecommercePrice Float?   @map("ecommerce_price")
  // ...
  @@map("product_ecommerce")
}

# Run:
npx prisma generate  # Only generate client, don't push/migrate
```

---

## Working with the Shared Documentation

### Updating docs (from either project)

```bash
cd docs  # This is the tecnicell-docs submodule

# Edit files...
git add .
git commit -m "update: describe new API endpoint"
git push origin main

# Go back to the parent project to record the new submodule commit
cd ..
git add docs
git commit -m "docs: sync submodule to latest"
git push
```

### Receiving doc updates

```bash
# From the parent project:
git submodule update --remote docs
git add docs
git commit -m "docs: sync submodule"
```

### First-time clone with submodules

```bash
git clone --recurse-submodules <repo-url>
# OR if already cloned:
git submodule init
git submodule update
```

---

## Common Issues & Troubleshooting

### "Prisma Client could not find schema"

```
npx prisma generate
```

### "Migration not applied"

```bash
# Check migration state
npx prisma migrate status

# Apply pending migrations
npx prisma migrate deploy
```

### "Schema drift detected" in Prisma Studio

This is expected if using `prisma db push` in development. To resolve:

```bash
# Create a baseline migration
npx prisma migrate diff --from-empty --to-schema-datamodel prisma/schema.prisma --script > migration.sql
```

### Orders with lowercase status causing errors

The DB has legacy orders with `status = 'preparing'` instead of `'PREPARING'`. If the storefront uses Prisma enums for direct DB reads, these rows will fail. Mitigation:

```bash
# Run this SQL in the Supabase SQL Editor:
UPDATE orders SET status = UPPER(status) WHERE status != UPPER(status);
```

### Product list shows "0 products" on Vercel after deploying fixes

**Root cause:** `unstable_cache` cached empty results from the old buggy code. Even with the fix deployed, the stale cached response (0 products from the old `ecommerce_products` query) is served instead of re-executing.

**Solutions:**
1. **Automatic** (already applied) — Cache keys now include a version prefix (`CACHE_VERSION = "v2"` in `src/modules/products/service.ts`). Bump this constant to bust all caches on next deploy.
2. **Manual** — Call the revalidation endpoint (requires `REVALIDATE_SECRET` env var):
   ```
   POST /api/revalidate?secret=<SECRET>&tag=products
   ```
3. **Wait** — The cache expires after the `revalidate` TTL (60s for product lists, 120s for details).

### Product page returns 404 in Vercel but works locally

**Root cause:** `ECOMMERCE_API_URL` is set on Vercel but points to the storefront's own URL (or another URL that doesn't have `/api/ecommerce/products/[slug]`). The storefront tries the API first, gets 404, and in `getProductBySlug` returns `null`, which triggers `notFound()` on the product detail page.

**Fix:** Either:
1. Set `ECOMMERCE_API_URL` to the actual ERP URL (where `GET /api/ecommerce/products/[slug]` exists)
2. Or delete/empty `ECOMMERCE_API_URL` in Vercel env to force DB-only mode

> All product functions now fall through to DB when the API fails, but `getProductBySlug` was the last one fixed — make sure the deployed commit includes `fae8ed0` or later.

### Submodule detached HEAD

If `cd docs && git status` shows "detached HEAD", the submodule is pinned to a specific commit. To work on docs:

```bash
cd docs
git checkout main
# Make changes...
git push origin main
cd ..
git add docs
git commit -m "docs: update submodule"
```

---

## Deployment

### ERP (Vercel)

```bash
# Production
vercel --prod

# Preview
vercel
```

Environment variables must be configured in Vercel dashboard:
- `DATABASE_URL`, `DIRECT_URL`
- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`

### Storefront (Vercel)

```bash
# Production
vercel --prod
```

Environment variables:
- `DATABASE_URL`, `DIRECT_URL` — Shared Supabase DB
- `ECOMMERCE_API_URL` → ERP production URL (leave empty to use DB-only mode)
- `NEXT_PUBLIC_URL` → Storefront URL

### Dual-mode fallback behavior

The product service (`src/modules/products/service.ts`) runs in two modes controlled by `ECOMMERCE_API_URL`:

| Mode | `ECOMMERCE_API_URL` | Data source |
|------|---------------------|-------------|
| **API mode** | Non-empty URL | Reads from ERP via `GET /api/ecommerce/products` |
| **DB mode** (default) | Empty | Reads via Prisma from shared DB (`product_ecommerce` → `products` → `product_media`) |

All 5 product queries wrap the API call in try/catch. If the ERP is unreachable, returns a non-2xx status, or throws any error, the function **silently falls through to DB mode**. This guarantees the storefront works even when the ERP is down or misconfigured.

```typescript
// Pattern used by all 5 product functions
if (USE_API) {
  try {
    const data = await getProductsApi(/* ... */);
    return data.products.map(fromApi);
  } catch {
    // API unavailable — fall through to DB mode
  }
}
```

### DB mode inner fallback

Within DB mode, there's a second level of fallback. When the Prisma query against `product_ecommerce` (mapped as `ecommerce_products`) succeeds but returns **0 results** (e.g., the table exists but has no records for a given category), the function does NOT stop — it continues to the legacy `products` table:

```typescript
return cached(cacheKey, 60, async () => {
  try {
    const [rows, total] = await db.ecommerce_products.findMany(/* ... */);
    if (total > 0) return { products: rows.map(fromDbEcommerce), total };
  } catch {
    // ecommerce_products unavailable — fall through to legacy
  }
  // Legacy fallback: products table (always has data)
  const [rows, total] = await db.products.findMany(/* ... */);
  return { products: rows.map(fromDb), total };
});
```

This handles the case where `product_ecommerce` exists in the database schema but hasn't been populated for all categories yet.

**Important:** On Vercel, if `ECOMMERCE_API_URL` is set but points to the storefront's own URL, every API call returns 404 because `/api/ecommerce/products` only exists in the ERP project. The fallback handles this gracefully, but the correct fix is to either:
- Point `ECOMMERCE_API_URL` to the actual ERP deployment URL, or
- Leave it empty to use DB-only mode (requires Prisma schema with `product_ecommerce` view)

---

## Code Quality

### TypeScript

Both projects use strict mode. Run type checking:

```bash
npx tsc --noEmit
# OR
npm run typecheck  # If configured
```

### Linting

```bash
npm run lint
```

### Build verification

Always verify the build before pushing:

```bash
npm run build
```

---

## Branch Strategy for Both Projects

```
main          ← Production-ready code
  └─ develop  ← Integration branch
       ├─ feat/xxx  ← Feature branches
       └─ fix/xxx   ← Bug fix branches
```

**Rules:**
- Never commit directly to `main` (except for documentation)
- Both projects' `main` branches should be compatible at all times
- When changing shared contracts (API, DB schema), coordinate across both projects
