# E-commerce Order System: Formal Spec-Driven Development Example

*[日本語版は下にあります / Japanese version below]*

## Overview (English)

This example demonstrates the **formal-spec-driven development approach** applied to a realistic e-commerce order processing system. It shows how to:

1. **Decompose a complex system** into formally-specified modules (Inventory, Order, Payment)
2. **Define precise contracts** between modules using VDM-SL pre/post conditions
3. **Verify interface composition** to ensure modules integrate correctly
4. **Generate implementation prompts** that constrain the AI to follow formal specs

### What This Example Teaches

- **Module Decomposition**: How to break down an e-commerce order system into three independent, testable modules with clear responsibilities
- **Formal Specification**: Writing VDM-SL specs with explicit pre-conditions, post-conditions, and invariants
- **Interface Contracts**: Using operation signatures and conditions to define module boundaries
- **Composition Verification**: How to check that one module's output satisfies the next module's input requirements
- **AI Code Generation**: How formal specs produce high-fidelity AI-generated code

## System Architecture

```
┌─────────────────────────────────────────────────────┐
│             E-commerce Order System                  │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  │  Inventory   │  │    Order     │  │   Payment    │
│  │   Module     │  │   Module     │  │   Module     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
│         │                 │                 │
│         └─────────────────┴─────────────────┘
│         (composition verified)
│
└─────────────────────────────────────────────────────┘
```

### Module Responsibilities

**Inventory Module** (`.vdm/inventory.vdmsl`)
- Manages product stock across warehouses
- Tracks reserved inventory for in-process orders
- Operations: CheckStock, ReserveStock, ReleaseStock, RestockProduct
- Invariant: reserved stock ≤ total stock, stock ≥ 0

**Order Module** (`.vdm/order.vdmsl`)
- Creates and manages orders
- Coordinates with Inventory for stock reservation
- Tracks order lifecycle: PENDING → CONFIRMED → COMPLETED
- Operations: CreateOrder, ConfirmOrder, CancelOrder, GetOrderStatus

**Payment Module** (`.vdm/payment.vdmsl`)
- Processes payments for confirmed orders
- Tracks payment status and refunds
- Integrates with Order module for payment authorization
- Operations: InitiatePayment, ProcessPayment, RefundPayment

## Module Interfaces and Composition

### Composition Chain

```
Inventory.ReserveStock.post
    ↓
    └─→ Order.CreateOrder.pre ✓

Order.ConfirmOrder.post
    ↓
    └─→ Payment.InitiatePayment.pre ✓

Payment.RefundPayment.post
    ↓
    └─→ Inventory.ReleaseStock.pre ✓
```

Each arrow represents a verified **composition point** where the post-condition of one operation satisfies the pre-condition of another.

### Key Design Decisions

1. **Reservation Pattern**: Inventory uses ReserveStock/ReleaseStock to prevent overselling. Reserved stock is tracked separately from available stock.

2. **Order State Machine**: Orders follow a strict state progression (PENDING → CONFIRMED → COMPLETED). This constrains valid operations at each state.

3. **Payment Isolation**: Payment module depends only on OrderId and amount, not inventory details. This reduces coupling.

4. **Idempotency**: Operations like RefundPayment check if already processed to support retries safely.

## How to Use This Example

### For Tech Leads

1. **Study the Module Specs** (5-10 minutes each)
   - Read `.vdm/inventory.vdmsl` to understand formal specification syntax
   - Note the pre-conditions, post-conditions, and invariants
   - Understand how state is represented

2. **Verify Composition** (10 minutes)
   - Open `.vdm/integration-check.md`
   - For each composition point, verify that the post-condition of one module's operation logically implies the pre-condition of the next module's operation
   - This is the key to ensuring modules work together correctly

3. **Generate Implementation** (20-30 minutes)
   - Use the prompts in the sections below to ask Claude to implement each module
   - Start with Inventory (simplest), then Order, then Payment
   - Each prompt includes the full VDM-SL spec

### For AI Code Generation

#### Inventory Module Prompt

```
Generate a Python implementation of this VDM-SL specification for an inventory system:

[INSERT CONTENTS OF .vdm/inventory.vdmsl]

Requirements:
- Implement each operation exactly as specified
- Include pre-condition checks that raise PreconditionViolation if violated
- Implement post-condition assertions to verify correctness
- Use the type system and invariants to guide design
- Include comprehensive docstrings referencing the spec

Generate complete, production-ready code with unit tests.
```

#### Order Module Prompt

```
Generate a Python implementation of this VDM-SL specification for an order system:

[INSERT CONTENTS OF .vdm/order.vdmsl]

This module depends on the Inventory module with this interface:
- ReserveStock(productId, quantity) → reserved_amount
- ReleaseStock(productId, reserved_amount)

Requirements:
- Implement each operation exactly as specified
- Pre-condition checks must verify Inventory preconditions are met
- Post-condition checks must verify Inventory postconditions hold
- Include comprehensive docstrings referencing the spec

Generate complete, production-ready code with unit tests.
```

#### Payment Module Prompt

```
Generate a Python implementation of this VDM-SL specification for a payment system:

[INSERT CONTENTS OF .vdm/payment.vdmsl]

This module depends on the Order module with this interface:
- GetOrderStatus(orderId) → OrderStatus
- ConfirmOrder(orderId) → OrderId

Requirements:
- Implement each operation exactly as specified
- Pre-condition checks must verify Order preconditions are met
- Post-condition checks must verify Order postconditions hold
- Include comprehensive docstrings referencing the spec

Generate complete, production-ready code with unit tests.
```

## File Structure

```
examples/ec-site-order/
├── README.md (this file)
├── .vdm/
│   ├── inventory.vdmsl
│   ├── order.vdmsl
│   ├── payment.vdmsl
│   └── integration-check.md
└── (generated implementations would go here)
```

---

## 概要（日本語）

このサンプルは、**フォーマル仕様駆動開発アプローチ**を現実的なEコマース注文処理システムに適用した例です。以下を示します：

1. **複雑なシステムの分解**: 正式に仕様記述された3つのモジュール（在庫・注文・決済）への分割
2. **精密なコントラクト定義**: VDM-SLの事前条件・事後条件を用いたモジュール間インターフェース定義
3. **インターフェース合成の検証**: モジュール統合が正しく機能することの確認
4. **AI実装プロンプト生成**: 形式仕様からAIに制約を与える実装プロンプト生成

### このサンプルが教えること

- **モジュール分解**: Eコマース注文システムを3つの独立した、テスト可能なモジュールに分割する方法
- **形式仕様**: VDM-SLで事前条件・事後条件・不変量を含む仕様の記述方法
- **インターフェースコントラクト**: 操作シグネチャと条件を用いたモジュール境界の定義
- **合成検証**: あるモジュールの出力が次のモジュールの入力要件を満たすことの確認
- **AI実装生成**: 形式仕様から高精度なAI生成コード生成方法

## システムアーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│          Eコマース注文システム                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  │   在庫モジュール │  │   注文モジュール  │  │   決済モジュール  │
│  │   (Inventory)   │  │   (Order)     │  │   (Payment)   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
│         │                 │                 │
│         └─────────────────┴─────────────────┘
│         （合成検証済み）
│
└─────────────────────────────────────────────────────┘
```

### モジュール責務

**在庫モジュール** (`.vdm/inventory.vdmsl`)
- 倉庫別の商品在庫を管理
- 処理中注文用の予約在庫を追跡
- 操作: CheckStock, ReserveStock, ReleaseStock, RestockProduct
- 不変量: 予約在庫 ≤ 総在庫、在庫 ≥ 0

**注文モジュール** (`.vdm/order.vdmsl`)
- 注文の作成・管理
- 在庫モジュールとの連携で在庫予約を調整
- 注文ライフサイクル追跡: PENDING → CONFIRMED → COMPLETED
- 操作: CreateOrder, ConfirmOrder, CancelOrder, GetOrderStatus

**決済モジュール** (`.vdm/payment.vdmsl`)
- 確定済み注文の決済処理
- 支払い状態と払い戻しを追跡
- 注文モジュールとの統合で決済認可
- 操作: InitiatePayment, ProcessPayment, RefundPayment

## モジュールインターフェースと合成

### 合成チェーン

```
Inventory.ReserveStock.post
    ↓
    └─→ Order.CreateOrder.pre ✓

Order.ConfirmOrder.post
    ↓
    └─→ Payment.InitiatePayment.pre ✓

Payment.RefundPayment.post
    ↓
    └─→ Inventory.ReleaseStock.pre ✓
```

各矢印は**合成点**を表し、一方の操作の事後条件が他方の事前条件を満たすことが検証されています。

### 主要な設計判断

1. **予約パターン**: 在庫は ReserveStock/ReleaseStock で過剰販売を防止。予約在庫を利用可能在庫と別途追跡。

2. **注文状態機械**: 注文は厳密な状態遷移に従う (PENDING → CONFIRMED → COMPLETED)。各状態での有効操作を制限。

3. **決済分離**: 決済モジュールは OrderId と金額のみに依存。在庫詳細には依存せず。結合度を低減。

4. **冪等性**: RefundPayment のような操作は処理済みかを確認。リトライを安全にサポート。

## このサンプルの使用方法

### テックリード向け

1. **モジュール仕様の研究** (各5～10分)
   - `.vdm/inventory.vdmsl` を読んで形式仕様の文法を理解
   - 事前条件・事後条件・不変量に注目
   - 状態表現を理解

2. **合成検証** (10分)
   - `.vdm/integration-check.md` を開く
   - 各合成点で、一方のモジュール操作の事後条件が他方の事前条件を論理的に含むかを検証
   - これがモジュール間統合の正確性を保証するキー

3. **実装生成** (20～30分)
   - 下記のセクションのプロンプトを使用して、各モジュール実装をClaudeに依頼
   - 在庫（最もシンプル）から開始し、注文、決済の順に進む
   - 各プロンプトは完全なVDM-SL仕様を含む

### AI実装生成向け

#### 在庫モジュールプロンプト

```
このVDM-SL仕様の在庫システムのPython実装を生成してください:

[.vdm/inventory.vdmsl の内容を挿入]

要件:
- 仕様通りに各操作を実装
- 事前条件チェックで違反時に PreconditionViolation 例外を発生
- 事後条件アサーションで正確性を検証
- 型システムと不変量を設計ガイドとして使用
- 仕様を参照する詳細なドキストリングを含める

本番対応の完全なコードとユニットテストを生成してください。
```

#### 注文モジュールプロンプト

```
このVDM-SL仕様の注文システムのPython実装を生成してください:

[.vdm/order.vdmsl の内容を挿入]

このモジュールは在庫モジュールに以下のインターフェースで依存します:
- ReserveStock(productId, quantity) → reserved_amount
- ReleaseStock(productId, reserved_amount)

要件:
- 仕様通りに各操作を実装
- 事前条件チェックで在庫モジュール事前条件を検証
- 事後条件チェックで在庫モジュール事後条件を検証
- 仕様を参照する詳細なドキストリングを含める

本番対応の完全なコードとユニットテストを生成してください。
```

#### 決済モジュールプロンプト

```
このVDM-SL仕様の決済システムのPython実装を生成してください:

[.vdm/payment.vdmsl の内容を挿入]

このモジュールは注文モジュールに以下のインターフェースで依存します:
- GetOrderStatus(orderId) → OrderStatus
- ConfirmOrder(orderId) → OrderId

要件:
- 仕様通りに各操作を実装
- 事前条件チェックで注文モジュール事前条件を検証
- 事後条件チェックで注文モジュール事後条件を検証
- 仕様を参照する詳細なドキストリングを含める

本番対応の完全なコードとユニットテストを生成してください。
```

## ファイル構成

```
examples/ec-site-order/
├── README.md (このファイル)
├── .vdm/
│   ├── inventory.vdmsl
│   ├── order.vdmsl
│   ├── payment.vdmsl
│   └── integration-check.md
└── (生成された実装はここに配置)
```

---

## Key Learnings

This example demonstrates:
- How formal specs make module boundaries explicit and testable
- Why composition verification is essential before implementation
- How AI code generation becomes reliable when constrained by formal specs
- The economics of specification cost vs. implementation risk reduction

Study this example before building your own formal-spec-driven system.
