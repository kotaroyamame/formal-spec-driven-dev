# E-commerce Order System: File Structure & Navigation Guide

## Quick Navigation

### Main Documentation
- **[README.md](./README.md)** - Start here
  - Overview of the example (EN/JP bilingual)
  - System architecture and module responsibilities
  - How to use this example to generate implementations
  - AI prompt templates for each module

### VDM-SL Formal Specifications
The `.vdm/` directory contains the formal specifications:

- **[.vdm/inventory.vdmsl](.vdm/inventory.vdmsl)** (211 lines)
  - Types: ProductId, StockInfo, Warehouse, ReservationId, StockCheckResult, StockException
  - State: inventory mapping, reservations, nextReservationId
  - Operations: CheckStock, ReserveStock, ReleaseStock, RestockProduct
  - Each operation has:
    - `ext` clauses (external state access: read/write)
    - `pre` conditions (what must be true before)
    - `post` conditions (what must be true after)
  - Invariants: reserved ≤ total, stock ≥ 0

- **[.vdm/order.vdmsl](.vdm/order.vdmsl)** (252 lines)
  - Types: OrderId, OrderStatus (<PENDING|CONFIRMED|COMPLETED|CANCELLED>), LineItem, Order
  - State: orders mapping, nextOrderId, pendingConfirmations
  - Operations: CreateOrder, ConfirmOrder, CancelOrder, GetOrderStatus, CompleteOrder
  - Composition points marked with comments:
    - `CreateOrder.pre` depends on `Inventory.ReserveStock.post`
    - `ConfirmOrder.post` guarantees preconditions for `Payment.InitiatePayment.pre`

- **[.vdm/payment.vdmsl](.vdm/payment.vdmsl)** (284 lines)
  - Types: PaymentId, PaymentStatus (<PENDING|AUTHORIZED|CAPTURED|REFUNDED|FAILED>), PaymentMethod, Payment, AuthorizationResponse
  - State: payments mapping, nextPaymentId, pendingRefunds
  - Operations: InitiatePayment, ProcessPayment, RefundPayment, GetPaymentStatus, CompletePayment
  - Composition points marked with comments:
    - `InitiatePayment.pre` depends on `Order.ConfirmOrder.post`
    - `RefundPayment.post` enables `Inventory.ReleaseStock.pre`

### Integration & Verification
- **[.vdm/integration-check.md](.vdm/integration-check.md)** (395 lines)
  - The KEY document for understanding module composition
  - Three verified composition points:
    1. Inventory.ReserveStock → Order.CreateOrder
    2. Order.ConfirmOrder → Payment.InitiatePayment
    3. Payment.RefundPayment → Inventory.ReleaseStock
  - For each composition point:
    - Shows the operation chain
    - Verifies preconditions and postconditions match
    - Provides code patterns demonstrating safe composition
    - Includes test cases

---

## How to Study This Example

### Phase 1: Understanding the Architecture (15-20 minutes)
1. Read [README.md](./README.md) - Overview section only
2. Study the system architecture diagram
3. Understand the three module responsibilities

### Phase 2: Learning VDM-SL Syntax (30-40 minutes)
1. Study [.vdm/inventory.vdmsl](.vdm/inventory.vdmsl)
   - This is the simplest module
   - Understand: types, state, operations, pre/post conditions
   - Note the invariants and how they're expressed

2. Study [.vdm/order.vdmsl](.vdm/order.vdmsl)
   - Build on Inventory understanding
   - See how Order depends on Inventory (via pre/post conditions)
   - Understand the state machine: PENDING → CONFIRMED → COMPLETED

3. Study [.vdm/payment.vdmsl](.vdm/payment.vdmsl)
   - Complete the module trio
   - See how Payment depends on Order

### Phase 3: Understanding Composition (20-30 minutes)
1. Open [.vdm/integration-check.md](.vdm/integration-check.md)
2. For each composition point, verify:
   - Does A.post logically imply B.pre?
   - What could go wrong if this wasn't checked?
3. Study the code patterns for safe composition
4. Review the test cases

### Phase 4: Generating Implementations (optional, 1-2 hours)
1. Use the prompts in [README.md](./README.md)
2. Start with Inventory module (simplest)
3. Then Order module (requires Inventory interface)
4. Then Payment module (requires Order interface)

---

## Key Concepts in This Example

### 1. Module Independence
- Each module is fully specified in isolation
- No implementation details leak between modules
- Only interfaces are visible (operations and types)

### 2. Formal Contracts
- Every operation has a contract: pre/post conditions
- Pre-conditions: what the module assumes is true before the operation
- Post-conditions: what the module guarantees is true after the operation
- Invariants: what must always be true about the state

### 3. Composition Verification
- When module A's operation output feeds into module B's operation input
- Must verify: A.post ⇒ B.pre (mathematically)
- This prevents "integration surprises" at implementation time

### 4. State Management
- Each module manages its own state (inventory, orders, payments)
- State changes are atomic within an operation
- Invariants are maintained before/after every operation

### 5. Exception Handling
- Modules return exceptions instead of throwing
- Exceptions propagate up as return values
- Caller decides how to handle exceptions

---

## Common Questions

**Q: Why VDM-SL?**
A: VDM-SL is expressive enough for complex invariants but simple enough to read. It's widely used in formal methods and well-supported by tools.

**Q: Why check composition manually?**
A: Formal verification tools exist, but manual verification teaches the concepts. A real project would use automated tools.

**Q: What's the point of specifications if I'll implement anyway?**
A: Specifications:
1. Make bugs obvious before you write code
2. Serve as executable requirements for AI code generation
3. Enable composition verification
4. Provide a contract for tests and integration

**Q: Can I use a different language than VDM-SL?**
A: Yes! You could use Z notation, Alloy, TLA+, etc. The important thing is formal pre/post conditions and invariants.

**Q: How do I scale this to more modules?**
A: 
1. Add new module with full VDM-SL spec
2. For each new module, identify what it depends on (other modules)
3. Find the composition points
4. Verify: existing module.post ⇒ new module.pre
5. Add to integration-check.md

---

## File Statistics

```
README.md                     369 lines (bilingual overview + prompts)
.vdm/inventory.vdmsl         211 lines (simplest module)
.vdm/order.vdmsl             252 lines (middle complexity)
.vdm/payment.vdmsl           284 lines (complete module)
.vdm/integration-check.md    395 lines (composition verification)
─────────────────────────────────────
Total                       1511 lines

Typical reading time:
- Quick overview: 15 minutes
- Full study: 2-3 hours
- Implementation generation: 1-2 hours additional
```

---

## Next Steps After This Example

1. **Adapt to your domain**: Replace products/orders/payments with your domain entities
2. **Add more modules**: Payment processors, inventory forecasting, etc.
3. **Generate code**: Use the prompts to generate implementations
4. **Write integration tests**: Based on composition verification
5. **Document lessons learned**: Update this structure for your team

This example is designed to be a reference. Copy it, modify it, and make it your own.
