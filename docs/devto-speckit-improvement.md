---
title: "GitHub Spec Kit Is 80% Right — Here's the Missing 20% That Would Make It Transformative"
published: false
description: "Spec Kit gets the philosophy right: intent should be the source of truth, not code. But natural language specs can't verify themselves. Adding formal specifications would close the gap."
tags: ai, github, softwaredevelopment, opensource
cover_image:
canonical_url:
---

## I Love Spec Kit. And That's Why I Want to Push It Further.

GitHub's [Spec Kit](https://github.com/github/spec-kit) is, in my assessment, the most intellectually honest attempt at AI-driven development to date. While most tools focus on *faster code generation*, Spec Kit asks a more fundamental question:

> What if intent — not code — was the source of truth?

This is exactly the right question. The Constitution → Specify → Plan → Tasks → Implement workflow is well-designed. The steerable gates where humans can intervene are smart. The 25+ agent support (Claude Code, Copilot, Gemini CLI, Cursor, etc.) shows pragmatic thinking about ecosystem diversity. The 40+ community extensions demonstrate real traction.

I've been working on a [formal specification-driven development framework](https://github.com/kotaroyamame/formal-spec-driven-dev) that starts from the same premise — **specifications should drive development, not the other way around**. But after months of research and implementation, I believe Spec Kit has a structural gap that, if filled, would make it dramatically more powerful.

That gap is **formal verifiability of specifications themselves**.

## The Problem: Structured Ambiguity

Spec Kit specifications are written in structured Markdown. This is a massive improvement over the alternatives:

- MetaGPT passes natural language PRDs between agents → ambiguity accumulates at each handoff
- ChatDev relies on dialogue-based consensus → non-deterministic and non-reproducible
- Devin/SWE-Agent infers specs from code → circular reasoning (code *is* the spec)

Spec Kit avoids all of these pitfalls by making specs explicit, structured, and human-editable. But here's the uncomfortable truth:

**Structured natural language reduces ambiguity. It does not eliminate it.**

Consider a Spec Kit specification like this:

```markdown
## User Story: Add to Cart
The user can add a product to their shopping cart.
The cart should reflect the updated quantity.
```

This is clear to a human reader. But it leaves critical questions unanswered:

- Can the user add 0 items? Negative quantities?
- What happens when stock is insufficient?
- If the product is already in the cart, does the quantity replace or accumulate?
- Is there a maximum cart size?

These aren't edge cases. They're **boundary conditions** — the exact places where modules interact and where bugs hide. A human spec author might catch some of them. But the point of a specification system is that **the system itself should make missing boundaries visible**.

## What Formal Specifications Add

In [VDM-SL](https://github.com/kotaroyamame/formal-spec-driven-dev/tree/main/templates/vdm-sl) (Vienna Development Method - Specification Language), the same requirement looks like this:

```
AddToCart: CustomerId * ProductId * nat1 ==> ()
AddToCart(cid, pid, qty) ==
  carts(cid) := if pid in set dom carts(cid)
                then carts(cid) ++ {pid |-> carts(cid)(pid) + qty}
                else carts(cid) ++ {pid |-> qty}
pre  pid in set dom inventory
     and inventory(pid) >= qty
     and qty > 0
     and CardinalityItems(carts(cid)) < MAX_CART_SIZE
post pid in set dom carts(cid)
     and carts(cid)(pid) = carts~(cid)(pid) + qty  -- accumulate, not replace
```

Notice what happens:

1. **`nat1` as the type of `qty`** — zero and negative quantities are structurally impossible. Not "tested against," not "documented as invalid" — *impossible at the type level*.
2. **`pre` conditions** — stock sufficiency and cart size limits are **explicit preconditions**. An AI agent reading this spec cannot "forget" about them.
3. **`post` conditions** — the behavior is unambiguous: quantities accumulate (not replace). Any agent implementing this module knows exactly what the expected behavior is.
4. **Machine-verifiable** — if another module's postcondition guarantees `inventory(pid) >= qty`, we can mechanically verify that it satisfies this precondition. No human review needed for interface consistency.

## The Three Gaps Formal Specs Would Fill in Spec Kit

### Gap 1: Boundary Conditions Are Invisible in Natural Language

In Spec Kit today, boundary conditions depend on the spec author remembering to include them. This is a human-reliability problem — the very thing we're trying to engineer out of the system.

Formal specifications force boundary conditions to be explicit because the type system and preconditions won't compile without them. A VDM-SL spec with `qty: nat` but no precondition on stock levels is **visibly incomplete** — the type checker can flag it.

### Gap 2: Cross-Module Consistency Is Unverifiable

This is the critical scaling problem. Spec Kit writes specs per task (≈ per module), but there's no mechanism to verify that **Task A's output satisfies Task B's input requirements**.

With 3 modules, you have 3 pairwise interfaces to check — manageable by human review. With 10 modules, you have 45. With 20 modules, 190. Human review does not scale linearly.

Formal specifications solve this with **compositional verification**: if Order module's `ConfirmOrder` operation has a postcondition, and Inventory module's `ReserveStock` has a precondition, you can mechanically check whether `ConfirmOrder.post ⇒ ReserveStock.pre`. This is the `A.post ⇒ B.pre` pattern — and it scales linearly with module count, not quadratically.

```
Module A (Order)                Module B (Inventory)
┌─────────────────┐            ┌─────────────────┐
│  ConfirmOrder()  │            │  ReserveStock()  │
│  post:           │──verify───│  pre:            │
│   order.status   │            │   productId ∈    │
│    = <CONFIRMED> │   A.post   │    dom inventory │
│   ∧ stock        │    ⇒       │   ∧ quantity > 0 │
│    reserved      │   B.pre    │   ∧ available ≥  │
│                  │            │     quantity     │
└─────────────────┘            └─────────────────┘
```

### Gap 3: Specs Can't Validate Themselves

Spec Kit's philosophy is "intent is the source of truth." But there's a meta-problem: **how do you verify that the spec itself is internally consistent?**

Natural language specs can contain contradictions that only surface during implementation:

```markdown
- Users must complete checkout within 30 minutes
- Users can save their cart and resume later
- Inventory is reserved at checkout start
```

Are these contradictory? If a user saves their cart at minute 29, resumes 3 hours later, is inventory still reserved? A human reader might catch this. A type-checker on formal specifications *will* catch it — the invariant on reserved inventory duration would conflict with the "resume later" postcondition.

## A Concrete Proposal: Spec Kit + Formal Specs

I'm not proposing to replace Spec Kit's natural language layer. I'm proposing to **add a formal verification layer beneath it**. Here's how it could work within Spec Kit's existing workflow:

### Phase: Constitution (unchanged)
Project principles, values, guidelines — these remain in natural language. They're governance, not computation.

### Phase: Specify (enhanced)
The human writes intent in natural language (as today). Then, an AI agent **generates a VDM-SL formalization** of the spec and explains it back:

> "Your spec says users can add products to their cart. I've formalized this with these constraints: quantities must be positive integers, stock must be sufficient, and cart has a maximum of 50 items. The quantity behavior is accumulative — adding 3 of a product that already has 2 in the cart gives 5, not 3. Does this match your intent?"

The human validates meaning. The machine validates consistency.

### Phase: Plan (enhanced)
The technical plan now includes **interface contracts** between modules: which module's postconditions feed into which module's preconditions. These contracts are in VDM-SL, mechanically verifiable.

### Phase: Tasks (enhanced)
Each task carries not just a natural language description, but a **formal module specification** — the types, state, operations, pre/post conditions. The AI agent implementing the task has an unambiguous contract.

### Phase: Implement (unchanged)
Agents implement against formal specs rather than natural language descriptions. Claude Code, Copilot, Gemini CLI — any of the 25+ supported agents can read VDM-SL and generate conforming code.

### New Phase: Verify
A dedicated verification step checks `A.post ⇒ B.pre` across all module boundaries. This happens at the **spec level**, before any code runs. Integration problems are caught before implementation, not after.

## Why This Matters for Multi-Agent Development

Single-agent development (one developer + one AI) can get by with natural language specs. The human catches ambiguities in real time. But the industry is clearly moving toward **multi-agent development** — multiple AI agents building different modules in parallel.

In multi-agent scenarios, natural language ambiguity becomes a scaling crisis:

| Modules | Pairwise Interfaces | Human Review Feasibility |
|---------|--------------------|-----------------------|
| 3       | 3                  | Easy                  |
| 5       | 10                 | Manageable            |
| 10      | 45                 | Difficult             |
| 20      | 190                | Impractical           |

Formal interface contracts are the only known mechanism that scales: each contract is verified independently, so verification effort grows linearly with module count (O(n)), not quadratically (O(n²)).

## What I'm Not Saying

Let me be clear about what this proposal is **not**:

- **Not "Spec Kit is bad."** Spec Kit is the best-designed spec-driven framework available. The philosophy is right. The UX is right. The community extensions are impressive.
- **Not "everyone should learn VDM-SL."** The AI reads and writes the formal notation. The human verifies meaning in natural language. No formal methods expertise required.
- **Not "formal specs replace tests."** Tests still matter for integration testing, performance testing, and E2E validation. Formal specs replace tests as the *center* of correctness — not as the entire verification strategy.
- **Not "this works for everything."** UI/UX, performance tuning, and quick bug fixes don't benefit from formal specifications. This is for **multi-module business logic systems** where correctness at module boundaries is critical.

## The Open Source Connection

We've built a complete framework around this idea:

**[`formal-spec-driven-dev`](https://github.com/kotaroyamame/formal-spec-driven-dev)** — Apache 2.0 licensed

It includes VDM-SL templates, AI prompt templates for all 4 development phases, multi-agent orchestration configs, and a working e-commerce example (Order, Inventory, Payment modules with formal interface contracts).

Our [comparison analysis](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/master/docs/comparison.md) covers the structural critiques of MetaGPT, ChatDev, Devin, Spec Kit, and Claude Code in detail, with a unified framework for understanding each approach's "source of truth" and its logical limitations.

If you're using Spec Kit today and thinking about how to scale it to multi-agent or multi-module projects, I'd love to hear from you. The combination of Spec Kit's excellent developer experience and formal verification's mathematical guarantees could be exactly what the ecosystem needs.

---

*Hikaru Ando (安藤光太郎) — [IID Systems](https://iid.systems)*
*[GitHub: formal-spec-driven-dev](https://github.com/kotaroyamame/formal-spec-driven-dev) | Apache 2.0*

---

**References:**
- [GitHub Spec Kit](https://github.com/github/spec-kit) — The toolkit this article proposes to enhance
- [Spec-Driven Development with AI (GitHub Blog)](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [Diving Into Spec-Driven Development (Microsoft Developer Blog)](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)
- [formal-spec-driven-dev](https://github.com/kotaroyamame/formal-spec-driven-dev) — Our formal specification framework
- [Comparison of AI-Driven Development Architectures](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/master/docs/comparison.md) — Structural critique of each approach
