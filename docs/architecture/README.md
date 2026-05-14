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
