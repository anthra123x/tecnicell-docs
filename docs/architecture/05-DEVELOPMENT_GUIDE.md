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

### Storefront Project (`tecnicell-store`)

```env
# Database (read-only access recommended)
DATABASE_URL="postgresql://readonly:password@localhost:5432/tecnicell"
DIRECT_URL="postgresql://readonly:password@localhost:5432/tecnicell"

# API Base URL (ERP production or local tunnel)
NEXT_PUBLIC_API_URL="http://localhost:3000/api"

# App
NEXT_PUBLIC_APP_URL="http://localhost:3001"
```

---

## Local Setup

### 1. Clone both projects

```bash
git clone https://github.com/anthra123x/inventario-tecnicell.git
git clone https://github.com/anthra123x/tecnicell-store.git
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
cd tecnicell-store
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
- `NEXT_PUBLIC_API_URL` → ERP production URL

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
