# Complete Usage Example: From Specification to Deployment

This document shows a complete end-to-end workflow using all four prompt templates.

## Scenario

**System:** E-Commerce Inventory Management System  
**Context:** Small startup building MVP, later scaling to enterprise  
**Team:** 3 developers, 1 tech lead, familiar with VDM-SL concepts  
**Timeline:** 2 weeks for MVP (spec → design → code → verify)  

---

## Phase 1: Specification Dialogue

### Setup

**File to use:** `phase1-specification.md`

**Parameters:**
```
{{SYSTEM_NAME}} = "E-Commerce Inventory Management"
{{DOMAIN}} = "e-commerce"
{{PRIMARY_ACTORS}} = "warehouse managers, fulfillment staff, catalog managers"
{{KEY_RISKS}} = "overselling, stock inconsistency across locations, race conditions"
{{LANGUAGE}} = "en"
{{OUTPUT_FORMAT}} = "vdmsl file + summary"
```

### Step 1: Copy System Prompt

Tech lead copies the entire system prompt from `phase1-specification.md` into Claude.

### Step 2: Start Dialogue

**Tech Lead:** 
> I want to formally specify an inventory management system. Here's the context:
> - Domain: E-commerce (multiple warehouses)
> - Key actors: Warehouse managers, fulfillment staff
> - Primary risk: We're afraid of overselling — selling more than we have
> - Scale: MVP is 1,000 products, ~100 orders/sec
> - Language: English
> 
> Let's start from scratch and build the specification together.

**Claude (Phase 1 prompt active):**
> Great! Let me help you build a rigorous specification. Before we dive into VDM-SL, I need to understand your domain better.
> 
> **Domain Understanding:**
> 1. You mentioned multiple warehouses. Does each warehouse track inventory independently, or is there a global pool?
> 2. When a customer places an order, do you immediately reserve stock, or do you only check availability at fulfillment time?
> 3. Can products have variants (sizes, colors), or is every SKU a single item?
> 4. What's your biggest fear operationally? Overselling to a customer? Or stock becoming inconsistent?

**Tech Lead:**
> Good questions. 
> 1. Multiple warehouses with independent stock levels, but we can fulfill from any warehouse
> 2. We want to reserve stock immediately when the order is placed
> 3. Yes, products have variants (like "T-Shirt size M" is one variant)
> 4. Overselling is the nightmare scenario — selling more than we have in total

**Claude:**
> Perfect. This tells me we need to model:
> - Variants as atomic inventory units
> - Orders as immediate stock reservations
> - Cross-warehouse coordination (so we don't oversell globally)
> 
> Let me start with type definitions. 
> 
> **I propose a Variant type:**
> ```vdm
> VariantId = token;
> Quantity = nat inv q == q > 0;
> ```
> 
> In plain English: A variant is identified by a unique ID (like "TSHIRT-M-BLK"). Quantity is a positive number of units.
> 
> But wait—I need to clarify: what's the maximum stock for a single variant? Are you worried about the number getting too large?

**Tech Lead:**
> Good point. Let's cap it at 1 million units per variant. That's plenty.

**Claude:**
> Got it. So I'll refine:
> ```vdm
> Quantity = nat inv q == q > 0 and q <= 1000000;
> ```
> 
> Now, does this match your intent? Should every variant start with at least 1 unit, or can we have zero-stock variants registered (for coming soon)?

**Tech Lead:**
> Initial stock should always be at least 1. But we should also support adding stock later (restocking).

**Claude:**
> Excellent. Now let me move to state. The system holds:
> - A map from VariantId to current stock (Quantity)
> - A log of orders (for audit/reporting)
> 
> The invariant is critical: **total stock across all variants can never increase by processing orders alone** (only by restocking). And we can never sell more than we have.
> 
> Shall I continue with operation definitions?

...[dialogue continues through Phase 1 protocol]...

### Step 3: Phase 1 Output

**Claude produces:**

1. **VDM-SL Specification** (spec.vdmsl):
```vdm
module InventoryManagement
exports all
definitions

types
  VariantId = token;
  Quantity = nat inv q == q > 0 and q <= 1000000;
  WarehouseId = token;
  OrderId = token;

  Inventory = map VariantId to Quantity;
  
  Order ::
    orderId    : OrderId
    variantId  : VariantId
    quantity   : Quantity
    timestamp  : Timestamp
  inv mk_Order(-, -, qty, -) == qty > 0;

state InventoryState of
  variants   : map VariantId to Inventory
  orders     : seq of Order
inv inv_NoOverselling(variants, orders) ==
  forall o in set elems orders &
    let availStock = variants(o.variantId) in
      availStock >= o.quantity

operations

  RegisterVariant: VariantId * Quantity ==> ()
  RegisterVariant(variantId, initialStock) ==
    (variants(variantId) := initialStock;
     orders := orders ^ [])
  pre variantId not in set dom variants and initialStock > 0
  post variants(variantId) = initialStock

  ProcessOrder: VariantId * Quantity ==> bool
  ProcessOrder(variantId, qty) ==
    let available = variants(variantId) in
      if available >= qty
      then (variants(variantId) := available - qty; return true)
      else return false
  pre variantId in set dom variants and qty > 0
  post (if RESULT then variants(variantId) = variants~(variantId) - qty
        else variants = variants~)

end InventoryManagement
```

2. **Specification Summary** (English, plain language):
> **System:** E-Commerce Inventory Management  
> **Overview:** This system tracks product inventory across variants and prevents overselling through atomic stock reservations.
>
> **Types:**
> - VariantId: Unique product variant identifier (e.g., "TSHIRT-M-BLK")
> - Quantity: Stock level, always positive, capped at 1 million units
> - Order: A customer order with timestamp and quantity
>
> **State:**
> - variants: Map from VariantId to current stock
> - orders: Immutable log of all orders
> - Invariant: For every order, stock was available (no overselling possible)
>
> **Operations:**
> - RegisterVariant: Add a new variant to inventory with initial stock
> - ProcessOrder: Attempt to reserve stock for an order
>   - Success: Stock decreases by ordered quantity
>   - Failure: Stock unchanged
>
> **Design Questions Surfaced:**
> - How do we handle concurrent orders? (Answer in Phase 2)
> - Do we need to track warehouse location? (Decision needed in Phase 2)
> - How long to keep order history? (Policy decision in Phase 2)

**Deliverable:** `inventory-spec.vdmsl` + `inventory-spec-summary.md`

---

## Phase 2: Technical Design

### Setup

**File to use:** `phase2-design.md`

**Parameters:**
```
{{VDMSL_SPEC}} = [from Phase 1]
{{NON_FUNCTIONAL_REQUIREMENTS}} = {
  "scale": "1,000 orders/sec target",
  "latency": "p99 < 200ms",
  "availability": "99.9% uptime",
  "cost": "AWS budget $5k/month"
}
{{TEAM_EXPERTISE}} = "Python/FastAPI, PostgreSQL comfortable"
{{DEPLOYMENT_CONSTRAINT}} = "AWS (ECS or Kubernetes)"
{{LANGUAGE}} = "en"
```

### Step 1: Copy System Prompt

Tech lead copies Phase 2 system prompt into Claude.

### Step 2: Start Design Dialogue

**Tech Lead:**
> Here's our VDM-SL spec from Phase 1. Now design the system for:
> - 1,000 orders/sec (MVP can start lower, but design for scale)
> - <200ms p99 latency
> - 99.9% uptime (important for customer trust)
> - AWS, $5k/month budget
> - Our team knows Python/FastAPI and PostgreSQL
> 
> Tell me: what architecture do you propose?

**Claude (Phase 2 prompt active):**
> Based on your spec and constraints, here's my proposal:
>
> **MVP Architecture:**
> - Single-service monolith (Python FastAPI)
> - PostgreSQL for durable state
> - Redis for caching hot variants (top 10% of orders hit these)
> - Docker on ECS for deployment
>
> **Why:**
> - Simpler than microservices for MVP
> - Python/FastAPI is what your team knows
> - PostgreSQL ACID transactions guarantee the VDM-SL atomicity property
> - Redis caches 80% of requests (improves latency)
> - ECS is cheaper than Kubernetes at this scale
>
> **Trade-off:** Single service means inventory and orders share resources. If stock checking is slow, order processing is slow. We can split to microservices later.
>
> **For scale (1k ops/sec):** We'll need read replicas on PostgreSQL. At 1k ops/sec, a single primary will bottleneck writes. Plan for multi-region failover in Phase 2.5 (not MVP).
>
> Does this direction feel right? Or should we plan for microservices now?

**Tech Lead:**
> Monolith for MVP makes sense. But tell me: how do you prevent overselling when orders come in fast? Race conditions?

**Claude:**
> Excellent question—this is where the VDM-SL spec shines.
>
> The spec says ProcessOrder is atomic: if stock >= qty, decrease stock AND confirm order. No interleaving.
>
> **Design to guarantee atomicity:**
> ```
> BEGIN TRANSACTION;
>   SELECT inventory.stock WHERE variant_id = X FOR UPDATE;  -- Row lock
>   IF stock >= qty THEN
>     UPDATE inventory SET stock = stock - qty WHERE variant_id = X;
>     INSERT INTO orders (...) VALUES (...);
>     COMMIT;  -- Lock released, atomicity guaranteed
>   ELSE
>     ROLLBACK;  -- No changes
> END;
> ```
>
> The `FOR UPDATE` lock ensures no two transactions race on the same variant. PostgreSQL serializes them. The VDM-SL post-condition is satisfied: either stock decreases by exactly qty, or nothing changes.
>
> This scales to 1k ops/sec on a single PostgreSQL instance (write throughput is ~5-10k TPS for simple queries). No race conditions possible.

**Tech Lead:**
> Great. What about the monitoring and alerting?

**Claude:**
> Good thinking. Here's what I'd add to Phase 2 design:
>
> **Monitoring:**
> - Log every order (timestamp, variant, qty, success/failure)
> - Metrics: orders/sec, success rate, p50/p99 latency
> - Alerts: if order success rate < 95%, if p99 > 500ms, if stock ever goes negative (invariant violation!)
>
> **Invariant Monitoring:**
> - Every 10 minutes, sum all stock and check it never exceeds initial total
> - Alert if invariant violated (would indicate a bug)
>
> This is non-functional (not in VDM-SL), but essential for production stability.

...[design dialogue continues]...

### Step 3: Phase 2 Output

**Claude produces:**

1. **Design Document** (design.md):
```markdown
# Inventory Management System - Technical Design

## Architecture
- Monolith service (Python FastAPI)
- PostgreSQL primary + 1 read replica
- Redis for cache (hot variants)
- ECS deployment

## Technology Stack
- Language: Python 3.11
- Framework: FastAPI (async, fast)
- ORM: SQLAlchemy
- Database: PostgreSQL 15
- Cache: Redis 7
- Testing: pytest + Hypothesis
- Deployment: Docker + ECS

## Data Model
```sql
CREATE TABLE variants (
  variant_id UUID PRIMARY KEY,
  sku VARCHAR(50) NOT NULL UNIQUE,
  stock INT NOT NULL CHECK (stock > 0 AND stock <= 1000000),
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
  order_id UUID PRIMARY KEY,
  variant_id UUID NOT NULL REFERENCES variants(variant_id),
  quantity INT NOT NULL CHECK (quantity > 0),
  status VARCHAR(20) DEFAULT 'confirmed',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Service Design

### ProcessOrder Operation
```
API: POST /orders/process
Input: { variant_id, quantity }

Service Flow:
  1. Validate input (qty > 0)
  2. Check Redis cache (is stock available?)
  3. If cache hit, proceed; if miss, fetch from DB
  4. Start transaction with row lock (SELECT ... FOR UPDATE)
  5. Check stock again (cache might be stale)
  6. If stock >= qty:
       - Decrement stock: UPDATE variants SET stock = stock - qty
       - Create order: INSERT INTO orders
       - COMMIT (lock released)
       - Invalidate Redis cache for this variant
       - Return { success: true }
     Else:
       - ROLLBACK (lock released)
       - Return { success: false }
  7. Log operation (for audit)

Invariant Enforcement:
  - Database CHECK constraint: stock >= 0
  - Row-level lock: no concurrent updates to same variant
  - Transaction: atomicity guaranteed
  - Monitoring: every 10m, verify total stock never exceeds initial
```

## Design Decisions & Trade-offs

| Decision | Why | Trade-off |
|----------|-----|-----------|
| Monolith | Simpler for MVP, team knows stack | Single point of failure; must scale vertical before horizontal |
| PostgreSQL | ACID + row locks = atomicity | Not as fast as NoSQL for reads; need read replicas for 1k ops/sec |
| Redis cache | 80% of orders hit 10% of variants | Cache invalidation is hard; stale data possible (but we check DB again) |
| Row-level locking | Prevents race conditions | Might block other transactions on same variant |
| ECS not Kubernetes | Cheaper at MVP scale, simpler ops | Fewer out-of-the-box features (custom monitoring) |

## Invariant Mapping

VDM-SL says: "For every order, stock was available (no overselling)"

How we enforce it:
- Database: CHECK constraint + row lock
- Application: Validation before transaction
- Monitoring: Daily invariant check (sum stock, verify consistency)

If invariant violated, alert immediately (indicates a bug).
```

2. **Architecture Diagram** (visual representation showing components, data flows)

**Deliverable:** `inventory-design.md`

---

## Phase 3: Implementation

### Setup

**File to use:** `phase3-implementation.md`

**Parameters:**
```
{{VDMSL_SPEC}} = [from Phase 1]
{{DESIGN_DOC}} = [from Phase 2]
{{LANGUAGE}} = "Python"
{{FRAMEWORK}} = "FastAPI"
{{DATABASE}} = "PostgreSQL"
{{OUTPUT_STYLE}} = "Module-by-module with tests"
```

### Step 1: Copy System Prompt

Tech lead copies Phase 3 system prompt into Claude.

### Step 2: Start Implementation Dialogue

**Tech Lead:**
> Here's the spec and design. Now let's implement. 
> - Language: Python 3.11
> - Framework: FastAPI
> - Database: PostgreSQL
> 
> Start with types and work your way to API endpoints. Include tests for each module.

**Claude (Phase 3 prompt active):**
> I'll implement module-by-module: types → repository → service → API → tests.
>
> **Step 1: Type Definitions**
> 
> Here's how I'll encode the VDM-SL types so invalid states are impossible:
>
> ```python
> from typing import NewType
> import uuid
>
> VariantId = NewType('VariantId', str)
> Quantity = NewType('Quantity', int)
> OrderId = NewType('OrderId', str)
>
> def validate_quantity(q: int) -> Quantity:
>     """Enforce Quantity invariant: 0 < q <= 1,000,000"""
>     if q <= 0 or q > 1_000_000:
>         raise ValueError(f"Quantity must be 1-1,000,000, got {q}")
>     return Quantity(q)
>
> class Variant:
>     """Type representing a product variant with invariant enforcement."""
>     def __init__(self, variant_id: VariantId, sku: str, stock: Quantity):
>         if not variant_id or not isinstance(variant_id, str):
>             raise TypeError("variant_id must be a non-empty string")
>         if not sku or len(sku) > 50:
>             raise ValueError("SKU must be 1-50 chars")
>         if not isinstance(stock, Quantity):
>             raise TypeError("stock must be Quantity type")
>         
>         self.variant_id = variant_id
>         self.sku = sku
>         self.stock = stock
>
>     def __repr__(self):
>         return f"Variant(id={self.variant_id}, sku={self.sku}, stock={self.stock})"
> ```
>
> This makes it impossible to create a Variant with invalid stock (the invariant is in `__init__`).

**Tech Lead:**
> Good. Now the repository layer.

**Claude:**
> **Step 2: Repository (Data Access)**
>
> ```python
> from sqlalchemy.orm import Session
>
> class VariantModel(Base):
>     """ORM model for variants. VDM-SL state variable."""
>     __tablename__ = 'variants'
>
>     variant_id = Column(String, primary_key=True)
>     sku = Column(String(50), nullable=False, unique=True)
>     stock = Column(Integer, nullable=False)
>     updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
>
>     __table_args__ = (
>         CheckConstraint('stock > 0 AND stock <= 1000000', name='check_stock_valid'),
>     )
>
> class VariantRepository:
>     """Repository for variants. Maps VDM-SL operations to DB queries."""
>
>     def __init__(self, session: Session):
>         self.session = session
>
>     def register_variant(self, variant_id: VariantId, sku: str, initial_stock: Quantity) -> None:
>         """
>         VDM-SL RegisterVariant operation.
>         
>         Pre-condition: variant must not already exist, initial_stock > 0
>         Post-condition: variant now in inventory with initial_stock
>         """
>         # Pre-condition check
>         existing = self.session.query(VariantModel).filter(
>             VariantModel.variant_id == str(variant_id)
>         ).one_or_none()
>
>         if existing:
>             raise ValueError(f"Variant {variant_id} already exists")
>
>         # Create and persist
>         model = VariantModel(
>             variant_id=str(variant_id),
>             sku=sku,
>             stock=int(initial_stock)
>         )
>         self.session.add(model)
>         self.session.commit()
>
>     def process_order(self, variant_id: VariantId, qty: Quantity) -> bool:
>         """
>         VDM-SL ProcessOrder operation. 
>         Atomically reserves stock if available.
>         
>         Pre-condition: variant exists, qty > 0
>         Post-condition: if sufficient stock, decreases by qty and returns true;
>                        else stock unchanged and returns false
>         """
>         # Pre-condition: qty must be positive
>         if qty <= 0:
>             raise ValueError(f"Order qty must be > 0, got {qty}")
>
>         try:
>             # Atomicity: row-level lock + transaction
>             model = self.session.query(VariantModel).filter(
>                 VariantModel.variant_id == str(variant_id)
>             ).with_for_update().one()  # Lock this row
>
>             if model.stock >= int(qty):
>                 # Sufficient stock: decrement and commit
>                 model.stock -= int(qty)
>                 model.updated_at = datetime.utcnow()
>                 self.session.commit()  # Lock released, changes persisted
>                 return True
>             else:
>                 # Insufficient stock: rollback (nothing changed)
>                 self.session.rollback()
>                 return False
>
>         except NoResultFound:
>             raise ValueError(f"Variant {variant_id} not found")
> ```
>
> The `with_for_update()` ensures the database locks this row, preventing race conditions. The transaction ensures atomicity (VDM-SL post-condition).

...[implementation continues through service and API layers]...

### Step 3: Phase 3 Output

**Claude produces:**

1. **Code modules:**
   - `src/types.py` — Type definitions
   - `src/models.py` — ORM models
   - `src/repositories/variant_repo.py` — Data access
   - `src/services/inventory_service.py` — Business logic
   - `src/api/routers/variants.py` — API endpoints
   - `tests/test_types.py` — Type invariant tests
   - `tests/test_repositories.py` — Integration tests
   - `tests/test_services.py` — Service tests
   - `tests/test_api.py` — API contract tests

2. **Example test output:**

```python
# tests/test_repositories.py

def test_process_order_post_decreases_stock(db_session):
    """
    VDM-SL post-condition: if stock >= qty, stock decreases by qty.
    
    Verify: stock(before) - qty = stock(after)
    """
    repo = VariantRepository(db_session)
    repo.register_variant(
        variant_id=VariantId("TSHIRT-M"), 
        sku="TS-001",
        initial_stock=Quantity(50)
    )

    result = repo.process_order(
        variant_id=VariantId("TSHIRT-M"),
        qty=Quantity(10)
    )

    assert result is True
    model = db_session.query(VariantModel).filter(
        VariantModel.variant_id == "TSHIRT-M"
    ).one()
    assert model.stock == 40  # 50 - 10

def test_process_order_post_nothing_changes_on_failure(db_session):
    """
    VDM-SL post-condition: if stock < qty, stock unchanged.
    """
    repo = VariantRepository(db_session)
    repo.register_variant(
        variant_id=VariantId("TSHIRT-L"),
        sku="TS-002",
        initial_stock=Quantity(5)
    )

    result = repo.process_order(
        variant_id=VariantId("TSHIRT-L"),
        qty=Quantity(10)  # More than available
    )

    assert result is False
    model = db_session.query(VariantModel).filter(
        VariantModel.variant_id == "TSHIRT-L"
    ).one()
    assert model.stock == 5  # Unchanged
```

**Deliverable:** Full working codebase in `src/` and `tests/`

---

## Phase 4: Verification

### Setup

**File to use:** `phase4-verification.md`

**Parameters:**
```
{{VDMSL_SPEC}} = [from Phase 1]
{{IMPLEMENTATION}} = [from Phase 3]
{{TEST_FRAMEWORK}} = "pytest + Hypothesis"
{{COVERAGE_TARGET}} = "95%"
{{LANGUAGE}} = "en"
```

### Step 1: Copy System Prompt

Tech lead copies Phase 4 system prompt into Claude.

### Step 2: Start Verification

**Tech Lead:**
> Now verify the implementation. Generate comprehensive tests proving it satisfies the spec. Use pytest + Hypothesis. Aim for 95% coverage.

**Claude (Phase 4 prompt active):**
> I'll generate tests for:
> 1. Pre-condition validation (invalid inputs rejected)
> 2. Post-condition verification (successful operations guarantee contracts)
> 3. Invariant preservation (state invariants hold before and after)
> 4. Properties (universal properties for all inputs)
> 5. Concurrency (atomicity guarantees)
> 6. Edge cases (boundaries, absence, temporal)
> 7. Cross-module interfaces (if any)
>
> Let me start.

**Claude produces:**

1. **Comprehensive test suite** with:
   - Pre-condition tests (10+ tests)
   - Post-condition tests (15+ tests)
   - Invariant tests (8+ tests)
   - Property-based tests (5+ properties)
   - Concurrency tests (3+ scenarios)
   - Edge case tests (12+ cases)

2. **Verification Report:**

```
# Formal Verification Report

## Executive Summary
System: E-Commerce Inventory Management
VDM-SL Spec: inventory-spec.vdmsl
Implementation: src/ (Python/FastAPI)
Test Date: 2026-03-28

Result: ✅ PASSED (71 tests green, 0 failed)
Coverage: 98% (above target of 95%)

---

## Test Coverage

### Pre-Condition Tests: ✅ 12/12 Passed
- [x] RegisterVariant: duplicate variant rejected
- [x] RegisterVariant: zero initial stock rejected
- [x] ProcessOrder: missing variant rejected
- [x] ProcessOrder: zero quantity rejected
- [x] ProcessOrder: negative quantity rejected
- [x] ProcessOrder: qty > max (1M) rejected
- ... (6 more)

### Post-Condition Tests: ✅ 18/18 Passed
- [x] ProcessOrder success: stock decreases by qty
- [x] ProcessOrder failure: stock unchanged
- [x] ProcessOrder success: other variants unaffected
- [x] RegisterVariant: variant now exists in DB
- [x] RegisterVariant: can check stock after register
- ... (13 more)

### Invariant Tests: ✅ 8/8 Passed
- [x] inv_StockPositive holds after RegisterVariant
- [x] inv_StockPositive holds after ProcessOrder (success)
- [x] inv_StockPositive holds after ProcessOrder (failure)
- [x] inv_StockPositive holds after any operation sequence
- [x] inv_TotalStockNeverIncreases holds
- ... (3 more)

### Property-Based Tests: ✅ 5/5 Properties (500+ cases)
- [x] For any (initial_stock, order_qty), result is correct
- [x] Total withdrawn stock <= initial stock (no overselling)
- [x] Invariant holds for 500+ random operation sequences
- [x] Stock never negative after any valid operation
- [x] Concurrent orders don't violate atomicity

### Concurrency Tests: ✅ 3/3 Passed
- [x] Concurrent orders on same variant: atomicity guaranteed
- [x] Concurrent orders on different variants: no interference
- [x] Race condition impossible (row-level lock verified)

### Edge Case Tests: ✅ 12/12 Passed
- [x] Register with stock = 1 (boundary)
- [x] Register with stock = 1,000,000 (boundary)
- [x] Order quantity = 1 (boundary)
- [x] Order quantity = available (boundary)
- [x] Empty inventory (absence)
- [x] Missing variant (absence)
- ... (6 more)

### Cross-Module Tests: ✅ 4/4 Passed
- (If multi-module system)

---

## Results Summary

| Category | Passed | Failed | Skipped |
|----------|--------|--------|---------|
| Pre-conditions | 12 | 0 | 0 |
| Post-conditions | 18 | 0 | 0 |
| Invariants | 8 | 0 | 0 |
| Properties | 5 | 0 | 0 |
| Concurrency | 3 | 0 | 0 |
| Edge Cases | 12 | 0 | 0 |
| Cross-Module | 4 | 0 | 0 |
| **Total** | **62** | **0** | **0** |

---

## Coverage Metrics

- Line coverage: 98% (above 95% target)
- Branch coverage: 95% (all decision points tested)
- Post-condition coverage: 100% (all VDM-SL guarantees verified)

---

## Conclusion

✅ **VERIFIED:** Implementation provably satisfies VDM-SL specification.

All pre-conditions properly rejected, all post-conditions verified, all invariants maintained, concurrency safe, no edge cases missed.

**Ready for production deployment.**
```

**Deliverable:** `verification-report.md`

---

## Summary

| Phase | Input | Output | Key Achievement |
|-------|-------|--------|-----------------|
| 1 (Spec) | Business requirements | VDM-SL formal spec | Precise, unambiguous requirements |
| 2 (Design) | VDM-SL spec | Architecture + design doc | Technology choices justified by spec |
| 3 (Code) | Design doc | Production code + tests | Code structure that makes bugs impossible |
| 4 (Verify) | Code + spec | Verification report | Proof that code satisfies spec |

**Total time:** 2 weeks (MVP)  
**Confidence:** High (formal verification)  
**Future scaling:** Design already accounts for 1k ops/sec; implementation ready for horizontal scaling

---

## Next Steps

1. Deploy to production (Phase 4 verification confirms correctness)
2. Monitor invariants (stock never negative; total stock consistent)
3. If bugs found: update VDM-SL first, then design, then code, then verify
4. Plan Phase 2.5 (multi-region failover, if needed)
5. Re-run verification after any code changes

**Success:** Started with unclear requirements, ended with formally verified system ready for production.
