# Claude Code × 形式仕様駆動開発 実践ガイド / Claude Code Integration Guide

*— CLAUDE.mdの階層構造を活用したマルチエージェント開発の具体的手法 —*

[日本語](#japanese--日本語) | [English](#english)

---

## Japanese / 日本語

### はじめに

本ガイドでは、形式仕様駆動開発をClaude Codeで実践する際の具体的な方法を解説します。核心は、Claude Codeが持つ**CLAUDE.mdの階層的ロード機構**が、形式仕様のスコープ分離と自然に対応するという発見です。

**参照元:** [Claude Code Memory Documentation](https://code.claude.com/docs/en/memory.md), [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices.md)

### CLAUDE.mdの仕組み

Claude Codeはセッション開始時に、ディレクトリ階層を遡って`CLAUDE.md`ファイルを収集し、コンテキストに注入します。

| スコープ | 配置場所 | ロードタイミング |
|:---|:---|:---|
| ユーザー | `~/.claude/CLAUDE.md` | 常時（全プロジェクト共通） |
| プロジェクトルート | `./CLAUDE.md` | セッション開始時に即座にロード |
| サブディレクトリ | `./modules/order/CLAUDE.md` | そのディレクトリ内のファイルに触れた時に**遅延ロード** |
| パス別ルール | `.claude/rules/*.md` | frontmatterの`paths`パターンに一致した時 |

この遅延ロード機構が、形式仕様駆動開発の「各エージェントは自モジュール仕様＋依存先IF仕様のみを持つ」という原則と一致します。

### 推奨ディレクトリ構成

```
project-root/
├── CLAUDE.md                           ← システム全体の仕様・アーキテクチャ契約
├── .claude/
│   └── rules/
│       ├── specification-phase.md      ← 仕様編集時の行動規約
│       └── implementation-phase.md     ← 実装編集時の行動規約
├── specs/
│   ├── shared-types.vdmsl             ← 全モジュール共通の型定義
│   ├── order-interface.vdmsl          ← Orderモジュールの公開IF契約
│   ├── inventory-interface.vdmsl      ← InventoryモジュールのIF契約
│   └── payment-interface.vdmsl        ← PaymentモジュールのIF契約
├── modules/
│   ├── order/
│   │   ├── CLAUDE.md                  ← Orderモジュール仕様 + 依存先IF参照
│   │   └── src/
│   ├── inventory/
│   │   ├── CLAUDE.md                  ← Inventoryモジュール仕様 + 依存先IF参照
│   │   └── src/
│   └── payment/
│       ├── CLAUDE.md                  ← Paymentモジュール仕様 + 依存先IF参照
│       └── src/
└── integration/
    └── CLAUDE.md                      ← 統合検証エージェント用の規約
```

### 各CLAUDE.mdの記述内容

#### ルートCLAUDE.md（常時ロード）

プロジェクト全体の設計判断とモジュール間依存関係を記述します。すべてのClaude Codeセッションがこの情報を持った状態で開始します。

```markdown
# Project Architecture

## Module Dependencies
- Order → Inventory (ReserveStock, ReleaseStock)
- Order → Payment (ProcessPayment, RefundPayment)
- Payment → Inventory (ConfirmShipment)

## Interface Contract Rule
All module boundaries must satisfy A.post ⇒ B.pre.
Never access another module's internal state directly.
Always use the interface operations defined in specs/*-interface.vdmsl.

## Shared Types
@specs/shared-types.vdmsl

## Development Phases
- Phase 1 (Specification): Human + AI define specs in specs/ directory
- Phase 2 (Design): Technical decisions within each module
- Phase 3 (Implementation): Code in modules/*/src/ against formal specs
- Phase 4 (Verification): Check A.post ⇒ B.pre across module boundaries
```

**ポイント：** `@specs/shared-types.vdmsl` 構文はClaude Codeのインポート機能で、外部ファイルの内容をCLAUDE.mdに展開します。共通型定義を全セッションで共有できます。

#### モジュール別CLAUDE.md（遅延ロード）

各モジュールのディレクトリに配置し、そのモジュールの仕様と依存先のインターフェースのみを参照します。

```markdown
# Order Module Specification

## This Module's Formal Specification
@specs/order.vdmsl

## Dependency Interface Contracts
This module depends on Inventory and Payment.
Their public interface contracts (NOT internal implementation):

@specs/inventory-interface.vdmsl
@specs/payment-interface.vdmsl

## Implementation Rules
- All operations MUST satisfy pre/post conditions defined in order.vdmsl
- NEVER import or access inventory or payment internal state
- Use ONLY the interface operations defined in the dependency contracts above
- When implementing an operation, verify that your postconditions
  satisfy the preconditions of any downstream module you call

## Testing Strategy
- Generate tests from pre/post conditions (boundary values from spec)
- Each precondition violation → expect specific error
- Each postcondition → assert in test
```

**遅延ロードの効果：** Claude Codeが`modules/order/src/`のファイルを読み書きする時にのみこのCLAUDE.mdがロードされます。Inventoryモジュールの作業中にはロードされません。これにより、各セッションのコンテキストウィンドウが自モジュール仕様＋依存先IF仕様のみで構成されます。

#### パス別ルール（.claude/rules/）

仕様ファイルと実装ファイルで異なる行動規約をClaude Codeに与えます。

**`.claude/rules/specification-phase.md`:**
```markdown
---
paths: ["specs/**"]
---
# Specification Phase Rules

When editing or creating specification files:
- All type changes MUST be reflected in dependent interface contracts
- After modifying a module spec, check A.post ⇒ B.pre for all dependencies
- Explain every spec change to the human in natural language:
  "This change means [plain language explanation]. Does this match your intent?"
- Flag any precondition that references a type not defined in shared-types.vdmsl
- When adding a new operation, explicitly list which modules will call it
```

**`.claude/rules/implementation-phase.md`:**
```markdown
---
paths: ["modules/*/src/**"]
---
# Implementation Phase Rules

When writing implementation code:
- Read the module's CLAUDE.md FIRST to load the formal specification
- Check every operation's pre/post conditions before implementing
- Generate tests that cover:
  - All precondition boundary values
  - All postcondition guarantees
  - All state invariant preservation
- Do NOT introduce state variables not declared in the VDM-SL spec
- Do NOT add operations not defined in the spec without updating the spec first
```

### コンテキスト管理の最適化

形式仕様がClaude Codeのコンテキスト管理においてなぜ有利かを定量的に示します。

| 方式 | コンテキストに入るもの | 推定行数 |
|:---|:---|:---|
| 自然言語仕様一括 | 全モジュールの要件定義書 | 500〜2,000行 |
| CLAUDE.md + VDM-SL | ルート契約 + 自モジュール仕様 + 依存先IF | 100〜300行 |

VDM-SLの簡潔さ（自然言語の1/3〜1/5の行数で同等の仕様を表現）がClaude Codeの推奨上限（CLAUDE.md 200行以内）と整合します。`@path/to/file` インポートで仕様本体を外部`.vdmsl`ファイルに分離し、CLAUDE.md自体は参照のみに保つことで管理性を維持できます。

### 実践ワークフロー

#### Phase 1: 仕様策定（Human + Claude Code）

```bash
# プロジェクト初期化
cd project-root/
claude   # Claude Codeセッション開始
```

Claude Codeとの対話で、ビジネス要件からVDM-SL仕様を導出します。

```
You: 「ECサイトの注文モジュールを設計したい。顧客がカートの商品を注文し、
      在庫確保→決済→出荷確認の流れがある」

Claude: 「以下の仕様を提案します。
  - OrderモジュールのCreateOrder操作:
    事前条件: カートが空でない、かつ全商品が在庫にある
    事後条件: 注文がPENDING状態で作成される
  この理解で正しいですか？」

You: 「在庫確認はInventoryモジュールの責任では？」

Claude: 「その通りです。OrderモジュールのConfirmOrderの事後条件で
  stock_reserved = trueを保証し、これがInventoryモジュールの
  ReserveStockの事前条件として機能します。
  つまりOrder.ConfirmOrder.post ⇒ Inventory.ReserveStock.pre
  というインターフェース契約です。」
```

この対話の成果物として`specs/`配下にVDM-SLファイルが生成され、ルートCLAUDE.mdに依存関係が記録されます。

#### Phase 2-3: モジュール並行開発

各モジュールを独立したClaude Codeセッションで開発します。

```bash
# ターミナル1: Orderモジュール
cd project-root/modules/order/
claude
# → ルートCLAUDE.md + modules/order/CLAUDE.md がロードされる
# → Order仕様 + Inventory/Payment IF契約のみがコンテキストに

# ターミナル2: Inventoryモジュール（並列）
cd project-root/modules/inventory/
claude
# → ルートCLAUDE.md + modules/inventory/CLAUDE.md がロードされる
# → Inventory仕様 + Order IF契約のみがコンテキストに
```

各セッションは互いの実装コードを見ず、インターフェース契約のみで開発します。

#### Phase 4: 統合検証

```bash
cd project-root/integration/
claude
```

統合検証セッションでは、全モジュールのインターフェース仕様を読み込み、`A.post ⇒ B.pre` を検査します。実装コードは読まず、仕様レベルのみで整合性を判定します。

### セッション間の一貫性

形式仕様がCLAUDE.mdに記載されていることで、Claude Codeの構造的な弱点——セッション間の整合性欠如——が緩和されます。

月曜日にOrderモジュールを実装し、金曜日にInventoryモジュールを実装する場合、従来は開発者の記憶が唯一の「契約」でした。CLAUDE.mdに形式仕様が記録されていれば、金曜日のセッションは自動的にOrderモジュールのインターフェース契約を読み込み、整合性を保った状態で開発を開始できます。

### 制限事項と注意点

1. **CLAUDE.mdのサイズ制約：** 各CLAUDE.md は200行以内が推奨。大規模な仕様は`@`インポートで外部ファイルに分離すること。
2. **遅延ロードの範囲：** サブディレクトリのCLAUDE.mdは、そのディレクトリ内のファイルに触れるまでロードされない。`/memory`コマンドで現在ロードされているCLAUDE.mdを確認可能。
3. **VDM-SLツールの未成熟：** 現時点ではVDM-SLの型検査をClaude Code内で自動実行する仕組みはない。形式仕様の整合性チェックはClaude Codeへの指示（プロンプト）で実施。
4. **適用範囲：** UI/UX、パフォーマンスチューニング、軽量バグ修正には過剰。対象はモジュール間境界の正しさが重要なビジネスロジックシステム。

---

## English

### Introduction

This guide explains how to practice formal-spec-driven development with Claude Code. The key insight is that Claude Code's **hierarchical CLAUDE.md loading mechanism** naturally maps to the scope separation of formal specifications.

**Reference:** [Claude Code Memory Documentation](https://code.claude.com/docs/en/memory.md), [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices.md)

### How CLAUDE.md Works

Claude Code collects `CLAUDE.md` files by walking up the directory tree at session start, injecting them into context.

| Scope | Location | Load Timing |
|:---|:---|:---|
| User | `~/.claude/CLAUDE.md` | Always (all projects) |
| Project root | `./CLAUDE.md` | Immediately at session start |
| Subdirectory | `./modules/order/CLAUDE.md` | **Lazy-loaded** when files in that directory are accessed |
| Path-specific rules | `.claude/rules/*.md` | When working with files matching the `paths` frontmatter pattern |

The lazy-loading mechanism aligns with the formal-spec-driven principle: "each agent needs only its own module spec + dependency interface specs."

### Recommended Directory Structure

```
project-root/
├── CLAUDE.md                           ← System-wide architecture contracts
├── .claude/
│   └── rules/
│       ├── specification-phase.md      ← Rules when editing specs
│       └── implementation-phase.md     ← Rules when writing code
├── specs/
│   ├── shared-types.vdmsl             ← Shared type definitions
│   ├── order-interface.vdmsl          ← Order module public interface
│   ├── inventory-interface.vdmsl      ← Inventory module interface
│   └── payment-interface.vdmsl        ← Payment module interface
├── modules/
│   ├── order/
│   │   ├── CLAUDE.md                  ← Order spec + dependency IF refs
│   │   └── src/
│   ├── inventory/
│   │   ├── CLAUDE.md                  ← Inventory spec + dependency IF refs
│   │   └── src/
│   └── payment/
│       ├── CLAUDE.md                  ← Payment spec + dependency IF refs
│       └── src/
└── integration/
    └── CLAUDE.md                      ← Integration verification rules
```

### CLAUDE.md Content for Each Level

#### Root CLAUDE.md (Always Loaded)

Documents system-wide architecture decisions and module dependencies. Every Claude Code session starts with this context.

```markdown
# Project Architecture

## Module Dependencies
- Order → Inventory (ReserveStock, ReleaseStock)
- Order → Payment (ProcessPayment, RefundPayment)
- Payment → Inventory (ConfirmShipment)

## Interface Contract Rule
All module boundaries must satisfy A.post ⇒ B.pre.
Never access another module's internal state directly.
Always use the interface operations defined in specs/*-interface.vdmsl.

## Shared Types
@specs/shared-types.vdmsl
```

**Key point:** The `@specs/shared-types.vdmsl` syntax is Claude Code's import feature, which expands the external file's content into the CLAUDE.md context.

#### Module-Level CLAUDE.md (Lazy-Loaded)

Placed in each module directory, referencing only that module's spec and its dependency interfaces.

```markdown
# Order Module Specification

## This Module's Formal Specification
@specs/order.vdmsl

## Dependency Interface Contracts
@specs/inventory-interface.vdmsl
@specs/payment-interface.vdmsl

## Implementation Rules
- All operations MUST satisfy pre/post conditions in order.vdmsl
- NEVER access inventory or payment internal state
- Use ONLY interface operations from dependency contracts
```

**Lazy-loading effect:** This CLAUDE.md loads only when Claude Code reads/writes files in `modules/order/src/`. During Inventory module work, it is not loaded. This ensures each session's context contains only its own module spec + dependency interface specs.

#### Path-Specific Rules (.claude/rules/)

Different behavioral rules for specification files vs. implementation files.

**`.claude/rules/specification-phase.md`:**
```markdown
---
paths: ["specs/**"]
---
# Specification Phase Rules
- All type changes MUST be reflected in dependent interface contracts
- After modifying a spec, verify A.post ⇒ B.pre for all dependencies
- Explain every spec change in natural language
```

**`.claude/rules/implementation-phase.md`:**
```markdown
---
paths: ["modules/*/src/**"]
---
# Implementation Phase Rules
- Read the module's CLAUDE.md FIRST to load the formal specification
- Check pre/post conditions before implementing each operation
- Generate tests covering all precondition boundary values
- Do NOT introduce state not declared in the VDM-SL spec
```

### Context Window Optimization

| Approach | Context Contents | Estimated Lines |
|:---|:---|:---|
| Natural language specs (all-in-one) | Full requirements for all modules | 500–2,000 lines |
| CLAUDE.md + VDM-SL | Root contracts + own module spec + dependency IFs | 100–300 lines |

VDM-SL's conciseness (1/3 to 1/5 the line count of equivalent natural language) aligns well with Claude Code's recommended CLAUDE.md size limit (under 200 lines).

### Practical Workflow

#### Phase 1: Specification (Human + Claude Code)

Start a Claude Code session at the project root to define system-wide specifications through dialogue.

```bash
cd project-root/
claude
```

The AI explains specs in natural language: "This says a customer can't place an order with an empty cart, and after placement, inventory is reserved. Does that match your business rules?"

Artifacts: VDM-SL files in `specs/`, dependency graph in root CLAUDE.md.

#### Phase 2-3: Parallel Module Development

Each module is developed in an independent Claude Code session.

```bash
# Terminal 1: Order module
cd project-root/modules/order/ && claude
# → Loads root CLAUDE.md + modules/order/CLAUDE.md
# → Context: Order spec + Inventory/Payment interface contracts only

# Terminal 2: Inventory module (parallel)
cd project-root/modules/inventory/ && claude
# → Loads root CLAUDE.md + modules/inventory/CLAUDE.md
# → Context: Inventory spec + Order interface contract only
```

Sessions never see each other's implementation code — only interface contracts.

#### Phase 4: Integration Verification

```bash
cd project-root/integration/ && claude
```

The integration session reads all interface specs and checks `A.post ⇒ B.pre` across module boundaries at the specification level only.

### Cross-Session Consistency

Formal specifications in CLAUDE.md mitigate Claude Code's structural weakness: lack of cross-session consistency.

If you implement Order on Monday and Inventory on Friday, the Friday session automatically loads Order's interface contract from CLAUDE.md and starts development with guaranteed consistency.

### Limitations

1. **CLAUDE.md size:** Keep each file under 200 lines. Use `@` imports for large specs.
2. **Lazy-loading scope:** Subdirectory CLAUDE.md files load only when files in that directory are accessed. Use `/memory` command to verify loaded files.
3. **VDM-SL tooling:** No automated VDM-SL type checking within Claude Code yet. Consistency checks are prompt-driven.
4. **Applicability:** Overkill for UI/UX, performance tuning, or simple bug fixes. Best suited for multi-module business logic systems.

---

## References

- [Claude Code Memory Documentation](https://code.claude.com/docs/en/memory.md)
- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices.md)
- [formal-spec-driven-dev](https://github.com/kotaroyamame/formal-spec-driven-dev) — Formal specification framework
- [Comparison of AI-Driven Development Architectures](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/master/docs/comparison.md)
