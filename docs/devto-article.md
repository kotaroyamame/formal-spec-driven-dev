---
title: "Stop Writing Tests First — Write Formal Specs. Let AI Agent Teams Build Your System."
published: false
description: "How VDM-SL formal specifications can serve as contracts between AI agents, enabling multi-agent autonomous development. Open-source framework with templates."
tags: ai, softwareengineering, architecture, opensource
cover_image:
canonical_url:
---

## The Problem Nobody's Talking About

Everyone's excited about AI-powered coding. GitHub Copilot autocompletes your functions. Claude and GPT-4 generate entire modules from prompts. But here's the uncomfortable truth: **we're still thinking about AI as a single tool**, when the real paradigm shift is **multiple AI agents working as a team**.

And that raises a question nobody's answering well:

> When three AI agents are building three modules in parallel, what prevents them from producing code that doesn't fit together?

Natural language specs? Too ambiguous. One agent interprets "the order must be valid" differently from another. Test-driven development? Tests are inductive — they verify specific cases, not the contract between modules. You can have 100% coverage on each module and still watch the system crash at integration.

## What If Agents Had Blueprints?

Think about how a construction project works. You don't hand three subcontractors a paragraph of prose and hope for the best. You give them **engineering drawings** — precise, unambiguous documents that define every interface: where the plumbing connects to the electrical, what load the foundation bears, which walls are structural.

Formal specifications are the engineering drawings of software. And a method called **VDM-SL** (Vienna Development Method - Specification Language), originally from IBM in the 1970s, turns out to be nearly ideal for AI agents to read, write, and reason about.

Here's why it matters for multi-agent development:

```
Module A (Order)                Module B (Inventory)
┌─────────────────┐            ┌─────────────────┐
│                  │            │                  │
│  PlaceOrder()    │            │  ReserveStock()  │
│  post:           │──contract──│  pre:            │
│   order.status   │            │   productId ∈    │
│    = <CONFIRMED> │   A.post   │    dom inventory │
│   ∧ stock        │    ⇒       │   ∧ quantity > 0 │
│    reserved      │   B.pre    │   ∧ available ≥  │
│                  │            │     quantity     │
└─────────────────┘            └─────────────────┘
```

**A's postcondition implies B's precondition.** This is mechanically verifiable. No ambiguity. No "alignment meetings" between agents. No integration surprises.

## A Concrete Example

Here's what a VDM-SL specification for an order module actually looks like:

```
module Order

types
  OrderId = nat1;
  ProductId = nat1;

  LineItem :: productId : ProductId
             quantity : nat1
             unitPrice : nat;

  OrderStatus = <PENDING> | <CONFIRMED> | <COMPLETED> | <CANCELLED>;

  Order :: orderId    : OrderId
           customerId : nat1
           lineItems  : seq of LineItem
           status     : OrderStatus
           totalAmount : nat;

state OrderStore of
  orders    : map OrderId to Order
  nextOrderId : nat1
inv mk_OrderStore(orders, nextOrderId) ==
  nextOrderId > 0
  and forall id in set dom orders & orders(id).orderId = id

operations

CreateOrder: nat1 * seq of LineItem ==> OrderId
CreateOrder(customerId, items) ==
  let id = nextOrderId in (
    orders := orders ++ {id |-> mk_Order(id, customerId, items,
                          <PENDING>, SumItems(items))};
    nextOrderId := nextOrderId + 1;
    return id
  )
pre items <> []
post RESULT = nextOrderId~ - 1
     and RESULT in set dom orders
     and orders(RESULT).status = <PENDING>;
```

Notice what this gives you:

- **Types with invariants** — `nat1` means positive integers only. No null, no zero.
- **State invariant** — the `inv` clause is always true. Every order ID in the map matches the order's own ID.
- **Pre-conditions** — `items <> []` means you can't create an empty order. This is a _structural guarantee_, not a test case.
- **Post-conditions** — after `CreateOrder`, the order exists in the store with status `PENDING`. Any agent reading this knows exactly what to expect.

An AI agent building the inventory module doesn't need to see the Order module's implementation. It only needs the **interface contract** — the types and operation signatures with pre/post conditions. That's a fraction of the context window.

## The Multi-Agent Architecture

Here's the workflow we propose:

### Phase 1: Human + Architect AI (1-3 days)

The human (domain expert) and an architect AI collaborate to define:
- System-wide VDM-SL specification
- Module decomposition
- Interface contracts between modules (the `A.post ⇒ B.pre` relationships)

**The human doesn't need to read VDM-SL.** The AI explains the spec in natural language: "This says a customer can't place an order unless they have at least one item, and after the order is placed, inventory is reserved. Does that match your business rules?"

### Phase 2-4: Parallel Module Agents

Each module gets its own AI agent. Agent A builds Order. Agent B builds Inventory. Agent C builds Payment. **In parallel.**

Each agent's context contains only:
- Its own module spec (~200-500 lines of VDM-SL)
- Interface contracts of dependent modules (~50-100 lines each)
- Cross-cutting decisions from Phase 1

No agent needs another agent's implementation code. This fits comfortably in current context windows.

### Integration Verification

A dedicated integration agent checks `A.post ⇒ B.pre` across all module boundaries. It never reads implementation code — only interface specs. If Order's `ConfirmOrder` postcondition guarantees `stock_reserved = true`, and Inventory's `ShipOrder` precondition requires `stock_reserved = true`, the composition is verified.

```
┌──────────────────────────────────────────────────┐
│            Phase 1: Architecture                  │
│     Human + Architect AI                          │
│     → System spec + module decomposition          │
│     → Interface contracts (A.post ⇒ B.pre)        │
└────────────┬─────────────┬───────────────┬───────┘
             │             │               │
             ▼             ▼               ▼
     ┌──────────┐  ┌──────────┐   ┌──────────┐
     │ Agent A  │  │ Agent B  │   │ Agent C  │
     │ Order    │  │Inventory │   │ Payment  │
     │ Module   │  │ Module   │   │ Module   │
     └─────┬────┘  └────┬─────┘   └────┬─────┘
           │             │               │
           ▼             ▼               ▼
     ┌──────────────────────────────────────────┐
     │      Integration Verification Agent       │
     │   Checks A.post ⇒ B.pre across all       │
     │   module boundaries (spec-level only)     │
     └──────────────────────────────────────────┘
```

## Why Not Just Use Natural Language Specs?

Three reasons:

**1. Ambiguity scales exponentially.** With 2 modules and natural language specs, you have a manageable amount of misinterpretation risk. With 10 modules and 45 pairwise interactions, the ambiguity explodes. Formal specs eliminate this at the root — `nat1` means one thing, everywhere, always.

**2. Verification becomes mechanical.** You can't machine-check whether "the order should be valid" in one module agrees with "valid orders have at least one item" in another. But you _can_ machine-check whether `Order.ConfirmOrder.post ⇒ Inventory.ReserveStock.pre`.

**3. Context windows stay small.** Each agent needs only its module spec + dependency interfaces — not the full natural-language requirements document that keeps growing. A 10-module system might have 200 pages of natural language specs but only 50 lines of interface contract per module boundary.

## Why TDD Doesn't Solve This

This is the part that might get me some angry comments, but hear me out.

Dijkstra said it in 1969:

> "Program testing can be used to show the presence of bugs, but never to show their absence."

TDD is **inductive reasoning**: you verify specific inputs and hope they generalize. Formal specification is **deductive reasoning**: you state what must always be true, then derive implementations that satisfy those conditions.

For a single developer, TDD is a useful discipline. For multi-agent AI development, it fails structurally:

| Aspect | TDD | Formal Specs |
|--------|-----|-------------|
| Reasoning | Inductive (specific → general) | Deductive (general → specific) |
| Inter-agent contract | Implicit in shared tests | Explicit in interface specs |
| Verification | Run tests, hope for the best | Mechanical proof of composition |
| Context per agent | Needs test suite + code + shared state | Needs only interface contracts |
| Scales with agents | Coordination overhead grows | Contracts remain verifiable |

Note: we're not saying "never write tests." Tests still have a role — integration tests for external systems, performance tests, E2E UI tests. What we're saying is: **tests are no longer the center.** Formal specifications are.

## The Human Role: Not Obsolete, Elevated

This paradigm doesn't remove humans. It redefines what humans do:

**Before:** Write code, write tests, review code, debug integration failures.

**After:**
- **Domain expert** — "This is how our business works. These are the rules."
- **Architecture decision maker** — "Split the system into these modules. This module depends on that one."
- **Quality judge** — "The AI says the spec means X. Does X match what we actually need?"

You don't need to read VDM-SL. You don't need to write code. You need **deep domain knowledge** and **the ability to ask sharp questions** when the AI explains its specifications back to you.

## The Open-Source Framework

We've packaged this entire methodology as an open-source project:

**[`formal-spec-driven-dev`](https://github.com/kotaroyamame/formal-spec-driven-dev)** — Apache 2.0 licensed

What's included:

📄 **Foundational paper** (Japanese + English) — the full theoretical argument, ~30 min read

📋 **VDM-SL templates** — module template, interface contract template. Copy, customize, go.

🤖 **AI prompt templates for all 4 phases:**
- Phase 1: Specification dialogue (elicit requirements → VDM-SL)
- Phase 2: Technical design (VDM-SL → architecture decisions)
- Phase 3: Implementation (VDM-SL → production code)
- Phase 4: Verification (code ↔ spec cross-check)

🔧 **Multi-agent orchestration:**
- `agent-config.yaml` — role definitions, context requirements, outputs
- `workflow.md` — step-by-step guide to running multi-agent development today

📦 **Working example:** E-commerce order system with 3 modules (Order, Inventory, Payment), complete with VDM-SL specs, integration verification, and cross-module contracts.

## Try It in 30 Minutes

1. Clone the repo
2. Open `templates/prompts/phase1-specification.md`
3. Copy the system prompt into Claude or GPT-4
4. Start describing a module from your own project
5. Watch the AI produce a VDM-SL spec, then explain it back to you in plain language

That's it. No VDM-SL expertise required. The AI reads and writes the formal notation. You verify the _meaning_.

## What We Need from the Community

This is early-stage. The methodology is sound, but the tooling ecosystem needs work:

- **Domain templates** — Healthcare, fintech, logistics, IoT. Each domain has its own patterns. We need VDM-SL templates for common domain models.
- **Orchestration tooling** — Better ways to coordinate multiple AI agents with shared specifications. Integration with existing agent frameworks.
- **Case studies** — Real teams trying this on real projects. What worked? What broke? Where are the rough edges?
- **Verification automation** — Tools that automatically check `A.post ⇒ B.pre` across module boundaries.

If any of this resonates, check out the repo and open an issue. PRs welcome. Case study reports are especially valuable — even "I tried this and it didn't work because..." helps enormously.

## The Bottom Line

The era of single-agent AI coding is already giving way to multi-agent development. The unsolved problem is coordination. Natural language specs create ambiguity that scales exponentially with the number of agents. Tests verify specific cases but can't guarantee compositional correctness.

Formal specifications — specifically, VDM-SL with pre/post conditions and invariants — provide what multi-agent development needs: **unambiguous, mechanically verifiable contracts between agents**.

The human role doesn't disappear. It elevates. You become the domain expert and architect who ensures the AI team builds the right thing. The specs ensure they build it correctly.

---

*Hikaru Ando (安藤光太郎) — IID Systems*
*[GitHub: formal-spec-driven-dev](https://github.com/kotaroyamame/formal-spec-driven-dev) | Apache 2.0*

---

*This article is also available in [Japanese (日本語)](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/main/docs/ja/paper.md).*
