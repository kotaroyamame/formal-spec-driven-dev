# Multi-Agent Orchestration Reference

このディレクトリには、形式仕様駆動開発におけるマルチエージェント・ワークフローを実装するためのテンプレートとガイドが含まれています。

This directory contains templates and guides for implementing multi-agent workflows in formal-specification-driven development.

---

## Files / ファイル

### 1. `agent-config.yaml`

**Purpose:** Agent role definitions and configuration template

**内容：**
- Agent roles (architect, module agents, integration agent)
- Phase breakdown (Phase 1-4)
- Context requirements for each agent
- Data flow between phases
- Module boundary templates
- Cross-cutting concerns (error handling, logging, auth)
- Human decision points and escalation procedures
- Tooling recommendations

**使い方：**
このファイルを参考に、あなたのプロジェクト用にカスタマイズしてください。各フェーズでどのエージェントが何をするべきか、どんなアウトプットが期待されるかを理解します。

### 2. `workflow.md`

**Purpose:** Complete bilingual (EN/JP) guide for orchestrating agents

**内容：**
- Multi-agent workflow overview with ASCII diagrams
- Phase-by-phase detailed walkthrough:
  - Phase 1: System architecture & formal specification (human + architect)
  - Phase 2: Module design (parallel per-module agents)
  - Phase 3: Implementation (parallel per-module agents)
  - Phase 4: Integration & verification (integration agent)
- How to handle cross-cutting concerns
- Shared data model management
- When human intervention is needed
- Practical implementation tips (TODAY):
  - Using Claude Projects
  - Creating specification repositories
  - Handoff checklists
  - Agent context documents
  - Contract verification
  - Troubleshooting guide
- Real timeline example (E-commerce order system)
- Integration with existing tools

**使い方：**
このガイドを読んで、マルチエージェント・ワークフローの全体像を理解してください。実装の詳細なステップが含まれているため、すぐに今日からプロジェクトに適用できます。

---

## How to Get Started / 開始方法

### Step 1: Understand the Concepts

1. Read [`workflow.md`](workflow.md) first (20-30 minutes) - especially the Phase overview section
2. Review [`agent-config.yaml`](agent-config.yaml) to see the agent role definitions
3. Refer back to the main project [`README.md`](../../README.md) for theoretical background

### Step 2: Set Up Your Project Structure

```bash
# Your project should look like:
my-project/
├── specs/
│   └── .vdm/
│       ├── system-spec.vdmsl       # Created in Phase 1
│       ├── shared-types.vdmsl
│       ├── module1.vdmsl
│       ├── module2.vdmsl
│       └── ...
├── modules/
│   ├── module1/
│   │   ├── DESIGN.md               # Phase 2 output
│   │   ├── src/                    # Phase 3 output
│   │   └── tests/
│   ├── module2/
│   │   └── ...
│   └── ...
├── integration/
│   ├── composability-report.md     # Phase 4 output
│   └── integration-tests.py
└── orchestration/
    ├── agent-config.yaml
    ├── workflow.md
    └── agent-context.md            # You create this
```

### Step 3: Create Agent Context Documents

For each phase and agent, create a context document:

```markdown
# Order Module Agent Context - Phase 3

## Your Module Spec
[Full VDM-SL of order module]

## Dependency Interface Specs
[Postconditions only from inventory and payment modules]

## Shared Types & Error Handling
[Customer, Product, Money, Result[T], etc.]

## Current Phase
Phase 3: Implementation

## Success Criteria
- Code satisfies pre/post conditions
- Unit tests cover preconditions
- Can compose with inventory and payment
```

### Step 4: Run Phases Sequentially

#### Phase 1 (1-3 days)

Use **Claude Projects**:
- Create project: "System Specification"
- Upload `templates/prompts/phase1-specification.md`
- Include: `templates/vdm-sl/module-template.vdmsl`
- Dialogue with architect agent to create system spec
- Output: `specs/.vdm/system-spec.vdmsl` + module specs

**Checklist before Phase 2:**
```
□ System spec is valid VDM-SL
□ Module boundaries are clear
□ Public operations specified for each module
□ Dependency graph is acyclic
□ Stakeholder approval obtained
```

#### Phase 2 (1-2 days per module, can be parallel)

For each module:
- Create new Claude Project or conversation
- Provide: own module spec + dependency specs (postconditions only)
- Use: `templates/prompts/phase2-design.md`
- Output: `modules/{module}/DESIGN.md`

**Checklist before Phase 3:**
```
□ Design explains internal structure
□ Operation decomposition is clear
□ How dependencies are called
□ Error handling strategy defined
□ Tech lead reviewed and approved
```

#### Phase 3 (2-5 days per module, can be parallel)

For each module:
- Continue project from Phase 2 or create new one
- Provide: design doc + module spec
- Use: `templates/prompts/phase3-implementation.md`
- Output: `modules/{module}/src/` + `tests/`

**Checklist before Phase 4:**
```
□ Implementation matches design and spec
□ Unit tests derived from preconditions
□ Code passes review
□ Module agent confirms spec satisfaction
```

#### Phase 4 (1-2 days)

- Create Claude Project: "Integration & Verification"
- Provide: all modules + all specs + system spec
- Use: `templates/prompts/phase4-verification.md`
- Output: `integration/composability-report.md` + integration tests

**Final GO/NO-GO Decision:**
```
□ All contracts verified (A.post ⇒ B.pre)
□ Integration tests pass
□ Tech lead reviews and approves
→ Ready for deployment
```

---

## Using This with Claude Projects

### Project Setup

```
Project: "Order System - Formal Spec Driven"

Instructions:
  [Copy relevant phase prompt here]

Files (upload as you progress):
  - specs/.vdm/system-spec.vdmsl
  - specs/.vdm/shared-types.vdmsl
  - modules/order/DESIGN.md
  - modules/order/src/order.py
  - etc.

Conversations (create for each phase/module):
  ├── Phase1-SystemSpecification
  ├── Phase2-OrderModuleDesign
  ├── Phase2-InventoryModuleDesign
  ├── Phase2-PaymentModuleDesign
  ├── Phase3-OrderModuleImplementation
  ├── Phase3-InventoryModuleImplementation
  ├── Phase3-PaymentModuleImplementation
  └── Phase4-IntegrationVerification
```

Each conversation maintains context across 20+ turns, allowing agents to work iteratively without losing context.

---

## Key Principles

### 1. Formal Specs as Contracts

Agents don't hand off code—they hand off **formal specifications**. Each agent only needs:
- Their own module spec (VDM-SL)
- Interface specs of dependencies (postconditions only)

This decouples modules and enables parallel development.

### 2. A.postcond ⇒ B.precond

The core verification in Phase 4: for every inter-module call, verify that the caller's postcondition satisfies the callee's precondition.

```
Order.finalize_order.post ⇒ Inventory.deduct.pre?
Order.finalize_order.post ⇒ Payment.charge.pre?

If both ✓, modules compose!
```

### 3. Human in the Loop

Critical decision points require human approval:
- Phase 1 spec approval
- Phase 2 design review
- Phase 3 code review
- Phase 4 integration report + GO/NO-GO

---

## Troubleshooting

### "Agents designed things that don't work together"

→ **Fix:** Have agents explicitly quote dependency postconditions in Phase 2. Add architect review before Phase 3.

### "Contracts verified but integration tests fail"

→ **Fix:** Specs are incomplete. Add constraints (timing, failure modes) and loop back to Phase 3.

### "Module agents invented different error handling"

→ **Fix:** Define shared error types in Phase 1 (Result[T], ApiError). Show examples in Phase 2/3.

### "It's taking too long"

→ **Fix:** Use Claude Projects (not one-shot messages) to maintain context. Keep agent context docs concise. Break Phase 3 into smaller chunks.

---

## Real Example: E-Commerce Order System

See [`workflow.md` → Real Example section](workflow.md#example-workflow-e-commerce-order-system-real) for a complete timeline:

- **Phase 1 (1-2 days):** Architect defines order, inventory, payment specs
- **Phase 2 (3-4 days):** Three agents design modules in parallel
- **Phase 3 (5-7 days):** Three agents implement modules in parallel
- **Phase 4 (8-9 days):** Integration agent verifies contracts and integration tests

**Total: 9 days vs. 4-6 weeks for traditional serial approach**

---

## Next Steps

1. **Read [workflow.md](workflow.md) in full** to understand all phases
2. **Customize [agent-config.yaml](agent-config.yaml)** for your system
3. **Create specs directory structure**
4. **Set up first Claude Project** for Phase 1
5. **Start Phase 1 dialogue** with architect agent
6. **Follow the Phase 1-4 checklists** as you progress

---

## References

- **Phase Prompts:** See `templates/prompts/` for:
  - `phase1-specification.md` - System architecture dialogue
  - `phase2-design.md` - Module design
  - `phase3-implementation.md` - Code generation
  - `phase4-verification.md` - Integration verification

- **VDM-SL Templates:** See `templates/vdm-sl/` for:
  - `module-template.vdmsl` - Single module spec
  - `interface-contract.vdmsl` - Inter-module contract

- **Main Documentation:**
  - `docs/ja/paper.md` - Theory (Japanese)
  - `docs/en/paper.md` - Theory (English)

---

## Support & Questions

If you have questions about:
- **How to use these templates:** Refer to [workflow.md](workflow.md)
- **YAML configuration:** See [agent-config.yaml](agent-config.yaml) comments
- **Specific phases:** Look up Phase N section in [workflow.md](workflow.md)
- **VDM-SL syntax:** See `templates/vdm-sl/README.md`
