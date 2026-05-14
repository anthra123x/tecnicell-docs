# Future Ecommerce Roadmap — Planned Improvements & Features

## Priority Matrix

| Priority | Feature | Effort | Impact | Dependencies |
|----------|---------|--------|--------|-------------|
| P0 | Fix stock reservation window | Medium | Critical | Orders module |
| P0 | Fix cancel-on-PENDING over-restore | Small | High | Orders module |
| P1 | Payment Gateway Integration | Large | High | Storefront + new `payments` table |
| P1 | Cart persistence / abandoned cart | Medium | Medium | Storefront only |
| P2 | Image storage migration | Medium | Medium | Ecommerce module |
| P2 | Webhooks for status changes | Small | Medium | Orders module |
| P2 | Order confirmation email | Small | Medium | Notifications module |
| P3 | API rate limiting | Small | Low | API middleware |
| P3 | SEO improvements | Medium | Medium | Storefront |
| P3 | Customer accounts / auth | Large | High | Storefront + new `customers` table |

---

## P0 — Critical Fixes

### Fix Stock Reservation Window

**Problem:** `POST /api/orders` checks stock availability but does NOT decrement it. Between order creation and admin confirmation (`PENDING → CONFIRMED`), another order or POS sale can consume the same stock.

**Proposed solution options:**

**Option A — Reserve on creation**
```typescript
// In orders.actions.ts :: createOrder
// Within the $transaction, after creating the order:
await tx.product.update({
  where: { id: item.productId },
  data: { stock: { decrement: item.quantity } },
})
```
- ✅ Simple change
- ❌ Requires reverting reserve if PENDING order is later CANCELLED (currently handled)

**Option B — Check on confirmation**
```typescript
// In orders.actions.ts :: updateOrderStatus (PENDING → CONFIRMED)
// Before decrementing, re-check stock:
for (const item of order.items) {
  const product = await tx.product.findUnique({ where: { id: item.productId } })
  if (!product || product.stock < item.quantity) {
    return { error: `Stock insuficiente para ${product.name}` }
  }
}
```
- ✅ Minimal change
- ✅ Prevents negative stock
- ❌ Customer already received "order confirmed" email

**Option C — Hybrid (reserve on creation + release on cancel)**
- Best UX but requires coordination between both teams

**Recommendation:** Implement Option B now (stock re-check at confirmation) as a safety net. Evaluate Option A for v2.

### Fix Cancel-on-PENDING Over-Restore

**Problem:** When cancelling a `PENDING` order, the code unconditionally increments stock. Since `PENDING` orders never decremented stock, this creates phantom stock.

**Fix:**
```typescript
// In orders.actions.ts :: updateOrderStatus
if (newStatus === 'CANCELLED') {
  const previousStatus = order.status
  if (previousStatus === 'CONFIRMED' || previousStatus === 'PREPARING' || previousStatus === 'SHIPPED') {
    // Only restore stock if it was previously decremented
    for (const item of order.items) {
      await tx.product.update({
        where: { id: item.productId },
        data: { stock: { increment: item.quantity } },
      })
    }
  }
  // If previousStatus was PENDING, do nothing (stock was never reserved)
}
```

---

## P1 — High Priority

### Payment Gateway Integration

**Goal:** Accept payments directly on the storefront (Mercado Pago, Stripe, etc.)

**Required changes:**

1. **New DB table** `payments`:
   ```prisma
   model Payment {
     id        String   @id @default(cuid())
     orderId   String
     method    String   // "MERCADO_PAGO", "STRIPE", "CASH_ON_DELIVERY"
     status    String   // "PENDING", "PAID", "FAILED", "REFUNDED"
     amount    Float
     gatewayId String?  // Payment gateway's transaction ID
     createdAt DateTime @default(now())
     updatedAt DateTime @updatedAt
     order     Order    @relation(fields: [orderId], references: [id])
     @@map("payments")
   }
   ```

2. **Storefront:** Payment form + gateway SDK integration
3. **ERP:** Order status auto-advance on payment confirmation
4. **API:** Add payment status to `GET /api/orders/[id]` response

**Considerations:**
- Cash on delivery (COD) is the current implicit flow
- Payment confirmation should trigger `PENDING → CONFIRMED` automatically
- Failed payments should not create inventory movements
- Webhook endpoint needed for gateway callbacks

### Cart Persistence

**Goal:** Save carts so users don't lose them on browser close.

**Options:**
- localStorage (current implicit approach — no code change needed)
- Server-side cart with `externalReference` lookup (recommended)
- Anonymous cart via device fingerprinting

---

## P2 — Medium Priority

### Image Storage Migration

**Current state:** Images use `storage_provider = "placeholder"` — URLs are stored directly in the `url` field (external links or direct upload URLs).

**Target state:** Migrate to Supabase Storage or Cloudinary.

**Migration plan:**
1. Add upload endpoint in the ERP: `POST /api/ecommerce/products/[id]/images/upload`
2. Upload to Supabase Storage bucket `product-images`
3. Store the provider (`"supabase"`) and storage ID in `product_media`
4. Serve images via CDN URL
5. Backfill existing placeholder images (optional)

### Webhooks for Status Changes

**Goal:** Notify the storefront (or external services) when order status changes.

**New API endpoint:**
```
POST /api/ecommerce/webhooks/order-status-changed
Body: { orderId, previousStatus, newStatus, timestamp }
```

**Use cases:**
- Send SMS/email to customer
- Update order tracking page in real-time
- Trigger fulfillment workflow

### Order Confirmation Email (Re-implement)

**Note:** Email notifications via Resend were previously implemented and removed by decision. If re-introduced, consider:
- Using a lightweight email service (Resend, SendGrid)
- Triggering on `PENDING → CONFIRMED` transition
- Template stored in `SystemSettings` or a new `email_templates` table
- Customer email is optional — only send if `clientEmail` is provided

---

## P3 — Nice to Have

### API Rate Limiting

**Why:** All API endpoints are currently public with no throttling.

**Implementation:**
- Vercel Edge Middleware or Upstash Rate Limiting
- Apply to `POST /api/orders` first (most sensitive)
- Per-IP rate limits: 10 requests/minute for POST, 100/minute for GET

### SEO Improvements

- Structured data (JSON-LD) for products
- Better meta tags (product-specific from API)
- Sitemap generation (`/sitemap.xml`)
- robots.txt optimization

### Customer Accounts / Auth

Full customer authentication for the storefront:
- Order history per customer
- Saved addresses
- Wishlists
- Faster checkout

**Requires:** New `customers` table in shared DB, Supabase Auth (separate from ERP staff auth), or a simpler auth approach.

---

## Technical Debt & Maintenance

| Item | Description | Effort |
|------|-------------|--------|
| Fix schema drift | Create migration to capture fields added via `db push` | Small |
| Standardize timestamps | Migrate old tables from `TIMESTAMP(3)` to `TIMESTAMPTZ` | Medium |
| Remove trigger | Replace PostgreSQL `updated_at` trigger with Prisma-only approach | Small |
| Remove legacy endpoint | Deprecate `GET /api/products?ecommerce=true` | Small |
| Type `any` cleanup | Remove `any` casts in API route handlers | Medium |
| Currency as Decimal | Migrate `Float` prices to `Decimal` for precision | Medium |

---

## Migration Guidelines

When implementing any of the above:

1. **Document the change first** in this roadmap
2. **Create a new migration file** — never edit existing migrations
3. **Update both projects** — if you add a field to a shared table, update both Prisma schemas
4. **Backward compatibility** — new API fields should be optional (nullable) for 1 release cycle
5. **Test the integration** — run both projects locally and verify the full flow
6. **Update agent prompts** — if the change affects how agents work, update the shared documentation

---

## Feature Request Template

To propose a new feature not listed here, create a document in `docs/architecture/proposals/` with:

```markdown
# Proposal: [Feature Name]

## Problem
What problem does this solve?

## Proposed Solution
High-level approach

## Changes Required
- DB: [new tables/fields]
- API: [new/modified endpoints]
- ERP: [new pages/actions]
- Storefront: [new pages/components]

## Risks
What could go wrong?

## Migration
How to roll out without breaking existing functionality
```
