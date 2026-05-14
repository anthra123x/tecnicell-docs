# Order Flow — Lifecycle, Stock & Edge Cases

## Overview

Orders originate from the storefront (public) and are managed in the ERP backoffice (staff-only). The order lifecycle involves status transitions that have **stock side effects**.

---

## Order Status Flow

```
                  ┌──────────┐
                  │ PENDING  │ ← Created by storefront
                  └────┬─────┘
                       │
              ┌────────┴────────┐
              │                 │
        ┌─────▼─────┐   ┌──────▼──────┐
        │ CONFIRMED  │   │  CANCELLED  │ ← Stock restored
        └─────┬─────┘   └─────────────┘
              │
        ┌─────▼──────┐
        │  PREPARING  │
        └─────┬──────┘
              │
        ┌─────▼────┐
        │ SHIPPED   │
        └─────┬────┘
              │
        ┌─────▼──────┐
        │ DELIVERED  │ ← Terminal state
        └────────────┘
```

### Transitions

| From | To | Stock Effect | Who Triggers |
|------|----|-------------|--------------|
| `PENDING` | `CONFIRMED` | **Decrement** stock for each item | ERP admin |
| `PENDING` | `CANCELLED` | None (stock was never reserved) | ERP admin |
| `CONFIRMED` | `PREPARING` | None | ERP admin |
| `PREPARING` | `SHIPPED` | None | ERP admin |
| `SHIPPED` | `DELIVERED` | None | ERP admin |
| `ANY` | `CANCELLED` | **Restore** stock for each item | ERP admin |

---

## Stock Side Effects — Deep Dive

### `PENDING → CONFIRMED` (Stock Decrement)

```typescript
// In orders.actions.ts :: updateOrderStatus
// For each order item:
await tx.product.update({
  where: { id: item.productId },
  data: { stock: { decrement: item.quantity } },
})
// Creates InventoryMovement with type EXIT, reason "Pedido online #[id]"
```

- This is executed inside a `prisma.$transaction()` with the status update
- If any product has insufficient stock, the transaction **still proceeds** (stock can go negative)

### `ANY → CANCELLED` (Stock Restore)

```typescript
// In orders.actions.ts :: updateOrderStatus
// For each order item:
await tx.product.update({
  where: { id: item.productId },
  data: { stock: { increment: item.quantity } },
})
// Creates InventoryMovement with type ENTRY, reason "Cancelacion pedido #[id]"
```

- Stock is restored regardless of current status
- If the order was already `CONFIRMED` (stock was decremented), cancel restores it correctly
- If the order was still `PENDING` (stock was never decremented), cancel still **increments** stock — this is a potential over-restore bug

> ⚠️ **Bug risk:** Cancelling a `PENDING` order incorrectly increments stock. The code does NOT check if the order was previously confirmed before restoring stock.

---

## Architectural Risk: Stock Reservation Window

```
Time →
│
├─ Storefront creates order (POST /api/orders)
│  └─ Stock CHECKED but NOT decremented
│
│  ... window where oversell is possible ...
│
├─ ERP admin confirms order (PENDING → CONFIRMED)
│  └─ Stock DECREMENTED
│
│  ... stock is now truly deducted ...
│
└─ ERP admin ships/delivers order
```

**Risk:** Between order creation and confirmation, another order or POS sale can consume the same stock. The check at creation time does NOT guarantee availability at confirmation time.

**Mitigation (future):** Reserve stock at `POST /api/orders` time or reject confirmation if stock is now insufficient.

---

## Complete Order Creation Flow (Storefront → ERP)

```
Storefront                     API                        Database
    │                           │                           │
    │  POST /api/orders         │                           │
    │  { clientName, items }    │                           │
    │──────────────────────────>│                           │
    │                           │                           │
    │                           │  Zod Validate             │
    │                           │  ──────────►              │
    │                           │  (400 if invalid)         │
    │                           │                           │
    │                           │  Check Stock (per item)   │
    │                           │  ────────────────────────►│
    │                           │  ◄────────────────────────│
    │                           │  (409 if insufficient)    │
    │                           │                           │
    │                           │  Check ExternalReference  │
    │                           │  ────────────────────────►│
    │                           │  ◄────────────────────────│
    │                           │  (409 if duplicate)       │
    │                           │                           │
    │                           │  $transaction:            │
    │                           │  ├─ INSERT INTO orders    │
    │                           │  └─ INSERT INTO order_items│
    │                           │  ────────────────────────►│
    │                           │                           │
    │  ◄────────────────────────│                           │
    │  201 { order }            │                           │
    │                           │                           │
```

---

## Order Status Management (ERP Admin)

Order status is managed via `updateOrderStatus(orderId, status)` server action in `orders.actions.ts`.

This function:
1. Validates the new status via Zod `UpdateOrderStatusSchema`
2. Opens a `prisma.$transaction()`
3. Updates the order status
4. If `CONFIRMED`: decrements stock, creates EXIT inventory movements
5. If `CANCELLED`: restores stock, creates ENTRY inventory movements
6. Returns `{ success: true }` or `{ error: "..." }`

**There is NO validation that prevents illegal transitions** (e.g., going from `PENDING` directly to `DELIVERED`, or trying to confirm an already-confirmed order). The code trusts the admin UI to enforce valid transitions.

---

## Edge Cases

| Scenario | Behavior | Correctness |
|----------|----------|-------------|
| Insufficient stock at confirmation | Stock goes negative | ❌ Should reject confirmation |
| Cancel PENDING order | Stock incremented (over-restore) | ❌ Should be no-op |
| Cancel already-cancelled order | Stock incremented again (double restore) | ❌ Should be no-op |
| Duplicate externalReference | 409 at creation | ✅ Correct |
| Product deleted after order created | FK constraint prevents product deletion | ✅ Safe |
| Order with 0 items | Zod validation rejects (min 1) | ✅ Correct |
| Negative total | Zod validation rejects (min 0) | ✅ Correct |
| Same product in multiple items | Allowed, each line decremented separately | ⚠️ Unusual but functional |
| Order from unknown phone number | Allowed (no client registry required) | ✅ By design |

---

## Inventory Movements for Orders

Each stock-affecting status change creates an `InventoryMovement` record:

| Transition | Movement Type | Reason |
|-----------|--------------|--------|
| `PENDING → CONFIRMED` | `EXIT` | `"Pedido online #[orderId]"` |
| `→ CANCELLED` | `ENTRY` | `"Cancelacion pedido #[orderId]"` |

These movements are visible in the ERP's inventory module and are used for audit trails and reports.

---

## Order Data Model (Summary)

```prisma
model Order {
  id                String      @id @default(cuid())
  clientName        String
  clientPhone       String
  clientCity        String?
  clientEmail       String?
  clientAddress     String?
  clientNotes       String?
  status            OrderStatus @default(PENDING)
  subtotal          Float       @default(0)
  shipping          Float       @default(0)
  total             Float       @default(0)
  externalReference String?     @unique
  internalNotes     String?
  createdAt         DateTime    @default(now())
  updatedAt         DateTime    @updatedAt
  items             OrderItem[]
  @@map("orders")
}
```

Fields the storefront provides at creation: `clientName`, `clientPhone`, `clientEmail`, `clientCity`, `clientAddress`, `clientNotes`, `subtotal`, `shipping`, `total`, `externalReference`, `items`.

Fields the ERP manages: `status`, `internalNotes`, `updatedAt`.

---

## Recommendations

1. **Add stock re-check at confirmation time** — Before allowing `PENDING → CONFIRMED`, verify all items still have sufficient stock
2. **Add idempotency key** — Use externalReference as an idempotency key to prevent duplicate submissions from the storefront
3. **Validate status transitions** — Add a whitelist of allowed transitions to prevent illegal state changes
4. **Fix cancel-on-PENDING bug** — Only restore stock if the order was previously confirmed
5. **Add webhook on status change** — Notify the storefront when order status changes (for customer email/SMS)
