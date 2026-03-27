# Multi-Agent Orchestration Workflow Guide
# マルチエージェント・オーケストレーション・ワークフロー・ガイド

**Bilingual: English (top) / 日本語 (bottom)**

---

## Table of Contents

1. [English Section](#english-section)
2. [日本語セクション](#日本語セクション)

---

# ENGLISH SECTION

## Overview

This guide explains how to orchestrate multiple AI agents in a formal-specification-driven development workflow. The key insight: **agents communicate through formal specifications (VDM-SL), not code**.

### Why Multi-Agent Orchestration?

Traditional single-agent AI development hits scalability limits:
- One agent cannot hold an entire system in context
- Handoff between "spec → design → implementation → testing" is error-prone
- No clear contract verification between components

**Multi-agent approach:**
- **Architect agent** (Phase 1): Defines system spec and module boundaries
- **Module agents** (Phase 2-4): Develop modules in parallel, using only:
  - Their own module spec
  - Interface specs of dependencies (postconditions → preconditions)
- **Integration agent** (Phase 4): Verifies contracts between modules

### Core Principle: Formal Specs as Contracts

```
Module A (Order)              Module B (Payment)
┌──────────────────┐         ┌──────────────────┐
│ finalize_order   │         │ charge           │
│ postcond:        │         │ precond:         │
│ order.status=    │ ─────→  │ payment > 0  ✓   │
│ "finalized"      │         │ payment.method   │
│                  │         │ != null      ✓   │
└──────────────────┘         └──────────────────┘

If A's postcond ⇒ B's precond, they compose!
```

This is the contract verification step. If all modules satisfy their specs, and contracts verify, the system works.

---

## Phase Overview

### Phase 1: System Architecture & Specification (Human + Architect)

**Duration:** 1-3 days
**Key Players:** Tech lead, domain experts, architect agent
**Deliverable:** Complete VDM-SL system specification + module decomposition

**What Happens:**

1. **Architect agent dialogues** with human stakeholders
2. **Elicits business requirements** and domain constraints
3. **Proposes types and invariants** (business rules encoded as VDM-SL)
4. **Defines operations** with pre/post conditions
5. **Decomposes into modules** with clear boundaries
6. **Produces formal spec** that will guide Phase 2-4

**Key Deliverables:**

```
specs/
├── .vdm/
│   ├── system-spec.vdmsl         # Everything
│   ├── order-module.vdmsl        # Extract for order module
│   ├── inventory-module.vdmsl    # Extract for inventory module
│   ├── payment-module.vdmsl      # Extract for payment module
│   └── shared-types.vdmsl        # Types shared across modules
│
└── module-decomposition.md       # Which module owns what
```

**Human Checkpoints:**

- ✅ After domain types: "Do these types capture our business rules?"
- ✅ After module boundaries: "Is each module's scope correct?"
- ✅ After contracts: "Are pre/post conditions correct?"
- ✅ Final approval: Sign off on spec before moving to Phase 2

**Practical Implementation (TODAY):**

Use **Claude Projects** with persistent context:

1. Start a Claude Project: "Order System Formal Spec"
2. Upload template: `templates/prompts/phase1-specification.md`
3. Include template: `templates/vdm-sl/module-template.vdmsl`
4. Start dialogue with architect agent:
   ```
   You are a formal specification architect.
   [Insert phase1-specification.md content]

   Let's define our Order Processing system...
   ```
5. As agent produces VDM-SL, iterate until stakeholder approves
6. Export final specs to `specs/.vdm/` directory

---

### Phase 2: Module Design (Per-Module Agents, Parallel)

**Duration:** 1-2 days per module
**Key Players:** One module agent per module
**Deliverable:** Detailed design + operation decomposition for each module

**What Happens:**

Each module agent independently designs their module:
1. **Receives:** Own module spec + interface specs of dependencies
2. **Asks:** "How should I structure this module internally?"
3. **Produces:** Design document with operation breakdown
4. **Constraints:**
   - Must respect own module spec (pre/post conditions)
   - Must not violate dependency contracts
   - Must ensure composability

**Example: Order Module Agent**

```
Input:
- Own spec: order-module.vdmsl (create_order, finalize_order, etc.)
- Dependency specs:
  - inventory.check_stock(product_id) → {available: nat}
  - payment.charge(amount) → {transaction_id: UUID, status: PaymentStatus}
- Shared types: Customer, Product, Money

Output:
- Design document:
  1. How will we structure the order state machine?
  2. Which operation calls inventory.check_stock and when?
  3. Which operation calls payment.charge and in what state?
  4. How do we handle failures (inventory unavailable, payment declined)?
```

**Key Deliverables:**

```
modules/order/
├── DESIGN.md                  # Design decisions, operation flow
├── preconditions.txt          # Preconditions for each operation
├── postconditions.txt         # Postconditions for each operation
└── dependencies.txt           # Which ops call which external modules
```

**Human Checkpoints:**

- ✅ Design review: "Does this design approach make sense?"
- ✅ Dependency check: "Are contract dependencies clear?"

**Practical Implementation (TODAY):**

For each module, create a new Claude Project or conversation:

```yaml
For order module:
  Prompt: "You are an expert module designer."
  Context:
    - Order module spec (from Phase 1)
    - Inventory interface spec (postconditions only)
    - Payment interface spec (postconditions only)
    - Phase 2 design prompt (templates/prompts/phase2-design.md)

  Task: "Design the internal structure of the order module to implement
         the spec while respecting the inventory and payment contracts."
```

Each module agent:
1. Opens Phase 2 design prompt
2. Receives own spec + dependency interface specs
3. Produces design doc
4. Waits for human approval before Phase 3

**Can run in parallel:** All three module agents work simultaneously (no dependencies between them).

---

### Phase 3: Implementation (Per-Module Agents, Parallel)

**Duration:** 2-5 days per module
**Key Players:** One module agent per module
**Deliverable:** Source code + unit tests for each module

**What Happens:**

Each module agent implements their design:

1. **Receives:** Design doc + own module spec + shared types
2. **Generates:** Implementation code in target language (Python, TypeScript, etc.)
3. **Auto-generates:** Unit tests based on preconditions
4. **Verifies:** "Does my code satisfy the spec?"
5. **Output:** Source code that can be reviewed and tested

**Example: Order Module Implementation**

```python
# Phase 3 output might look like:

def create_order(customer: Customer, items: List[OrderItem]) -> Order:
    """
    Precondition (from spec):
    - customer.id must be valid (exists in system)
    - items must not be empty
    - each item.quantity must be positive

    Postcondition (from spec):
    - returns Order with status="pending"
    - order.total = sum of item prices
    - order is recorded in system
    """
    # Implementation goes here...

def finalize_order(order: Order) -> Order:
    """
    Precondition (from spec):
    - order.status == "pending"
    - order.total > 0

    Postcondition (from spec):
    - order.status == "finalized"
    - calls payment.charge(order.total)
    - calls inventory.deduct(order.items)
    - returns updated order
    """
    # Check preconditions
    assert order.status == "pending"
    assert order.total > 0

    # Call dependencies (via formal contracts)
    payment_result = payment.charge(order.total)
    if payment_result.status != "success":
        # What should happen if payment fails?
        # Spec should say...
        raise PaymentFailedError(...)

    inventory.deduct(order.items)

    # Update state
    order.status = "finalized"
    return order
```

**Key Deliverables:**

```
modules/order/
├── src/
│   ├── order.py               # Implementation
│   ├── types.py               # Local types
│   └── errors.py              # Error definitions
│
└── tests/
    ├── test_create_order.py    # Auto-generated from precond
    ├── test_finalize_order.py
    └── test_error_cases.py
```

**Human Checkpoints:**

- ✅ Code review: "Does code match design and spec?"
- ✅ Test verification: "Do tests align with preconditions?"
- ✅ Ready for integration: "Is this module ready to integrate?"

**Practical Implementation (TODAY):**

```yaml
For order module Phase 3:
  Create new conversation or continue project
  Prompt: "You are an expert module developer."
  Context:
    - Order design doc (from Phase 2)
    - Order module spec (from Phase 1)
    - Shared types and error patterns
    - Phase 3 implementation prompt (templates/prompts/phase3-implementation.md)
    - Example implementations from project

  Task: "Implement the order module in Python/TypeScript/Go.
         Generate unit tests based on preconditions.
         Verify your code satisfies the spec."
```

Each module agent:
1. Takes design doc + spec
2. Generates implementation
3. Generates tests
4. Submits for code review

**Can run in parallel:** All three module agents implement simultaneously.

**Critical for Success:**

- Each agent only knows about modules they call (dependency interfaces)
- Each agent does NOT need to know about other modules' implementation details
- Tests are derived from preconditions (not invented) → systematic coverage
- Code should explicitly check preconditions before calling external modules

---

### Phase 4: Integration & Verification (All Modules + Integration Agent)

**Duration:** 1-2 days
**Key Players:** Integration agent, all module agents, tech lead
**Deliverable:** Verified integrated system + deployment checklist

**What Happens:**

Integration agent performs the critical verification step:

```
For every inter-module call in the system:
  A.operation_postcondition ⇒ B.operation_precondition?

If ALL such implications are true → System is correct! ✓
If ANY fails → Identify which contract failed, fix it
```

**Example: Verifying Composability**

```
Order module calls: inventory.check_stock(product_id)
  Inventory postcondition: returns {available: nat, reserved: nat}
  Order precondition: expects {available: nat, reserved: nat}
  Verification: ✓ Postcond includes everything precond needs

Order module calls: payment.charge(amount)
  Payment postcondition: returns {transaction_id: UUID, status: PaymentStatus}
  Order precondition: expects {transaction_id: UUID, status: PaymentStatus}
  Verification: ✓ Postcond matches precond

All contracts verified → can integrate!
```

**Integration Agent Responsibilities:**

1. **Collect artifacts:**
   - All module implementations
   - All unit tests
   - All specifications
   - Integration test spec (system-level scenarios)

2. **Verify contracts:**
   - For each inter-module call: A.post ⇒ B.pre?
   - Document which modules compose, which don't
   - Flag any contract violations

3. **Generate integration tests:**
   - End-to-end scenarios based on system spec
   - Test happy paths (all contracts satisfied)
   - Test error paths (what if a module fails?)

4. **Produce integration report:**
   - Composability matrix (which pairs work)
   - Test results
   - Go/no-go recommendation

**Key Deliverables:**

```
integration/
├── composability-report.md     # Which contracts verified
├── integration-tests.py        # System-level tests
├── test-results.txt            # Pass/fail
└── deployment-checklist.md     # Go-no-go decision
```

**Practical Implementation (TODAY):**

```yaml
Phase 4 workflow:
  1. Architect agent: Extract system-level test spec
  2. Integration agent: Receives all modules + specs

  Phase4Prompt:
    Role: "Integration verification specialist"
    Context:
      - All module implementations
      - All module specs
      - System spec (from Phase 1)
      - Phase 4 verification prompt (templates/prompts/phase4-verification.md)

    Tasks:
      1. Contract verification: A.post ⇒ B.pre for every call
      2. Generate integration tests from system spec
      3. Verify tests pass
      4. Produce go/no-go report
```

**Manual Review Steps:**

1. Integration agent flags which contracts verified
2. If all verified + tests pass → integration agent recommends GO
3. If any failed → identify which module needs fixing, loop back to Phase 3
4. Tech lead reviews report and decides whether to deploy

---

## Cross-Cutting Concerns: Handling Shared Patterns

Some patterns span multiple modules (error handling, logging, auth, etc.). Define these in Phase 1, share with all agents.

### Example: Error Handling Contract

**Phase 1: Architect agent defines**

```vdmsl
-- In shared-types.vdmsl

types

  ErrorCode = <VALIDATION_ERROR> | <NOT_FOUND> | <CONFLICT> | <INTERNAL_ERROR> | <TIMEOUT>

  ApiError = {
    code: ErrorCode,
    message: seq of char,
    details: map seq of char to seq of char
  }

  Result[T] = Success(T) | Failure(ApiError)

end
```

**Phase 2-4: Module agents all use this pattern**

```python
# All module implementations use Result[T]

def create_order(customer: Customer, items: List[OrderItem]) -> Result[Order]:
    # Return Success(order) or Failure(error)
    ...

def charge(amount: Money) -> Result[Transaction]:
    # Return Success(transaction) or Failure(error)
    ...
```

**Benefit:** All modules handle errors the same way. Integration agent can verify error handling contracts.

### Example: Logging Contract

**Phase 1: Architect agent defines**

```vdmsl
types

  LogLevel = <DEBUG> | <INFO> | <WARN> | <ERROR>

  LogEntry = {
    timestamp: Time,
    level: LogLevel,
    operation: seq of char,
    user_id: UUID,
    message: seq of char
  }

end
```

**Phase 2-4: Module agents all emit logs using this format**

**Benefit:** Logging is structured across all modules. Easy to correlate events during integration testing.

### Example: Authentication Contract

**Phase 1: Architect agent defines**

```vdmsl
types

  User = {
    id: UUID,
    email: seq of char,
    roles: set of Role
  }

  AuthContext = {
    user: User,
    token: seq of char,
    exp_time: Time
  }

end

operations

  -- Shared function: verify auth before calling any operation
  check_auth(user: User) -> bool
    post RESULT = (user.id <> nil and user.email <> nil)

end
```

**Phase 2-4: Module agents check auth before processing requests**

```python
def create_order(auth_context: AuthContext, customer: Customer, items: List[OrderItem]) -> Result[Order]:
    # Precondition: auth must be valid
    assert auth_context.user is not None
    assert auth_context.token is not None
    assert auth_context.exp_time > now()

    # Then proceed with business logic...
    ...
```

**Benefit:** All modules enforce auth the same way. Integration agent can verify auth contracts.

---

## Shared Data Model Management

Shared types live in `specs/.vdm/shared-types.vdmsl`. Changes must be carefully managed.

**Change Process:**

1. Module agent identifies need for type change
2. Proposes change to architect agent
3. Architect agent evaluates impact on other modules
4. All affected module agents notified
5. Each module agent confirms their contracts still hold
6. Change approved and propagated

**Example:**

```
Scenario: Payment module wants to add "fee" field to Transaction
  Original: Transaction = {id: UUID, amount: Money, status: PaymentStatus}
  Proposed: Transaction = {id: UUID, amount: Money, fee: Money, status: PaymentStatus}

Order module checks: Does my precondition still work?
  My precondition expected: {id, amount, status}
  New version provides: {id, amount, fee, status}
  Impact: ✓ Still works (fee is extra field, not required)

Inventory module checks: Don't use Transaction, no impact ✓

Decision: Change approved ✓
```

---

## When Human Intervention is Needed

### Red Flags → Escalate to Human

| Situation | Escalation | Resolution |
|-----------|------------|-----------|
| Spec is ambiguous | Architect + tech lead | Architect agent clarifies |
| Module boundary is wrong | Tech lead | Restart Phase 1 decomposition |
| Module design won't work | Tech lead + architect | Redesign with Phase 2 restart |
| Code doesn't match spec | Tech lead + code reviewer | Module agent fixes or escalates |
| Contract verification fails | Integration agent + tech lead | Identify which module has bug |
| Cross-module communication is unclear | All involved agents + tech lead | Clarify contract, potentially adjust specs |

### Decision Framework

```
Agent proposes X

Human asks: "Is X correct according to spec/design/intent?"
  └─ YES → Approve, agent proceeds
  └─ NO → Explain why, agent revises
  └─ UNCLEAR → Escalate to architect for clarification
```

---

## Practical Tips for Orchestrating Today

### 1. Start Small (3-module system)

Don't try a 20-module system first. Use the e-commerce example:
- Order (depends on inventory, payment)
- Inventory (no dependencies)
- Payment (no dependencies)

This covers all the patterns without being overwhelming.

### 2. Use Claude Projects for Persistence

```
One Claude Project per phase + agent combination:

Projects:
├── Phase1-SystemArchitecture
│   └── Long conversation with architect agent
│
├── Phase2-OrderModuleDesign
│   └── Order agent works through design
│
├── Phase2-InventoryModuleDesign
│   └── Inventory agent works through design
│
├── Phase3-OrderModuleImplementation
│   └── Order agent generates code
│
├── Phase4-Integration
│   └── Integration agent verifies contracts
```

**Why?** Claude Projects maintain context across 20+ turns, letting agents iterate without re-explaining the spec each time.

### 3. Create a Specification Repository

```
specs/
├── .vdm/
│   ├── system-spec.vdmsl           # Source of truth
│   ├── shared-types.vdmsl
│   ├── order-module.vdmsl
│   ├── inventory-module.vdmsl
│   └── payment-module.vdmsl
│
├── interface-specs.md              # Human-readable version of contracts
│
└── CHANGES.md                       # Log of spec changes + approvals
```

Keep these in version control (Git). Reference them in every agent prompt.

### 4. Create a Handoff Checklist

**Phase 1 → 2 Handoff:**
```
□ System spec is syntactically valid VDM-SL
□ Module boundaries are clear
□ Each module's public operations are specified
□ Dependency graph is acyclic (no circular deps)
□ Shared types are defined
□ Architect agent confirms: "This is correct"
□ Tech lead confirms: "This matches requirements"
→ PROCEED TO PHASE 2
```

**Phase 2 → 3 Handoff:**
```
□ Design document explains module structure
□ How each operation maps to internal functions
□ How dependencies are called
□ Error handling strategy
□ Tech lead reviews design
→ PROCEED TO PHASE 3
```

**Phase 3 → 4 Handoff:**
```
□ Implementation code matches design
□ Unit tests cover preconditions
□ Code reviewed for quality
□ Tests pass locally
□ Module agent confirms: "Code satisfies spec"
→ PROCEED TO PHASE 4
```

**Phase 4 Go/No-Go:**
```
□ All contracts verified (A.post ⇒ B.pre)
□ Integration tests pass
□ No unresolved contract violations
□ Tech lead reviews integration report
→ DECISION: GO or NO-GO
```

### 5. Maintain a "Agent Context Document"

Each agent needs this document. Include:

```markdown
# Order Module Agent Context

## Your Module Spec
[full VDM-SL of order module]

## Dependency Interface Specs
[postconditions only of inventory and payment]

## Shared Types
[Customer, Product, Money, etc.]

## Error Handling Contract
[how to return errors, Result[T] pattern]

## Current Phase
Phase 3: Implementation

## What You're Doing
Generate Python code for order module

## Success Criteria
- Code satisfies pre/post conditions
- Unit tests cover preconditions
- Integration with inventory/payment contracts verified
```

When you message the agent, paste this + your task. The agent can then work autonomously without losing context.

### 6. Use CLI Tools for Contract Verification

Manual checklist:
```bash
# For each inter-module call:
# 1. Extract caller's postcondition
# 2. Extract callee's precondition
# 3. Check: postcond ⇒ precond?

# Example:
Order.finalize_order() postcondition:
  order.status = "finalized"
  payment_result.status = "success"

Payment.charge() precondition:
  amount > 0
  user_id != null

Question: Does Order's postcond give Payment what it needs?
  Payment needs: amount > 0, user_id != null
  Order provides: order.status, payment_result.status
  Problem: amount and user_id not in postcond! ⚠️

Solution: Adjust Order.charge call or Payment precondition
```

### 7. Document Agent Prompts as You Refine Them

As you run phases, you'll improve the phase-specific prompts. Document what works:

```markdown
# What Works Well in Phase 2 (Design)

## Effective Prompt Elements
- Explicitly show the module spec
- Show dependency interface specs (postcond only)
- Ask agent to draw a data flow diagram
- Ask agent to list operation → internal function mapping
- Ask: "What could go wrong at each step?"

## Common Agent Mistakes
- Designing without reading dependency contracts
- Overcomplicating error handling
- Not checking for circular dependencies

## Fixes
- Always start Phase 2 prompt with "Here's your spec: ..."
- Ask agent to quote preconditions before designing
- Ask agent to verify no circular deps
```

Store this alongside the phase prompts.

---

## Example Workflow: E-Commerce Order System (Real)

### Timeline

**Day 1-2: Phase 1 (Architect)**
```
Start: 9 AM
- Architect agent dialogues with tech lead
- Elicits order requirements, inventory model, payment flow
- Proposes types: Order, Product, Customer, Payment
- Tech lead reviews and approves
- Architect produces: system-spec.vdmsl

Handoff: specs/ directory populated
→ READY FOR PHASE 2
```

**Day 3-4: Phase 2 (Design, Parallel)**
```
Start: 9 AM (3 projects running in parallel)
- Order module agent: Designs order creation, state machine
- Inventory agent: Designs stock tracking, deduction
- Payment agent: Designs charging, refunds

Each agent:
  Receives: own spec + dependency postconds
  Produces: DESIGN.md
  Time: ~4 hours each
  Overlap: All 3 can run simultaneously

Tech lead reviews designs (1 hour each)
→ READY FOR PHASE 3
```

**Day 5-7: Phase 3 (Implementation, Parallel)**
```
Start: 9 AM (3 projects running in parallel)
- Order agent: Writes Python, generates tests
- Inventory agent: Writes Python, generates tests
- Payment agent: Writes Python, generates tests

Each agent:
  Receives: design + spec
  Produces: src/, tests/
  Time: ~16 hours per module
  Overlap: All 3 run simultaneously

Code review (2 hours per module)
→ READY FOR PHASE 4
```

**Day 8-9: Phase 4 (Integration)**
```
Start: 9 AM
- Integration agent: Receives all 3 modules
- Verifies contracts:
  order.finalize_order.post ⇒ inventory.deduct.pre? ✓
  order.finalize_order.post ⇒ payment.charge.pre? ✓
  All contracts verified!

- Generates integration tests (end-to-end scenarios)
- Runs tests against integrated system
- All tests pass!

Tech lead reviews integration report
Decision: GO

Deployment plan created
→ READY FOR DEPLOYMENT
```

**Total Timeline: 9 days for 3-module system**

Compare to traditional:
- Single engineer: 4-6 weeks (serial: design → code → test → integrate)
- Three engineers: 2-3 weeks (parallel but with integration friction)
- Three agents + 1 tech lead: 9 days (parallel + formal contracts eliminate friction)

---

## Troubleshooting Common Issues

### Issue 1: "Module agents designed things that don't work together"

**Root Cause:** Module agents didn't understand dependency contracts clearly.

**Fix:**
1. In Phase 2, have each agent explicitly quote dependency postconditions
2. Ask agent: "Given these postconditions, how will you call this module?"
3. Have architect agent review Phase 2 designs before Phase 3 starts
4. Add a "contract walkthrough" before Phase 3

### Issue 2: "Contracts verified but integration tests fail"

**Root Cause:** Formal specs don't capture all constraints (e.g., timing, performance, failure modes).

**Fix:**
1. Add constraints to specs: "payment.charge must complete within 2 seconds"
2. Add error handling specs: "If payment fails, order must rollback to pending"
3. Have integration agent generate negative tests too
4. Loop back: Update specs → Phase 3 to implement updated contracts

### Issue 3: "Shared type changes broke all modules"

**Root Cause:** No change management process.

**Fix:**
1. Freeze shared types after Phase 1 (or have clear change process)
2. If change needed: architect evaluates impact on all contracts
3. All affected module agents re-verify contracts
4. Only then propagate change
5. Document in CHANGES.md

### Issue 4: "Module agents keep inventing their own error types"

**Root Cause:** No shared error handling contract.

**Fix:**
1. In Phase 1, architect agent must define common error types (Result[T], ApiError, etc.)
2. Add to shared-types.vdmsl
3. In Phase 2, explicitly show module agents the error contract
4. In Phase 3, ask agent to use shared error types

### Issue 5: "Takes too long for agents to generate code"

**Root Cause:** Agents lack context or are being too verbose.

**Fix:**
1. Use Claude Projects to maintain context (not one-shot messages)
2. Keep agent context document tight and focused
3. Ask agents for conciseness: "Generate working code, minimal comments"
4. Provide examples from similar modules
5. Break Phase 3 into smaller chunks (operation by operation)

---

## Integration with Existing Tools

### Using Claude Projects (Recommended for Teams)

```
Project: Order System Formal Spec Driven Development

Instructions (add to project):
[Templates/prompts/phase1-specification.md]

Files (upload):
specs/.vdm/system-spec.vdmsl (once created)
specs/.vdm/shared-types.vdmsl
[Module specs as created]

Create sub-conversations for each phase:
├── Phase1-ArchitectureDialogue
│   ├ Follow system spec prompt
│   ├ Dialogue to refine specs
│   └→ Output: system-spec.vdmsl
│
├── Phase2-OrderDesign
│   ├ Show order module spec
│   ├ Show dependency postconds
│   └→ Output: order DESIGN.md
│
├── Phase3-OrderImplementation
│   ├ Show design doc
│   ├ Show module spec
│   └→ Output: src/, tests/
│
└── Phase4-Integration
    ├ Show all modules + specs
    ├ Show phase4-verification prompt
    └→ Output: integration-report.md
```

### Using Custom Orchestration Script (Advanced)

If you want to automate handoffs:

```python
#!/usr/bin/env python3
import anthropic

class MultiAgentOrchestrator:
    def __init__(self):
        self.client = anthropic.Anthropic()

    def phase1_architect(self, requirements_doc):
        """Run Phase 1: System spec"""
        return self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=8000,
            system=self.load_prompt("phase1-specification"),
            messages=[{
                "role": "user",
                "content": f"Here are our requirements:\n{requirements_doc}\nLet's build a formal spec..."
            }]
        )

    def phase2_module_design(self, module_spec, dependency_specs):
        """Run Phase 2: Module design"""
        context = f"Module Spec:\n{module_spec}\n\nDependency Specs:\n{dependency_specs}"
        return self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=4000,
            system=self.load_prompt("phase2-design"),
            messages=[{
                "role": "user",
                "content": f"{context}\n\nDesign this module..."
            }]
        )

    def phase3_implementation(self, design_doc, module_spec):
        """Run Phase 3: Implementation"""
        context = f"Design:\n{design_doc}\n\nSpec:\n{module_spec}"
        return self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=8000,
            system=self.load_prompt("phase3-implementation"),
            messages=[{
                "role": "user",
                "content": f"{context}\n\nImplement this module in Python..."
            }]
        )

    def phase4_integration(self, all_modules, all_specs):
        """Run Phase 4: Integration verification"""
        context = f"Modules:\n{all_modules}\n\nSpecs:\n{all_specs}"
        return self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=4000,
            system=self.load_prompt("phase4-verification"),
            messages=[{
                "role": "user",
                "content": f"{context}\n\nVerify contracts and generate integration tests..."
            }]
        )

    def load_prompt(self, phase):
        with open(f"templates/prompts/{phase}.md") as f:
            return f.read()

# Usage:
orchestrator = MultiAgentOrchestrator()

# Phase 1
system_spec = orchestrator.phase1_architect(requirements)

# Phase 2 (parallel for each module)
order_design = orchestrator.phase2_module_design(order_spec, dependency_specs)
inventory_design = orchestrator.phase2_module_design(inventory_spec, {})
payment_design = orchestrator.phase2_module_design(payment_spec, {})

# Phase 3 (parallel for each module)
order_code = orchestrator.phase3_implementation(order_design, order_spec)
# ...

# Phase 4
integration_report = orchestrator.phase4_integration(all_modules, all_specs)
```

This enables fully automated orchestration while maintaining human checkpoints.

---

# 日本語セクション

## 概要

本ガイドでは、形式仕様駆動開発のワークフローにおいて、複数のAIエージェントをオーケストレーションする方法を説明します。

**核となる考え方：エージェント間の通信は「コード」ではなく「形式仕様（VDM-SL）」を通じて行う**

### なぜマルチエージェント・オーケストレーションなのか？

従来の単一エージェント開発の限界：
- 1つのエージェントでは、システム全体をコンテキストに入れられない
- 「仕様→設計→実装→テスト」の手渡しが不確実
- コンポーネント間に「契約」がないため、互いに独立して開発できない

**マルチエージェント・アプローチ：**
- **アーキテクト・エージェント**（フェーズ1）：システム仕様とモジュール分解を定義
- **モジュール・エージェント**（フェーズ2-4）：以下のみを必要として並行開発
  - 自分のモジュール仕様
  - 依存モジュールのインターフェース仕様（事後条件 → 事前条件）
- **統合エージェント**（フェーズ4）：モジュール間の契約を検証

### 核となる原則：形式仕様を「契約」として機能させる

```
モジュールA（注文）         モジュールB（決済）
┌──────────────────┐       ┌──────────────────┐
│ finalize_order   │       │ charge           │
│ 事後条件:        │       │ 事前条件:        │
│ order.status=    │ ───→  │ payment > 0  ✓   │
│ "finalized"      │       │ payment.method   │
│                  │       │ != null      ✓   │
└──────────────────┘       └──────────────────┘

AのpostcondがBのprecondを満たせば合成可能！
```

これが「契約検証」ステップです。すべてのモジュールが仕様を満たし、契約が検証できれば、システムは正しく動作します。

---

## フェーズ概要

### フェーズ1：システム・アーキテクチャ・仕様（人間 + アーキテクト・エージェント）

**期間：** 1-3日
**キープレイヤー：** 技術リード、ドメイン専門家、アーキテクト・エージェント
**成果物：** 完全なVDM-SL形式仕様 + モジュール分解

**何が起こるか：**

1. **アーキテクト・エージェント**が人間のステークホルダーと対話
2. **ビジネス要件**とドメイン制約をヒアリング
3. **型と不変条件**を提案（ビジネスルールをVDM-SLでエンコード）
4. **操作**を定義（事前条件・事後条件付き）
5. **モジュール分解**を実行（明確な境界を持つモジュールに）
6. **形式仕様**を完成（フェーズ2-4を導く）

**主要な成果物：**

```
specs/
├── .vdm/
│   ├── system-spec.vdmsl         # すべてを含む
│   ├── order-module.vdmsl        # 注文モジュール用抽出
│   ├── inventory-module.vdmsl    # 在庫モジュール用抽出
│   ├── payment-module.vdmsl      # 決済モジュール用抽出
│   └── shared-types.vdmsl        # モジュール間で共有される型
│
└── module-decomposition.md       # 各モジュールが何を所有するか
```

**人間のチェックポイント：**

- ✅ ドメイン型の提案後：「この型でビジネスルールをキャプチャしていますか？」
- ✅ モジュール境界定義後：「各モジュールの責任は正しいですか？」
- ✅ 契約定義後：「事前/事後条件は正しいですか？」
- ✅ 最終承認：フェーズ2に進む前に仕様に署名

**実装方法（今すぐやる）：**

**Claude Projects**を使用：

1. Claudeプロジェクトを作成：「注文システム形式仕様」
2. テンプレートをアップロード：`templates/prompts/phase1-specification.md`
3. テンプレートを含める：`templates/vdm-sl/module-template.vdmsl`
4. アーキテクト・エージェントとの対話開始：
   ```
   あなたは形式仕様の設計者です。
   [phase1-specification.md のコンテンツを挿入]

   注文処理システムを定義しましょう...
   ```
5. エージェントがVDM-SLを生成するにつれて反復
6. ステークホルダーが承認したら、仕様を`specs/.vdm/`にエクスポート

---

### フェーズ2：モジュール設計（モジュール別エージェント、並行実行）

**期間：** モジュール当たり1-2日
**キープレイヤー：** モジュール別エージェント
**成果物：** 詳細設計 + 各モジュールの操作分解

**何が起こるか：**

各モジュール・エージェントが独立してモジュール設計：

1. **受け取る：** 自分のモジュール仕様 + 依存モジュールのインターフェース仕様
2. **考える：** 「このモジュール内部をどう構造化すべき？」
3. **成果：** 操作分解を含む設計ドキュメント
4. **制約：**
   - 自分のモジュール仕様を尊重（事前/事後条件）
   - 依存契約を違反しない
   - 合成可能性を確保

**例：注文モジュール・エージェント**

```
入力：
- 自分の仕様：order-module.vdmsl（create_order, finalize_order など）
- 依存仕様：
  - inventory.check_stock(product_id) → {available: nat}
  - payment.charge(amount) → {transaction_id: UUID, status: PaymentStatus}
- 共有型：Customer, Product, Money

出力：
- 設計ドキュメント：
  1. 注文のステートマシンをどう構造化するか？
  2. どの操作がいつ inventory.check_stock を呼ぶか？
  3. どの操作がいつ payment.charge を呼ぶか？
  4. 失敗時（在庫なし、決済失敗）をどう処理するか？
```

**主要な成果物：**

```
modules/order/
├── DESIGN.md                  # 設計判断、操作フロー
├── preconditions.txt          # 各操作の事前条件
├── postconditions.txt         # 各操作の事後条件
└── dependencies.txt           # どの操作がどう外部モジュールを呼ぶか
```

**人間のチェックポイント：**

- ✅ 設計レビュー：「この設計アプローチは合理的ですか？」
- ✅ 依存チェック：「契約依存は明確ですか？」

**実装方法（今すぐやる）：**

各モジュール用に新しいClaudeプロジェクトまたは会話を作成：

```yaml
注文モジュール用：
  プロンプト：「あなたはモジュール設計の専門家です」
  コンテキスト：
    - 注文モジュール仕様（フェーズ1から）
    - 在庫インターフェース仕様（事後条件のみ）
    - 決済インターフェース仕様（事後条件のみ）
    - フェーズ2設計プロンプト（templates/prompts/phase2-design.md）

  タスク：「仕様を実装しながら在庫・決済の契約を尊重するような
           注文モジュールの内部構造を設計してください」
```

各モジュール・エージェント：
1. フェーズ2設計プロンプトを開く
2. 自分の仕様 + 依存インターフェース仕様を受け取る
3. 設計ドキュメントを生成
4. フェーズ3に進む前に人間の承認を待つ

**並行実行可能：** 3つのモジュール・エージェントが同時に動作（それらの間に依存関係なし）

---

### フェーズ3：実装（モジュール別エージェント、並行実行）

**期間：** モジュール当たり2-5日
**キープレイヤー：** モジュール別エージェント
**成果物：** 各モジュールのソースコード + ユニットテスト

**何が起こるか：**

各モジュール・エージェントが設計を実装：

1. **受け取る：** 設計ドキュメント + 自分のモジュール仕様 + 共有型
2. **生成：** ターゲット言語（Python、TypeScript など）の実装コード
3. **自動生成：** 事前条件に基づくユニットテスト
4. **検証：** 「自分のコードが仕様を満たすか？」
5. **成果：** レビューとテスト可能なソースコード

**例：注文モジュール実装**

```python
# フェーズ3の出力例：

def create_order(customer: Customer, items: List[OrderItem]) -> Order:
    """
    事前条件（仕様から）：
    - customer.id は有効（システムに存在）
    - items は空でない
    - 各 item.quantity は正数

    事後条件（仕様から）：
    - status="pending" の Order を返す
    - order.total = 品目価格の合計
    - order がシステムに記録される
    """
    # 実装はここ...

def finalize_order(order: Order) -> Order:
    """
    事前条件（仕様から）：
    - order.status == "pending"
    - order.total > 0

    事後条件（仕様から）：
    - order.status == "finalized"
    - payment.charge(order.total) を呼ぶ
    - inventory.deduct(order.items) を呼ぶ
    - 更新された order を返す
    """
    # 事前条件をチェック
    assert order.status == "pending"
    assert order.total > 0

    # 依存を呼ぶ（形式契約経由）
    payment_result = payment.charge(order.total)
    if payment_result.status != "success":
        # 決済に失敗した場合何をするか？
        # 仕様が言うべき...
        raise PaymentFailedError(...)

    inventory.deduct(order.items)

    # 状態を更新
    order.status = "finalized"
    return order
```

**主要な成果物：**

```
modules/order/
├── src/
│   ├── order.py               # 実装
│   ├── types.py               # ローカル型
│   └── errors.py              # エラー定義
│
└── tests/
    ├── test_create_order.py    # 事前条件から自動生成
    ├── test_finalize_order.py
    └── test_error_cases.py
```

**人間のチェックポイント：**

- ✅ コードレビュー：「コードは設計と仕様に合致していますか？」
- ✅ テスト検証：「テストは事前条件と整合していますか？」
- ✅ 統合準備：「このモジュールは統合できる状態ですか？」

**実装方法（今すぐやる）：**

```yaml
注文モジュール・フェーズ3：
  新しい会話またはプロジェクト継続
  プロンプト：「あなたはモジュール開発の専門家です」
  コンテキスト：
    - 注文設計ドキュメント（フェーズ2から）
    - 注文モジュール仕様（フェーズ1から）
    - 共有型とエラーパターン
    - フェーズ3実装プロンプト（templates/prompts/phase3-implementation.md）
    - プロジェクトからの実装例

  タスク：「PythonまたはTypeScriptで注文モジュールを実装してください。
           事前条件に基づくユニットテストを生成してください。
           コードが仕様を満たすことを検証してください。」
```

各モジュール・エージェント：
1. 設計ドキュメント + 仕様を取得
2. 実装を生成
3. テストを生成
4. コードレビューに提出

**並行実行可能：** 3つのモジュール・エージェントが同時に実装

**成功の鍵：**

- 各エージェントは自分が呼ぶモジュールのみを知る
- 他のモジュールの内部実装を知らない
- テストは事前条件から導出（発明ではない）→ 体系的なカバレッジ
- コードは外部モジュール呼び出し前に明示的に事前条件をチェック

---

### フェーズ4：統合・検証（すべてのモジュール + 統合エージェント）

**期間：** 1-2日
**キープレイヤー：** 統合エージェント、すべてのモジュール・エージェント、技術リード
**成果物：** 検証済み統合システム + デプロイメント・チェックリスト

**何が起こるか：**

統合エージェントが重要な検証ステップを実行：

```
システム内の、モジュール間呼び出しすべてについて：
  A.operation_postcondition ⇒ B.operation_precondition？

すべての含意が真 → システムは正しい！✓
いずれかが偽 → どの契約が失敗したか特定して修正
```

**例：合成可能性の検証**

```
注文モジュールが呼ぶ：inventory.check_stock(product_id)
  在庫の事後条件：{available: nat, reserved: nat} を返す
  注文の事前条件：{available: nat, reserved: nat} を期待
  検証：✓ 事後条件は事前条件が必要なすべてを含む

注文モジュールが呼ぶ：payment.charge(amount)
  決済の事後条件：{transaction_id: UUID, status: PaymentStatus} を返す
  注文の事前条件：{transaction_id: UUID, status: PaymentStatus} を期待
  検証：✓ 事後条件が事前条件に合致

すべての契約が検証済み → 統合可能！
```

**統合エージェントの責任：**

1. **成果物を収集：**
   - すべてのモジュール実装
   - すべてのユニットテスト
   - すべての仕様
   - 統合テスト仕様（システムレベルのシナリオ）

2. **契約を検証：**
   - モジュール間呼び出しごと：A.post ⇒ B.pre？
   - どのモジュールが合成されるか、されないか記録
   - 契約違反のフラグ

3. **統合テストを生成：**
   - システム仕様に基づくエンドツーエンドシナリオ
   - 正常系（すべての契約が満たされる）
   - 異常系（モジュールが失敗した場合）

4. **統合レポートを作成：**
   - 合成可能性マトリックス（どのペアが動作）
   - テスト結果
   - GO/NO-GO推奨

**主要な成果物：**

```
integration/
├── composability-report.md     # どの契約が検証されたか
├── integration-tests.py        # システムレベルテスト
├── test-results.txt            # 合格/不合格
└── deployment-checklist.md     # GO/NO-GO決定
```

**実装方法（今すぐやる）：**

```yaml
フェーズ4ワークフロー：
  1. アーキテクト・エージェント：システムレベルテスト仕様を抽出
  2. 統合エージェント：すべてのモジュール + 仕様を受け取る

  フェーズ4プロンプト：
    役割：「統合検証スペシャリスト」
    コンテキスト：
      - すべてのモジュール実装
      - すべてのモジュール仕様
      - システム仕様（フェーズ1から）
      - フェーズ4検証プロンプト（templates/prompts/phase4-verification.md）

    タスク：
      1. 契約検証：すべての呼び出しについて A.post ⇒ B.pre？
      2. 統合テストをシステム仕様から生成
      3. テストが合格することを検証
      4. GO/NO-GO レポートを作成
```

**人間のレビュー・ステップ：**

1. 統合エージェントが検証済み契約をフラグ
2. すべてが検証済み + テスト合格 → 統合エージェントがGOを推奨
3. いずれかが失敗 → どのモジュールが修正必要かを特定、フェーズ3に戻る
4. 技術リードがレポートをレビューしてデプロイを決定

---

## 横断的関心事：共有パターンの処理

いくつかのパターンは複数のモジュールにまたがります（エラー処理、ロギング、認可など）。これらはフェーズ1で定義し、すべてのエージェントと共有します。

### 例：エラー処理契約

**フェーズ1：アーキテクト・エージェントが定義**

```vdmsl
-- shared-types.vdmsl 内

types

  ErrorCode = <VALIDATION_ERROR> | <NOT_FOUND> | <CONFLICT> | <INTERNAL_ERROR> | <TIMEOUT>

  ApiError = {
    code: ErrorCode,
    message: seq of char,
    details: map seq of char to seq of char
  }

  Result[T] = Success(T) | Failure(ApiError)

end
```

**フェーズ2-4：モジュール・エージェントがこのパターンを使用**

```python
# すべてのモジュール実装が Result[T] を使用

def create_order(customer: Customer, items: List[OrderItem]) -> Result[Order]:
    # Success(order) または Failure(error) を返す
    ...

def charge(amount: Money) -> Result[Transaction]:
    # Success(transaction) または Failure(error) を返す
    ...
```

**利点：** すべてのモジュールが同じ方法でエラーを処理。統合エージェントがエラー処理契約を検証可能。

### 例：ロギング契約

**フェーズ1：アーキテクト・エージェントが定義**

```vdmsl
types

  LogLevel = <DEBUG> | <INFO> | <WARN> | <ERROR>

  LogEntry = {
    timestamp: Time,
    level: LogLevel,
    operation: seq of char,
    user_id: UUID,
    message: seq of char
  }

end
```

**フェーズ2-4：モジュール・エージェントがこの形式でログを出力**

**利点：** ロギングはすべてのモジュールで構造化。統合テスト中のイベント相関が容易。

### 例：認可契約

**フェーズ1：アーキテクト・エージェントが定義**

```vdmsl
types

  User = {
    id: UUID,
    email: seq of char,
    roles: set of Role
  }

  AuthContext = {
    user: User,
    token: seq of char,
    exp_time: Time
  }

end

operations

  -- 共有関数：操作呼び出し前に認可を検証
  check_auth(user: User) -> bool
    post RESULT = (user.id <> nil and user.email <> nil)

end
```

**フェーズ2-4：モジュール・エージェントがリクエスト処理前に認可をチェック**

```python
def create_order(auth_context: AuthContext, customer: Customer, items: List[OrderItem]) -> Result[Order]:
    # 事前条件：認可は有効
    assert auth_context.user is not None
    assert auth_context.token is not None
    assert auth_context.exp_time > now()

    # ビジネスロジックに進む...
    ...
```

**利点：** すべてのモジュールが同じ方法で認可を実装。統合エージェントが認可契約を検証可能。

---

## 共有データモデル管理

共有型は `specs/.vdm/shared-types.vdmsl` に存在します。変更は注意深く管理する必要があります。

**変更プロセス：**

1. モジュール・エージェントが型変更の必要性を特定
2. アーキテクト・エージェントに変更を提案
3. アーキテクト・エージェントが他のモジュールへの影響を評価
4. 影響を受けるすべてのモジュール・エージェントに通知
5. 各モジュール・エージェントが自分の契約がまだ有効かを確認
6. 変更が承認され、伝播される

**例：**

```
シナリオ：決済モジュールが Transaction に「fee」フィールドを追加したい
  元の定義：Transaction = {id: UUID, amount: Money, status: PaymentStatus}
  提案版：Transaction = {id: UUID, amount: Money, fee: Money, status: PaymentStatus}

注文モジュール確認：事前条件はまだ動作するか？
  期待していた：{id, amount, status}
  新版が提供：{id, amount, fee, status}
  影響：✓ まだ動作（fee は余分フィールド、不要）

在庫モジュール確認：Transaction を使用しない、影響なし ✓

決定：変更が承認される ✓
```

---

## 人間介入が必要なときを知る

### 赤旗 → 人間にエスカレーション

| 状況 | エスカレーション | 解決 |
|------|------------|------|
| 仕様が曖昧 | アーキテクト + 技術リード | アーキテクト・エージェントが明確化 |
| モジュール境界が間違い | 技術リード | フェーズ1分解をやり直し |
| モジュール設計が動作不可 | 技術リード + アーキテクト | フェーズ2をやり直して設計 |
| コードが仕様に合致しない | 技術リード + コード審査者 | モジュール・エージェントが修正またはエスカレーション |
| 契約検証が失敗 | 統合エージェント + 技術リード | どのモジュールがバグを持つか特定 |
| モジュール間通信が不明 | すべての関連エージェント + 技術リード | 契約を明確化、仕様調整の可能性 |

### 決定フレームワーク

```
エージェントが X を提案

人間：「X は仕様/設計/意図に従って正しいですか？」
  └─ はい → 承認、エージェントが進む
  └─ いいえ → 理由を説明、エージェントが改訂
  └─ 不明確 → アーキテクトにエスカレーション
```

---

## 実践的なティップス（今すぐ実行）

### 1. 小さく始める（3モジュール・システム）

20モジュール・システムから始めないでください。eコマース例を使用：
- Order（inventory、payment に依存）
- Inventory（依存なし）
- Payment（依存なし）

圧倒されることなく、すべてのパターンをカバーします。

### 2. 永続性にはClaudeプロジェクトを使用

```
プロジェクト：

├── フェーズ1-システムアーキテクチャ
│   └── アーキテクト・エージェントとの長い会話
│
├── フェーズ2-注文モジュール設計
│   └── 注文エージェントが設計を進める
│
├── フェーズ2-在庫モジュール設計
│   └── 在庫エージェントが設計を進める
│
├── フェーズ3-注文モジュール実装
│   └── 注文エージェントがコードを生成
│
└── フェーズ4-統合
    └── 統合エージェントが契約を検証
```

**なぜ？** Claude Projects は 20 ターン以上にわたってコンテキストを保ち、エージェントが毎回仕様を再説明することなく反復できます。

### 3. 仕様リポジトリを作成

```
specs/
├── .vdm/
│   ├── system-spec.vdmsl           # 信頼の源
│   ├── shared-types.vdmsl
│   ├── order-module.vdmsl
│   ├── inventory-module.vdmsl
│   └── payment-module.vdmsl
│
├── interface-specs.md              # 契約の人間が読める版
│
└── CHANGES.md                       # 仕様変更 + 承認のログ
```

これらをバージョン管理（Git）に保ち、すべてのエージェント・プロンプトで参照します。

### 4. ハンドオフ・チェックリストを作成

**フェーズ1 → 2 ハンドオフ：**
```
□ システム仕様は構文的に有効なVDM-SL
□ モジュール境界は明確
□ 各モジュールの公開操作は仕様化される
□ 依存グラフは非循環（循環依存なし）
□ 共有型が定義される
□ アーキテクト・エージェント確認：「これは正しい」
□ 技術リード確認：「これは要件に合致する」
→ フェーズ2に進む
```

**フェーズ2 → 3 ハンドオフ：**
```
□ 設計ドキュメントがモジュール構造を説明
□ 各操作が内部関数にどのようにマップされるか
□ 依存がどのように呼ばれるか
□ エラー処理戦略
□ 技術リードが設計をレビュー
→ フェーズ3に進む
```

**フェーズ3 → 4 ハンドオフ：**
```
□ 実装コードは設計と合致
□ ユニットテストは事前条件をカバー
□ コードの品質をレビュー
□ テストはローカルで合格
□ モジュール・エージェント確認：「コードが仕様を満たす」
→ フェーズ4に進む
```

**フェーズ4 GO/NO-GO：**
```
□ すべての契約が検証される（A.post ⇒ B.pre）
□ 統合テストが合格
□ 未解決の契約違反なし
□ 技術リードが統合レポートをレビュー
→ 決定：GOまたはNO-GO
```

### 5. エージェント・コンテキスト・ドキュメントを保守

各エージェントがこのドキュメントを必要とします。含める：

```markdown
# 注文モジュール・エージェント・コンテキスト

## あなたのモジュール仕様
[注文モジュールのVDM-SL全文]

## 依存インターフェース仕様
[在庫と決済の事後条件のみ]

## 共有型
[Customer, Product, Money など]

## エラー処理契約
[エラーをどう返すか、Result[T] パターン]

## 現在のフェーズ
フェーズ3：実装

## あなたがやること
注文モジュール用の Python コードを生成

## 成功基準
- コードが事前/事後条件を満たす
- ユニットテストが事前条件をカバー
- 在庫・決済との統合契約が検証される
```

エージェントにメッセージするとき、このドキュメント + タスクを貼り付けます。エージェントはコンテキストを失わずに自律的に動作できます。

### 6. クライアント・ツールを使用して契約を検証

手動チェックリスト：
```bash
# モジュール間呼び出しごと：
# 1. 呼び出し元の事後条件を抽出
# 2. 被呼び出し元の事前条件を抽出
# 3. チェック：事後条件 ⇒ 事前条件？

# 例：
Order.finalize_order() 事後条件：
  order.status = "finalized"
  payment_result.status = "success"

Payment.charge() 事前条件：
  amount > 0
  user_id != null

問題：Order の事後条件に amount と user_id がない！⚠️

解決：Order.charge 呼び出しを調整するか Payment 事前条件を調整
```

### 7. エージェント・プロンプトをドキュメント化して改善

フェーズを進めるにつれ、フェーズ固有のプロンプトを改善します。何が機能するかをドキュメント化：

```markdown
# フェーズ2（設計）で有効な要素

## 効果的なプロンプト要素
- 明示的にモジュール仕様を表示
- 依存インターフェース仕様を表示（事後条件のみ）
- エージェントにデータフロー図を描かせる
- 操作 → 内部関数マッピングのリストを作らせる
- 各ステップで「何が悪くなる可能性があるか？」と質問

## エージェント・ミス
- 依存契約を読まずに設計
- エラー処理を過度に複雑化
- 循環依存をチェックしない

## 修正
- フェーズ2プロンプトを常に「ここがあなたの仕様：...」で始める
- 事前条件を引用させてから設計させる
- 循環依存がないことを検証させる
```

これをフェーズ・プロンプトと一緒に保存します。

---

## 例：実eコマース注文システム

### タイムライン

**1-2日目：フェーズ1（アーキテクト）**
```
開始：朝9時
- アーキテクト・エージェントが技術リードと対話
- 注文要件、在庫モデル、決済フローをヒアリング
- 型を提案：Order, Product, Customer, Payment
- 技術リードがレビューして承認
- アーキテクトが完成：system-spec.vdmsl

ハンドオフ：specs/ ディレクトリを配置
→ フェーズ2の準備完了
```

**3-4日目：フェーズ2（設計、並行実行）**
```
開始：朝9時（3つのプロジェクトが同時に実行）
- 注文モジュール・エージェント：注文作成、ステートマシンを設計
- 在庫エージェント：ストック追跡、 deduction を設計
- 決済エージェント：チャージ、払い戻しを設計

各エージェント：
  受け取る：自分の仕様 + 依存事後条件
  生成：DESIGN.md
  時間：各 ~4時間
  重複：3つとも同時に実行可能

技術リードが設計をレビュー（各 1時間）
→ フェーズ3の準備完了
```

**5-7日目：フェーズ3（実装、並行実行）**
```
開始：朝9時（3つのプロジェクトが同時に実行）
- 注文エージェント：Python を書く、テストを生成
- 在庫エージェント：Python を書く、テストを生成
- 決済エージェント：Python を書く、テストを生成

各エージェント：
  受け取る：設計 + 仕様
  生成：src/, tests/
  時間：モジュール当たり ~16時間
  重複：3つとも同時に実行可能

コード・レビュー（モジュール当たり 2時間）
→ フェーズ4の準備完了
```

**8-9日目：フェーズ4（統合）**
```
開始：朝9時
- 統合エージェント：3つのモジュールをすべて受け取る
- 契約を検証：
  order.finalize_order.post ⇒ inventory.deduct.pre? ✓
  order.finalize_order.post ⇒ payment.charge.pre? ✓
  すべての契約が検証される！

- 統合テストを生成（エンドツーエンドのシナリオ）
- 統合システムに対してテストを実行
- すべてのテストが合格！

技術リードが統合レポートをレビュー
決定：GO

デプロイメント・プランを作成
→ デプロイメント準備完了
```

**総タイムライン：3モジュール・システムで 9日間**

従来との比較：
- 単一エンジニア：4-6週間（直列：設計→コード→テスト→統合）
- 3人のエンジニア：2-3週間（並行だが統合の摩擦あり）
- 3エージェント + 1技術リード：9日間（並行 + 形式契約で摩擦排除）

---

## よくあるトラブルシューティング

### 問題1：「モジュール・エージェントが互いに動作しないもの を設計した」

**根本原因：** モジュール・エージェントが依存契約をはっきり理解していない。

**修正：**
1. フェーズ2で、各エージェントに依存の事後条件を明示的に引用させる
2. エージェントに質問：「これらの事後条件が与えられて、このモジュールをどう呼ぶ？」
3. フェーズ3開始前に、アーキテクト・エージェントにフェーズ2設計をレビューさせる
4. フェーズ3前に「契約ウォークスルー」を追加

### 問題2：「契約が検証されたがテストが失敗する」

**根本原因：** 形式仕様が制約をすべてキャプチャしていない（例：タイミング、パフォーマンス、失敗モード）。

**修正：**
1. 仕様に制約を追加：「payment.charge は 2秒以内に完了する必要がある」
2. エラー処理仕様を追加：「決済に失敗した場合、注文は pending に戻らなければならない」
3. 統合エージェントに負のテストも生成させる
4. ループバック：仕様更新 → フェーズ3で更新された契約を実装

### 問題3：「共有型の変更がすべてのモジュールを壊した」

**根本原因：** 変更管理プロセスがない。

**修正：**
1. フェーズ1後に共有型をフリーズ（明確な変更プロセスがない限り）
2. 変更が必要な場合：アーキテクトが影響をすべての契約に評価
3. 影響を受けるすべてのモジュール・エージェントが契約を再検証
4. その後のみ、変更を伝播
5. CHANGES.md にドキュメント化

### 問題4：「モジュール・エージェントが独自のエラー型を発明し続ける」

**根本原因：** 共有エラー処理契約がない。

**修正：**
1. フェーズ1で、アーキテクト・エージェントが共通エラー型を定義（Result[T], ApiError など）
2. フェーズ2で、モジュール・エージェントにエラー契約を明示的に表示
3. フェーズ3で、エージェントに共有エラー型を使用させる
4. プロンプトで同じエラーパターンの例を示す

### 問題5：「エージェントがコードを生成するのに時間がかかる」

**根本原因：** エージェントがコンテキストを持たないか、冗長すぎる。

**修正：**
1. Claude Projects を使用してコンテキストを保ち（単発メッセージではなく）
2. エージェント・コンテキスト・ドキュメントを簡潔に保つ
3. エージェントに簡潔さを促す：「実装コード、最小限のコメント」
4. 同様のモジュールからの実装例を提供
5. フェーズ3を小さなチャンク（操作ごと）に分割

---

## 既存ツールとの統合

### Claude Projects 使用（チーム向け推奨）

```
プロジェクト：注文システム形式仕様駆動開発

指示（プロジェクトに追加）：
[Templates/prompts/phase1-specification.md]

ファイル（アップロード）：
specs/.vdm/system-spec.vdmsl（フェーズ1で作成後）
specs/.vdm/shared-types.vdmsl
[モジュール仕様が作成されたら]

各フェーズ用に会話を作成：
├── フェーズ1-アーキテクチャ対話
│   ├ システム仕様プロンプトに従う
│   ├ ステークホルダーと対話して仕様を改善
│   └→ 出力：system-spec.vdmsl
│
├── フェーズ2-注文設計
│   ├ 注文モジュール仕様を表示
│   ├ 依存の事後条件を表示
│   └→ 出力：注文の DESIGN.md
│
├── フェーズ3-注文実装
│   ├ 設計ドキュメントを表示
│   ├ モジュール仕様を表示
│   └→ 出力：src/, tests/
│
└── フェーズ4-統合
    ├ すべてのモジュール + 仕様を表示
    ├ フェーズ4検証プロンプトを表示
    └→ 出力：integration-report.md
```

### カスタム・オーケストレーション・スクリプト（高度）

自動ハンドオフが必要な場合：

```python
#!/usr/bin/env python3
import anthropic

class マルチエージェント・オーケストレーター：
    def __init__(self):
        self.client = anthropic.Anthropic()

    def フェーズ1_アーキテクト(self, 要件ドキュメント):
        """フェーズ1を実行：システム仕様"""
        return self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=8000,
            system=self.プロンプト_読み込み("phase1-specification"),
            messages=[{
                "role": "user",
                "content": f"要件：\n{要件ドキュメント}\n形式仕様を作成しましょう..."
            }]
        )

    def フェーズ2_モジュール設計(self, モジュール仕様, 依存仕様):
        """フェーズ2を実行：モジュール設計"""
        コンテキスト = f"モジュール仕様：\n{モジュール仕様}\n\n依存仕様：\n{依存仕様}"
        return self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=4000,
            system=self.プロンプト_読み込み("phase2-design"),
            messages=[{
                "role": "user",
                "content": f"{コンテキスト}\n\nこのモジュールを設計してください..."
            }]
        )

    # 続きは同様...

# 使用方法：
orchestrator = マルチエージェント・オーケストレーター()

# フェーズ1
system_spec = orchestrator.フェーズ1_アーキテクト(要件)

# フェーズ2（各モジュール向け並行実行）
order_design = orchestrator.フェーズ2_モジュール設計(order_spec, 依存_specs)
# ...

# フェーズ3（各モジュール向け並行実行）
order_code = orchestrator.フェーズ3_実装(order_design, order_spec)
# ...

# フェーズ4
integration_report = orchestrator.フェーズ4_統合(全モジュール, 全仕様)
```

これにより、人間のチェックポイントを維持しながら、フル自動オーケストレーションが可能になります。

---

## まとめ

本ガイドは、形式仕様駆動開発を実践するための完全なプレイブックを提供します：

1. **フェーズ1：** 人間 + アーキテクト・エージェントが仕様を協同作成
2. **フェーズ2-4：** モジュール・エージェントが形式契約を使って並行開発
3. **統合：** 統合エージェントが契約を検証して合成可能性を確認

このアプローチにより：
- **スケーラビリティ：** 複数エージェントが独立して作業
- **品質：** 形式契約により互換性が保証される
- **速度：** 並行開発で時間短縮
- **信頼性：** 機械的検証により人的エラー削減

今すぐ始めてください！小さな3モジュール・システムで試し、スケールアップしましょう。
