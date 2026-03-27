# AI Prompt Templates for Formal-Spec-Driven Development

These templates provide **copy-paste-ready system prompts** for using Claude, GPT-4, or similar LLMs to execute the four phases of formal-spec-driven AI development.

## The Four Phases

### Phase 1: Specification Dialogue / 仕様対話
**File:** `phase1-specification.md`

**Purpose:** Translate natural-language business requirements into a formal VDM-SL specification.

**Role:** Specification Architect (仕様設計家)

**Key Activities:**
- Dialogue with stakeholders to understand domain and requirements
- Define types with invariants that encode business rules
- Model system state and consistency rules
- Specify operations with pre/post-conditions
- Explore edge cases and design questions
- Produce formal VDM-SL spec + natural-language summary

**Input:** Business requirements (natural language, user stories, acceptance criteria)
**Output:** VDM-SL specification file + specification summary document

**Template Size:** 300 lines (comprehensive system prompt + usage guide)

---

### Phase 2: Technical Design / 技術設計
**File:** `phase2-design.md`

**Purpose:** Translate VDM-SL specification into concrete technical design decisions.

**Role:** Technical Architect

**Key Activities:**
- Analyze VDM-SL spec and extract design constraints
- Gather non-functional requirements (scale, performance, availability, cost)
- Propose overall architecture (monolith vs. microservices, sync vs. async, etc.)
- Select technology stack (language, framework, database, deployment)
- Map VDM-SL types to data models with invariant enforcement
- Design service operations with transaction/locking strategies
- Address Phase 1 design questions with implementation decisions
- Document rationale for every major choice

**Input:** VDM-SL specification + non-functional requirements
**Output:** Technical design document with architecture diagram, data model, service specs

**Template Size:** 433 lines

---

### Phase 3: Implementation / 実装
**File:** `phase3-implementation.md`

**Purpose:** Generate production-ready code from VDM-SL spec and design.

**Role:** Software Engineer (also following the role of formal verification in code)

**Key Activities:**
- Implement type definitions that encode business rules (make invalid states unrepresentable)
- Implement data access layer (repositories) with invariant enforcement
- Implement service layer (business logic respecting pre/post-conditions)
- Implement API layer with request/response validation
- Write tests that verify post-conditions and invariants
- Organize code to reflect VDM-SL module structure
- Document code with VDM-SL traceability

**Input:** VDM-SL specification + technical design document
**Output:** Production-ready code (modules: types, repositories, services, API) + unit/integration tests

**Template Size:** 806 lines

---

### Phase 4: Verification / 検証
**File:** `phase4-verification.md`

**Purpose:** Systematically verify that implemented code satisfies the VDM-SL specification.

**Role:** Formal Verification Engineer

**Key Activities:**
- Extract testable properties from VDM-SL (pre/post-conditions, invariants)
- Generate pre-condition tests (verify invalid inputs are rejected)
- Generate post-condition tests (verify successful operations guarantee their contracts)
- Generate invariant tests (verify state invariants hold before and after operations)
- Generate property-based tests (verify properties for all possible inputs)
- Generate concurrency tests (verify atomicity and isolation)
- Generate edge case tests (boundary values, absence cases, temporal edge cases)
- Generate cross-module interface tests (verify module contracts)
- Produce verification report with coverage metrics

**Input:** VDM-SL specification + implementation code
**Output:** Comprehensive test suite + verification report showing all tests passing

**Template Size:** 923 lines

---

## How to Use These Templates

### Option 1: Use the Exact Prompt as System Instruction

1. **Choose a phase** based on what you're working on
2. **Copy the entire system prompt section** from the relevant file (delimited by triple backticks)
3. **Paste it into your LLM's system prompt or custom instructions**
4. **Replace placeholders** marked `{{VARIABLE}}` with your specific context
5. **Start the dialogue** by asking the LLM to begin the phase work

Example for Phase 1:
```
I want to specify an inventory management system. Here's the context:
- Domain: E-commerce
- Key actors: Warehouse managers, fulfillment staff
- Primary risk: Overselling (selling more than we have)
- Language preference: English

Let's start the specification dialogue. I'll describe the system and you guide me through the VDM-SL specification process.
```

### Option 2: Adapt the Prompt to Your Workflow

- **Bilingual teams:** Customize the LANGUAGE parameter to "ja" for Japanese or "bilingual" for mixed
- **Different frameworks:** Phase 3 has language/framework parameters; adapt examples if using a different stack
- **Specific compliance:** Phase 2 accepts regulatory constraints (GDPR, HIPAA, PCI-DSS); add your requirements
- **Custom testing approach:** Phase 4 shows pytest/Hypothesis; adapt to your test framework

### Option 3: Build a Custom LLM Workflow

These templates can be chained into an automated workflow:

```
Phase 1 → (LLM using phase1 prompt) → VDM-SL spec
Phase 2 → (LLM using phase2 prompt) → Design doc
Phase 3 → (LLM using phase3 prompt) → Code + tests
Phase 4 → (LLM using phase4 prompt) → Verification report
```

Example orchestration (pseudocode):
```python
# Phase 1: Create spec
spec = llm.run_phase(
    prompt=load_file("phase1-specification.md"),
    context={"SYSTEM_NAME": "Inventory Manager", "DOMAIN": "e-commerce"}
)

# Phase 2: Design from spec
design = llm.run_phase(
    prompt=load_file("phase2-design.md"),
    context={
        "VDMSL_SPEC": spec,
        "NON_FUNCTIONAL_REQUIREMENTS": {"scale": "100 ops/sec", "cost": "minimal"}
    }
)

# Phase 3: Implement from design
code = llm.run_phase(
    prompt=load_file("phase3-implementation.md"),
    context={"VDMSL_SPEC": spec, "DESIGN_DOC": design, "LANGUAGE": "python"}
)

# Phase 4: Verify implementation
report = llm.run_phase(
    prompt=load_file("phase4-verification.md"),
    context={"VDMSL_SPEC": spec, "IMPLEMENTATION": code}
)
```

---

## Key Features of These Templates

### Comprehensive System Prompts
Each prompt is 300+ lines and includes:
- Core principles and mindset for the phase
- Step-by-step protocol (what to do, in what order)
- Detailed examples (code samples, dialogue snippets)
- Common pitfalls to avoid
- Parameters to customize
- VDM-SL primer (types, operations, syntax)
- Best practices specific to the phase

### Bilingual Support
All templates support English and Japanese:
- Core content is in English (for technical clarity)
- Japanese translations for names and explanations
- Parameter `LANGUAGE` controls output language
- Useful for teams that prefer Japanese discussions or documentation

### Practical, Copy-Paste Ready
- No hypothetical examples — all examples are production-grade
- Code samples show patterns you can use directly
- Dialogue examples show exactly what to ask/expect
- Clear separation between prompt (what to copy) and guidance (how to use it)

### VDM-SL Expertise Built-In
No need to be a VDM-SL expert:
- Phase 1 teaches the notation through dialogue (human never reads VDM-SL directly)
- Phase 2 explains VDM-SL-to-design mapping
- Phase 3 shows VDM-SL-to-code traceability
- Phase 4 extracts testable properties from VDM-SL
- All phases include concrete syntax and examples

### Correctness Guarantees
The templates emphasize:
- **Phase 1:** Precise specification (no ambiguity)
- **Phase 2:** Design decisions justified by spec constraints
- **Phase 3:** Code structure that makes invalid states unrepresentable
- **Phase 4:** Comprehensive tests proving correctness

---

## Workflow Examples

### Example 1: Specify a New Feature

Use **Phase 1** to nail down requirements before coding:

```
Tech Lead: "We need a shipping calculator that applies regional surcharges. Let's use VDM-SL to get the spec right."

LLM (Phase 1 prompt): "Let me understand your domain. Who calculates shipping? When? What rules apply to surcharges?"

Tech Lead: "A microservice calculates shipping. Rules are: base cost + weight surcharge + region surcharge. US/Europe/Asia each have different rates."

LLM: "Let me model this formally. Here's a type for ShippingRegion and a post-condition that guarantees the right surcharge is applied..."

[Dialogue continues until spec is solid]

Output: VDM-SL spec ensuring no region is forgotten, no surcharge is double-counted, etc.
```

### Example 2: Design System for Scale

Use **Phase 2** to make architectural decisions transparent:

```
Tech Lead: "We have a spec. Now we need to design for 100k ops/sec with <100ms latency. What's the right architecture?"

LLM (Phase 2 prompt): "Let me analyze your non-functional requirements. At 100k ops/sec, a monolith with PostgreSQL might bottleneck. Let me propose a sharded design..."

[LLM proposes data sharding strategy, caching layer, async processing for non-critical updates]

Output: Design doc showing exactly where invariants are enforced (DB constraints, app logic, caching invalidation), and why each choice was made.
```

### Example 3: Generate Implementation Fast

Use **Phase 3** to go from design to code:

```
Tech Lead: "We have design. Generate code module-by-module."

LLM (Phase 3 prompt): "I'll start with types. Here's how I'll encode the business rules so invalid states are impossible to construct..."

[LLM generates types, then repository, then service, then API, with tests for each]

Output: Production-ready code where every operation respects VDM-SL contracts.
```

### Example 4: Verify Correctness Comprehensively

Use **Phase 4** to prove the code is correct:

```
Tech Lead: "Is the implementation correct? Run verification."

LLM (Phase 4 prompt): "I'll generate tests for every VDM-SL property. Pre-conditions, post-conditions, invariants, edge cases, concurrency..."

[LLM generates 100+ tests checking every aspect of the spec]

Output: Verification report: ✅ 63 tests passed, 0 failed. All invariants verified. Code ready for production.
```

---

## File Structure

```
templates/prompts/
├── README.md (this file)
├── phase1-specification.md          (300 lines)
├── phase2-design.md                 (433 lines)
├── phase3-implementation.md         (806 lines)
└── phase4-verification.md           (923 lines)
```

---

## Common Customizations

### Customize for Your Language/Framework

**Phase 3** generates code. Adjust for your stack:
```
{{LANGUAGE}}: "Python" → (change to Java, TypeScript, Rust, etc.)
{{FRAMEWORK}}: "FastAPI" → (change to Express, Spring Boot, Actix, etc.)
{{DATABASE}}: "PostgreSQL" → (change to MongoDB, DynamoDB, etc.)
```

### Customize for Your Compliance Needs

**Phase 2** accepts regulatory constraints:
```
{{REGULATORY}}: "GDPR" → (design adds data retention, consent tracking)
{{REGULATORY}}: "HIPAA" → (design adds encryption, audit logging)
{{REGULATORY}}: "PCI-DSS" → (design avoids storing payment card data)
```

### Customize for Your Team

**Phase 1** language setting:
```
{{LANGUAGE}}: "en" → English dialogue and documentation
{{LANGUAGE}}: "ja" → Japanese dialogue and documentation
{{LANGUAGE}}: "bilingual" → Mix as needed
```

---

## Glossary

- **VDM-SL:** Vienna Development Method – Specification Language. A formal specification notation for defining system requirements precisely.
- **Pre-condition:** An assumption that must be true before an operation runs. If violated, the operation's behavior is not specified.
- **Post-condition:** A guarantee that will be true after an operation succeeds. If verified, the operation is correct.
- **Invariant:** A property that must always be true. In Phase 1, state invariants (e.g., "stock >= 0"). In Phase 3, these invariants are enforced in code.
- **Formal Specification:** A mathematical (precise, unambiguous) description of what a system must do.
- **Correctness by Construction:** Building code so that invalid states are impossible (type system prevents them) and operations provably maintain invariants.

---

## Next Steps

1. **Choose a phase** relevant to your current work (starting with Phase 1 if you're beginning a new system)
2. **Read the template** to understand the phase's approach
3. **Copy the system prompt** section (delimited by triple backticks)
4. **Customize {{VARIABLE}} placeholders** with your context
5. **Paste into your LLM** and start the dialogue

---

## Support & Questions

These templates are self-contained and come with extensive guidance. If you have questions:
- **Phase 1 specific:** See the "Dialogue Protocol" and "Edge Case Exploration" sections
- **Phase 2 specific:** See the "Design Protocol" and "VDM-SL to Implementation Mapping" sections
- **Phase 3 specific:** See the "Implementation Protocol" and "Type Definitions" sections
- **Phase 4 specific:** See the "Verification Protocol" and "Test Coverage" sections

All templates follow the same structure:
1. **Overview** — What this phase does
2. **System Prompt** (copy-paste section)
3. **Core Principles** — Mindset for the phase
4. **Protocol** — Step-by-step procedure
5. **Practical Examples** — Code, dialogue, test patterns
6. **Common Pitfalls** — What to avoid
7. **Parameters** — What to customize
8. **Usage Guide** — How to use the template
9. **Example Session** — A real dialogue snippet

---

**Version:** 1.0
**Updated:** 2026-03-28
**Language:** English (bilingual-compatible)
