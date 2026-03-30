# formal-spec-driven-dev

**Formal Specifications as Contracts for Multi-Agent AI Development**

<div align="center">

![Banner](docs/images/banner.png)

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Language](https://img.shields.io/badge/Language-VDM--SL%20%7C%20Markdown%20%7C%20YAML-green.svg)](#)
[![Status](https://img.shields.io/badge/Status-Active%20Development-brightgreen.svg)](#)

[日本語](#japanese--日本語) | [English](#english)

</div>

---

## Japanese / 日本語

### ビジョン

AI駆動開発において、複数のエージェントがチームのように協調するためには何が必要か？

従来のテスト駆動開発（TDD）は、帰納的推論に基づいています。テストケースを書いて実装を検証しますが、それはあくまで特定のケースの正当性を示すだけです。一方、形式仕様（Formal Specifications）は演繹的推論を提供します。

**本プロジェクトの核となる仮説：** 形式仕様（VDM-SL）をエージェント間の「契約」として機能させることで、以下が実現できる：

- 複数のAIエージェントが疎結合で並行開発可能
- 各エージェントは自分が担当するモジュール仕様と依存モジュールのインターフェース仕様のみ必要
- モジュール間の互換性を機械的に検証可能（A.post ⇒ B.pre 合成可能性）
- 人間の役割が「ドメイン専門家 + アーキテクチャ決定者」へシフト
- 形式的な保証により、エージェントが生成したコードの正当性を機械的に検証

### このリポジトリについて

本リポジトリは、形式仕様駆動開発のアプローチを体系化し、実践可能にするための完全なプレイブックです。VDM-SLの仕様テンプレート、複数エージェントを協調させるプロンプト、実装例を提供します。

**対象者：** 技術リード・アーキテクト、AI駆動開発の評価・導入を検討している組織

### クイックスタート（3ステップ）

#### 1. 論文を読む
まずは基礎となる論文を理解してください：

- **日本語版:** [`docs/ja/paper.md`](docs/ja/paper.md) - 形式仕様駆動開発の理論と実践
- **英語版:** [`docs/en/paper.md`](docs/en/paper.md) - English full article

読了時間: 20-30分

#### 2. テンプレートを試す
提供されているテンプレートを使用して、小規模なモジュール仕様を作成してみてください：

```bash
# VDM-SL仕様テンプレートの確認
cd templates/vdm-sl/
cat module-template.vdmsl

# AIプロンプトテンプレートの確認
cd templates/prompts/
ls -la
```

詳細は [`templates/vdm-sl/README.md`](templates/vdm-sl/README.md) を参照

#### 3. 実装例を実行
実際のE-commerceオーダーシステムの例を見て、プロセス全体を理解してください：

```bash
cd examples/ec-site-order/
cat README.md
```

このサンプルでは、以下を確認できます：
- 3つのモジュール（注文、在庫、決済）の仕様
- 各エージェントが受け取ったプロンプト
- VDM-SLから生成された実装コード

### 主要な概念

#### 形式仕様を「契約」として機能させる

```
モジュールA                    モジュールB
┌─────────────┐               ┌─────────────┐
│ 事前条件   │               │ 事前条件   │
│ (precond)  │               │ (precond)  │
│            │               │            │
│ 事後条件   │───契約─────→│ 事前条件   │
│ (postcond) │               │ (precond)  │
│            │    A.post     │            │
└─────────────┘    ⇒         │ 事後条件   │
                   B.pre     │ (postcond) │
                             └─────────────┘
```

A の事後条件が B の事前条件を満たせば、機械的に合成可能性が検証できます。

#### エージェントの役割分担

1. **ドメイン専門家エージェント**
   - ビジネス要件から形式仕様を対話的に導出
   - モジュール間のインターフェース設計

2. **実装エージェント**
   - 形式仕様を与えられて、実装を生成
   - テストケース生成も自動化

3. **検証エージェント**
   - VDM-SLスペックの形式的検証を実行
   - 仕様間の合成可能性を確認

#### TDD vs. 形式仕様駆動開発

| 項目 | TDD（帰納的） | 形式仕様駆動（演繹的） |
|------|-------------|-------------------|
| 推論方法 | テストケース → 正当性の帰納 | 仕様 → 実装の演繹 |
| 検証スコープ | テストケースが対象する部分 | 仕様全体をカバー |
| エージェント協調 | テストを共有・参照 | 仕様を共有・参照 |
| スケーラビリティ | テスト数の増加に伴うメンテナンスコスト | 仕様の明確さによる |
| 機械的保証 | なし | あり |

### リポジトリ構成

```
formal-spec-driven-dev/
├── README.md                          # このファイル
├── LICENSE                            # Apache 2.0
├── CONTRIBUTING.md                    # 貢献ガイド
│
├── docs/
│   ├── ja/
│   │   ├── paper.md                   # 論文（日本語版）
│   │   └── architecture-guide.md      # 複数エージェント・アーキテクチャガイド
│   ├── en/
│   │   ├── paper.md                   # 論文（英語版）
│   │   └── architecture-guide.md      # Multi-agent Architecture Guide
│   └── images/                        # ダイアグラム等
│
├── templates/
│   ├── vdm-sl/
│   │   ├── module-template.vdmsl      # 単一モジュール用テンプレート
│   │   ├── interface-contract.vdmsl   # モジュール間契約用テンプレート
│   │   └── README.md                  # VDM-SL使用方法
│   │
│   ├── prompts/
│   │   ├── phase1-specification.md    # フェーズ1：仕様対話プロンプト
│   │   ├── phase2-design.md           # フェーズ2：設計プロンプト
│   │   ├── phase3-implementation.md   # フェーズ3：実装プロンプト
│   │   └── phase4-verification.md     # フェーズ4：検証プロンプト
│   │
│   └── orchestration/
│       ├── agent-config.yaml          # エージェント役割定義
│       └── workflow.md                # オーケストレーションワークフロー
│
├── examples/
│   └── ec-site-order/
│       ├── README.md                  # このサンプルの説明
│       ├── .vdm/                      # VDM-SL仕様ファイル
│       │   ├── order-module.vdmsl
│       │   ├── inventory-module.vdmsl
│       │   └── payment-module.vdmsl
│       └── prompts/                   # 実際に使用したプロンプト
│
└── .github/
    └── ISSUE_TEMPLATE/                # Issue テンプレート
```

### 貢献方法

このプロジェクトへの貢献を歓迎します。以下の方法でご参加ください：

1. **フィードバック・意見提案**
   - GitHub Issues で機能提案・バグ報告を作成

2. **ドキュメント改善**
   - 日本語・英語の文章改善、例の追加

3. **新しいテンプレート・例の提供**
   - 異なるドメインのVDM-SLテンプレート
   - 新しいプロンプトパターン

4. **実装への協力**
   - 検証ツールの改善
   - オーケストレーション機能の拡張

詳細は [`CONTRIBUTING.md`](CONTRIBUTING.md) を参照してください。

### ライセンス

このプロジェクトは Apache License 2.0 の下で公開されています。
詳細は [`LICENSE`](LICENSE) を参照してください。

### 関連リンク

- **論文（日本語）:** [docs/ja/paper.md](docs/ja/paper.md)
- **論文（英語）:** [docs/en/paper.md](docs/en/paper.md)
- **Claude Code実践ガイド:** [docs/claude-code-integration.md](docs/claude-code-integration.md) — CLAUDE.mdの階層構造を活用した具体的な開発方法
- **マルチエージェント・アーキテクチャガイド（日本語）:** [docs/ja/architecture-guide.md](docs/ja/architecture-guide.md)
- **マルチエージェント・アーキテクチャガイド（英語）:** [docs/en/architecture-guide.md](docs/en/architecture-guide.md)
- **VDM-SLテンプレート使用方法:** [templates/vdm-sl/README.md](templates/vdm-sl/README.md)

### 著者

**Hikaru Ando** (ando@iid.systems)
IID Systems

---

## English

### Vision

What does it take for multiple AI agents to collaborate like a development team?

Traditional Test-Driven Development (TDD) relies on inductive reasoning. Writing test cases validates an implementation, but only for specific scenarios. Formal specifications, by contrast, provide deductive reasoning.

**The core hypothesis of this project:** By making formal specifications (VDM-SL) function as "contracts" between agents, we can achieve:

- Multiple AI agents developing in parallel with loose coupling
- Each agent needing only their own module spec and dependent module interface specs
- Mechanical verification of module compatibility (A.post ⇒ B.pre composability)
- A shift in human roles to "domain expert + architecture decision maker"
- Formal guarantees enabling mechanical verification of agent-generated code

### About This Repository

This repository systematizes the formal-specification-driven development approach and makes it practically deployable. It provides VDM-SL specification templates, prompts for coordinating multiple agents, and working examples.

**Intended for:** Tech leads and architects evaluating and adopting AI-driven development within their organizations

### Quick Start (3 Steps)

#### 1. Read the Paper
First, understand the foundational theory:

- **Japanese version:** [`docs/ja/paper.md`](docs/ja/paper.md) - Theory and practice of formal-specification-driven development
- **English version:** [`docs/en/paper.md`](docs/en/paper.md) - Full English article

Reading time: 20-30 minutes

#### 2. Try the Templates
Use the provided templates to create a small module specification:

```bash
# Review VDM-SL specification templates
cd templates/vdm-sl/
cat module-template.vdmsl

# Review AI prompt templates
cd templates/prompts/
ls -la
```

Details available in [`templates/vdm-sl/README.md`](templates/vdm-sl/README.md)

#### 3. Run the Example
Examine the e-commerce order system example to understand the entire process:

```bash
cd examples/ec-site-order/
cat README.md
```

The sample demonstrates:
- Specifications for three modules (order, inventory, payment)
- Actual prompts given to each agent
- Implementation code generated from VDM-SL specs

### Key Concepts

#### Formal Specifications as Contracts

```
Module A                       Module B
┌─────────────┐               ┌─────────────┐
│ Precondition│               │ Precondition│
│ (precond)   │               │ (precond)   │
│             │               │             │
│ Postcondition           ──→│ Precondition│
│ (postcond)  │───Contract    │ (precond)   │
│             │    A.post ⇒   │             │
└─────────────┘    B.pre     │ Postcondition
                             │ (postcond)  │
                             └─────────────┘
```

When A's postcondition satisfies B's precondition, mechanical composability verification becomes possible.

#### Agent Role Distribution

1. **Domain Expert Agent**
   - Derives formal specifications from business requirements through dialogue
   - Designs module interfaces

2. **Implementation Agent**
   - Generates implementation from formal specs
   - Automates test case generation

3. **Verification Agent**
   - Executes formal verification of VDM-SL specs
   - Confirms composability between specs

#### TDD vs. Formal-Specification-Driven Development

| Aspect | TDD (Inductive) | Formal-Spec-Driven (Deductive) |
|--------|-----------------|--------------------------------|
| Reasoning | Test cases → Inductive proof | Spec → Deductive implementation |
| Verification scope | Only tested cases | Entire specification |
| Agent coordination | Shared test suite | Shared specification |
| Scalability | Test maintenance overhead | Specification clarity |
| Mechanical guarantee | None | Yes |

### Repository Structure

```
formal-spec-driven-dev/
├── README.md                          # This file
├── LICENSE                            # Apache 2.0
├── CONTRIBUTING.md                    # Contribution Guidelines
│
├── docs/
│   ├── ja/
│   │   ├── paper.md                   # Paper (Japanese)
│   │   └── architecture-guide.md      # Multi-agent Architecture Guide (JP)
│   ├── en/
│   │   ├── paper.md                   # Paper (English)
│   │   └── architecture-guide.md      # Multi-agent Architecture Guide (EN)
│   └── images/                        # Diagrams and illustrations
│
├── templates/
│   ├── vdm-sl/
│   │   ├── module-template.vdmsl      # Single module template
│   │   ├── interface-contract.vdmsl   # Inter-module contract template
│   │   └── README.md                  # How to use VDM-SL templates
│   │
│   ├── prompts/
│   │   ├── phase1-specification.md    # Phase 1: Specification dialogue prompt
│   │   ├── phase2-design.md           # Phase 2: Design prompt
│   │   ├── phase3-implementation.md   # Phase 3: Implementation prompt
│   │   └── phase4-verification.md     # Phase 4: Verification prompt
│   │
│   └── orchestration/
│       ├── agent-config.yaml          # Agent role definitions
│       └── workflow.md                # Orchestration workflow guide
│
├── examples/
│   └── ec-site-order/
│       ├── README.md                  # This example explained
│       ├── .vdm/                      # VDM-SL specification files
│       │   ├── order-module.vdmsl
│       │   ├── inventory-module.vdmsl
│       │   └── payment-module.vdmsl
│       └── prompts/                   # Actual prompts used
│
└── .github/
    └── ISSUE_TEMPLATE/                # Issue templates
```

### How to Contribute

We welcome contributions to this project. You can participate in the following ways:

1. **Feedback and Suggestions**
   - Create issues for feature requests or bug reports on GitHub

2. **Documentation Improvements**
   - Improve Japanese and English documentation
   - Add examples and clarifications

3. **New Templates and Examples**
   - Provide VDM-SL templates for different domains
   - Contribute new prompt patterns

4. **Implementation Support**
   - Improve verification tools
   - Enhance orchestration features

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for detailed guidelines.

### License

This project is released under the Apache License 2.0.
See [`LICENSE`](LICENSE) for details.

### Related Links

- **Paper (Japanese):** [docs/ja/paper.md](docs/ja/paper.md)
- **Paper (English):** [docs/en/paper.md](docs/en/paper.md)
- **Claude Code Integration Guide:** [docs/claude-code-integration.md](docs/claude-code-integration.md) — Practical guide using CLAUDE.md hierarchy for formal-spec-driven development
- **Multi-Agent Architecture Guide (Japanese):** [docs/ja/architecture-guide.md](docs/ja/architecture-guide.md)
- **Multi-Agent Architecture Guide (English):** [docs/en/architecture-guide.md](docs/en/architecture-guide.md)
- **VDM-SL Templates Usage:** [templates/vdm-sl/README.md](templates/vdm-sl/README.md)

### Author

**Hikaru Ando** (ando@iid.systems)
IID Systems

---

<div align="center">

Made with commitment to formal methods and AI-driven development.

</div>
