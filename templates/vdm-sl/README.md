# VDM-SL Template Guide: Formal Specification for AI-Driven Development

*Available in: [English](#english) | [日本語](#日本語)*

---

## English

### Overview

These VDM-SL templates enable **tech leads and AI agents** to write formally verifiable specifications that are:

- **Unambiguous**: Pre/post conditions make implicit assumptions explicit
- **Composable**: One module's output can safely feed into another's input
- **Automatable**: AI agents can verify correctness and detect integration issues
- **Executable**: Templates can be validated using VDM-SL tools (Overture, VDMJ)

### What's Included

1. **module-template.vdmsl** - A complete module specification showing:
   - Type definitions with invariants (domain model)
   - State definition with consistency rules
   - Operations with explicit pre/post conditions
   - Helper functions for common queries
   - Detailed comments for AI agent interpretation

2. **interface-contract.vdmsl** - A contract definition showing:
   - Exported types and operations visible to other modules
   - Compositional verification patterns
   - Safe operation sequences with proof
   - Dependency expectations (what other modules must provide)
   - State invariants that must always hold

3. **README.md** - This guide (you are here)

### Quick Start

#### Step 1: Choose Your Domain

The templates use an "Order Management" system as the example. Replace this with your domain:

- E-commerce → Replace `OrderManagement` with your service name
- Payment processing → Adapt operations for payment lifecycle
- Authentication → Redefine types for users, tokens, sessions
- Any stateful system → Follow the same structural pattern

#### Step 2: Define Your Types

In `module-template.vdmsl`, section 2 (TYPE DEFINITIONS):

```vdmsl
types

  -- Your domain concept
  MyEntity = nat
    inv entity == entity > 0;  -- Invariant: positive IDs only

  -- Enum for lifecycle
  MyEntityStatus = <CREATED> | <ACTIVE> | <ARCHIVED>;

  -- Complex record
  MyRecord :: id : nat
             status : MyEntityStatus
             inv mk_MyRecord(i, s) == i > 0;
```

**Why invariants matter**:
- Type system prevents invalid instances
- AI agents can assume invariants always hold
- Reduces need for runtime checks

#### Step 3: Define State + Invariants

In `module-template.vdmsl`, section 3 (STATE DEFINITION):

```vdmsl
state MyState of
  entities : map nat to MyRecord
  next_id : nat

  inv state ==
    -- All IDs in map are positive
    (forall id in set dom entities & id > 0)
    -- next_id is unique
    and (forall id in set dom entities & id < next_id)
    -- Consistency: entity ID matches map key
    and (forall id in set dom entities & entities(id).id = id)

  init s == s = mk_MyState({}, 1)
end
```

**Why this matters**:
- Single source of truth for consistency rules
- Every operation must preserve this invariant
- AI agents verify: `inv_MyState(state)` before/after each operation

#### Step 4: Write Operations with Contracts

In `module-template.vdmsl`, section 4 (OPERATIONS):

```vdmsl
operations

  CreateEntity(name : str)
    eid : nat
  pre
    -- What must be true BEFORE this operation
    len name > 0  -- Name cannot be empty
  post
    -- What will be true AFTER this operation
    RESULT > 0    -- Returns positive ID
    and (let created = GetEntity(RESULT)
         in created.status = <CREATED>);

  UpdateEntity(eid : nat, new_status : MyEntityStatus)
    result : OperationResult
  pre
    eid > 0
    and (let entity = GetEntity(eid)
         in IsValidTransition(entity.status, new_status))
  post
    (RESULT = <SUCCESS>)
    and (let updated = GetEntity(eid)
         in updated.status = new_status
         and updated.id = eid);
```

**Reading the contract**:

| Element | Meaning | For AI Agents |
|---------|---------|---------------|
| `pre` | Client responsibility | "I must verify this before calling" |
| `post` | System guarantee | "I can rely on this after the call" |
| `RESULT` | Return value | Reference the operation's return in postcondition |
| `~` (tilde) | Previous state | `entities~(id)` = state before operation |

### Relationship: module-template ↔ interface-contract

```
module-template.vdmsl
  │
  ├─ Internal implementation details
  │  ├─ Full state definition
  │  └─ Complete type invariants
  │
  └─ All operations + conditions
       │
       └─ ┌─────────────────────────────────┐
           │ Publish only public contract   │
           └─────────────────────────────────┘
                      ↓
interface-contract.vdmsl
  │
  ├─ Exported types (public API)
  ├─ Exported operations + contracts only
  └─ Composition rules for multi-agent systems
```

**Key difference**:
- **module-template**: "How the module works" (implementation)
- **interface-contract**: "What other modules can rely on" (API)

### Compositional Verification Pattern

This is the heart of multi-agent development: **safe integration without reimplementation**.

#### Example: Three-Module Composition

```
Module A: OrderManagement
Module B: PaymentProcessing
Module C: Shipping
```

**Safe Sequence** (verified by contract):

```vdmsl
-- Step 1: Create order (Module A)
let oid = OrderManagement.CreateOrder(cust, items)
-- A.post guarantees: oid exists, status = PENDING

-- Step 2: Confirm order (Module A)
let confirm = OrderManagement.ConfirmOrder(oid)
-- A.post guarantees: status = CONFIRMED
-- This satisfies B.pre (requires CONFIRMED status)

-- Step 3: Process payment (Module B)
let payment = PaymentProcessing.AuthorizePayment(oid, amount)
-- B.post guarantees: payment authorized for this order
-- This satisfies C.pre (requires payment authorization)

-- Step 4: Ship order (Module C)
let shipment = Shipping.ShipOrder(oid)
-- C.post guarantees: shipment created, order status = SHIPPED
```

**Why this is safe**:

```
A.CreateOrder.post ⇒ A.ConfirmOrder.pre        ✓
A.ConfirmOrder.post ⇒ B.AuthorizePayment.pre    ✓
B.AuthorizePayment.post ⇒ C.ShipOrder.pre       ✓
```

**Unsafe Sequence** (AI agent must detect):

```vdmsl
-- ERROR: Skipping confirmation
let oid = OrderManagement.CreateOrder(cust, items)
-- status = PENDING, but...

let shipment = Shipping.ShipOrder(oid)
-- ShipOrder.pre requires status = CONFIRMED ✗
-- PRECONDITION VIOLATED: AI agent must abort
```

### AI Agent Interpretation Guide

When an AI agent encounters a VDM-SL specification, it should:

#### 1. Extract the Contract

```python
# Pseudocode: what an AI agent does
for operation in module.operations:
    precondition = operation.pre   # What must be true before
    postcondition = operation.post  # What will be true after

    # Store this contract for verification
    contracts[operation.name] = (precondition, postcondition)
```

#### 2. Verify Preconditions Before Calling

```python
def call_operation(op_name, args):
    pre, post = contracts[op_name]

    # Check: does precondition hold in current state?
    if not evaluate(pre, current_state):
        raise PreconditionViolated(op_name, pre)

    # Safe to call
    result = execute(op_name, args)

    # Verify: did postcondition hold?
    if not evaluate(post, current_state):
        raise PostconditionViolated(op_name, post)

    return result
```

#### 3. Compose Operations

```python
def compose_operations(ops: List[str], state):
    """Verify that a sequence of operations is safe to execute."""

    current_state = state
    for op in ops:
        pre, post = contracts[op]

        # Can I call this operation now?
        if not evaluate(pre, current_state):
            return False  # This sequence is unsafe

        # Execute and update state
        current_state = execute(op, current_state)

        # Verify postcondition
        if not evaluate(post, current_state):
            return False  # Implementation violates contract

    return True  # Sequence is safe
```

#### 4. Detect Integration Issues

```python
# Example: two modules in sequence
module_a_post = "entities(id).status = CONFIRMED"
module_b_pre = "entities(id).status = CONFIRMED"

# Does A's output satisfy B's input?
if module_a_post implies module_b_pre:
    # Safe to compose
    pass
else:
    # Integration issue: A doesn't guarantee what B needs
    raise IntegrationError("Module A's postcondition doesn't imply Module B's precondition")
```

### Best Practices for Writing Specs

#### 1. Make Assumptions Explicit

**WRONG** ❌
```vdmsl
operations
  ProcessOrder(order_id : OrderId)
    result : bool
  pre true
  post (RESULT = true);  -- Says nothing about what happens
```

**RIGHT** ✓
```vdmsl
operations
  ProcessOrder(order_id : OrderId)
    result : OperationResult
  pre
    order_id > 0
    and (let order = GetOrder(order_id)
         in order.status = <PENDING>)
  post
    (RESULT = <SUCCESS>) =>
      (let processed = GetOrder(order_id)
       in processed.status = <PROCESSING>)
    and (RESULT = <ERROR>) =>
      (let unchanged = GetOrder(order_id)
       in unchanged.status = <PENDING>);
```

**Why**: Other AI agents can't reason about the implicit contract.

#### 2. Use Tilde (~) for State References

```vdmsl
-- ~ means "previous state" (before operation)
post
  -- "updated_at is greater than before"
  orders(order_id).updated_at > orders~(order_id).updated_at
```

#### 3. Distinguish Pure Functions from Operations

**Pure Function** (no side effects):
```vdmsl
functions
  CalculateTotal(items : seq of LineItem) : nat ==
    sum [item.quantity * item.unit_price | item in seq items];
```

**Operation** (may modify state):
```vdmsl
operations
  CreateOrder(...) oid : OrderId
  pre ... post ...;
```

**Why**: AI agents can safely call pure functions multiple times; operations have side effects.

#### 4. Always Check State Invariants

```vdmsl
post
  -- ... operation-specific postconditions ...

  -- ALWAYS end with:
  and inv_OrderState(OrderState);  -- Invariant preserved
```

#### 5. Make Error Cases Explicit

```vdmsl
operations
  CancelOrder(order_id : OrderId)
    result : OperationResult
  pre
    order_id > 0
  post
    -- Success case
    ((order.status = <PENDING> or order.status = <CONFIRMED>) =>
      (RESULT = <SUCCESS> and GetOrder(order_id).status = <CANCELLED>))
    -- Failure case
    and ((order.status = <SHIPPED> or order.status = <DELIVERED>) =>
      (RESULT = <CANCELLED_TOO_LATE> and GetOrder(order_id) = GetOrder~(order_id)));
```

### Troubleshooting

#### Problem: "Operation X's postcondition doesn't match Operation Y's precondition"

**Cause**: Modules are not composable. For example:
- A's postcondition: `status = PENDING`
- B's precondition: `status = CONFIRMED`
- These don't match → can't call B after A

**Solution**:
1. Adjust A's postcondition to guarantee what B needs, OR
2. Insert an intermediate operation (like `ConfirmOrder`) to transition the state

#### Problem: "Invariant violated after operation"

**Cause**: Operation doesn't preserve state invariant.

**Solution**:
1. Check: Does postcondition include `and inv_ModuleState(...)`?
2. Check: Does operation correctly update all relevant fields?
3. Verify: Is the invariant actually achievable?

#### Problem: "Can't verify if Module A → B is safe"

**Cause**: A's postcondition is too weak. For example:
- A.post: `order exists` (not enough detail)
- B.pre: `order.status = CONFIRMED` (B needs this specifically)

**Solution**: Strengthen A's postcondition to include the status information B requires.

---

## 日本語

### 概要

これらのVDM-SLテンプレートは、**テックリード＆AIエージェント**が以下の特性を持つ形式仕様を作成することを可能にします：

- **明確性**: 前提条件/事後条件が暗黙の仮定を明示化する
- **合成可能性**: あるモジュールの出力が別のモジュールの入力として安全に活用できる
- **自動化可能性**: AIエージェントが正確性を検証し、統合問題を検出できる
- **実行可能性**: テンプレートをVDM-SLツール（Overture、VDMJ）で検証できる

### 含まれるもの

1. **module-template.vdmsl** - 完全なモジュール仕様：
   - 不変条件付き型定義（ドメインモデル）
   - 一貫性ルール付きの状態定義
   - 明確な前提条件/事後条件を持つ操作
   - 一般的なクエリ用のヘルパー関数
   - AIエージェント解釈用の詳細なコメント

2. **interface-contract.vdmsl** - コントラクト定義：
   - 他のモジュールに見える公開型と操作
   - 合成検証パターン
   - 証明付きの安全な操作シーケンス
   - 依存関係の期待値（他のモジュールが提供すべきもの）
   - 常に保持される状態不変条件

3. **README.md** - このガイド

### クイックスタート

#### ステップ1: ドメインを選ぶ

テンプレートは「注文管理」システムを例として使っています。あなたのドメインに置き換えます：

- Eコマース → `OrderManagement`をサービス名に置き換え
- 決済処理 → 決済ライフサイクル用に操作を適応
- 認証 → ユーザー、トークン、セッション用に型を再定義
- あらゆるステートフルシステム → 同じ構造パターンに従う

#### ステップ2: 型を定義する

`module-template.vdmsl`のセクション2（TYPE DEFINITIONS）で：

```vdmsl
types

  -- あなたのドメイン概念
  MyEntity = nat
    inv entity == entity > 0;  -- 不変条件：正のIDのみ

  -- ライフサイクル用のEnum
  MyEntityStatus = <CREATED> | <ACTIVE> | <ARCHIVED>;

  -- 複雑なレコード
  MyRecord :: id : nat
             status : MyEntityStatus
             inv mk_MyRecord(i, s) == i > 0;
```

**不変条件が重要な理由**：
- 型システムが無効なインスタンスを防ぐ
- AIエージェントは不変条件が常に保持されると仮定できる
- ランタイムチェックの必要性を削減

#### ステップ3: 状態 + 不変条件を定義する

`module-template.vdmsl`のセクション3（STATE DEFINITION）で：

```vdmsl
state MyState of
  entities : map nat to MyRecord
  next_id : nat

  inv state ==
    -- マップ内のすべてのIDは正
    (forall id in set dom entities & id > 0)
    -- next_idはユニーク
    and (forall id in set dom entities & id < next_id)
    -- 一貫性：エンティティIDはマップキーと一致
    and (forall id in set dom entities & entities(id).id = id)

  init s == s = mk_MyState({}, 1)
end
```

**なぜこれが重要か**：
- 一貫性ルールの単一の真実のソース
- すべての操作がこの不変条件を保持する必要がある
- AIエージェントは各操作の前後で`inv_MyState(state)`を検証する

#### ステップ4: コントラクト付きの操作を書く

`module-template.vdmsl`のセクション4（OPERATIONS）で：

```vdmsl
operations

  CreateEntity(name : str)
    eid : nat
  pre
    -- この操作の前に真である必要があること
    len name > 0  -- 名前は空にできない
  post
    -- この操作の後に真になること
    RESULT > 0    -- 正のIDを返す
    and (let created = GetEntity(RESULT)
         in created.status = <CREATED>);

  UpdateEntity(eid : nat, new_status : MyEntityStatus)
    result : OperationResult
  pre
    eid > 0
    and (let entity = GetEntity(eid)
         in IsValidTransition(entity.status, new_status))
  post
    (RESULT = <SUCCESS>)
    and (let updated = GetEntity(eid)
         in updated.status = new_status
         and updated.id = eid);
```

**コントラクトの読み方**：

| 要素 | 意味 | AIエージェント向け |
|------|------|------------------|
| `pre` | クライアント責任 | 「呼び出す前にこれを検証する必要がある」 |
| `post` | システム保証 | 「呼び出し後、これに依拠できる」 |
| `RESULT` | 戻り値 | 事後条件で操作の戻り値を参照 |
| `~` (チルダ) | 前の状態 | `entities~(id)` = 操作前の状態 |

### 関係：module-template ↔ interface-contract

```
module-template.vdmsl
  │
  ├─ 内部実装の詳細
  │  ├─ 完全な状態定義
  │  └─ 完全な型不変条件
  │
  └─ すべての操作 + 条件
       │
       └─ ┌─────────────────────────────────┐
           │ 公開コントラクトのみ発行        │
           └─────────────────────────────────┘
                      ↓
interface-contract.vdmsl
  │
  ├─ 公開型（公開API）
  ├─ 公開操作とコントラクトのみ
  └─ マルチエージェントシステム用の合成ルール
```

**主な違い**：
- **module-template**: 「モジュールがどのように機能するか」（実装）
- **interface-contract**: 「他のモジュールが頼れるもの」（API）

### 合成検証パターン

これはマルチエージェント開発の心臓です：**再実装なしの安全な統合**。

#### 例：3モジュール合成

```
モジュールA: OrderManagement
モジュールB: PaymentProcessing
モジュールC: Shipping
```

**安全なシーケンス**（コントラクトで検証）：

```vdmsl
-- ステップ1：注文作成（モジュールA）
let oid = OrderManagement.CreateOrder(cust, items)
-- A.postが保証：oidが存在、status = PENDING

-- ステップ2：注文確認（モジュールA）
let confirm = OrderManagement.ConfirmOrder(oid)
-- A.postが保証：status = CONFIRMED
-- これはB.preを満たす（CONFIRMED状態が必要）

-- ステップ3：支払い処理（モジュールB）
let payment = PaymentProcessing.AuthorizePayment(oid, amount)
-- B.postが保証：この注文の支払いが認可された
-- これはC.preを満たす（支払い認可が必要）

-- ステップ4：注文発送（モジュールC）
let shipment = Shipping.ShipOrder(oid)
-- C.postが保証：発送が作成された、注文status = SHIPPED
```

**これが安全な理由**：

```
A.CreateOrder.post ⇒ A.ConfirmOrder.pre        ✓
A.ConfirmOrder.post ⇒ B.AuthorizePayment.pre    ✓
B.AuthorizePayment.post ⇒ C.ShipOrder.pre       ✓
```

**危険なシーケンス**（AIエージェントが検出すべき）：

```vdmsl
-- エラー：確認をスキップ
let oid = OrderManagement.CreateOrder(cust, items)
-- status = PENDING だが...

let shipment = Shipping.ShipOrder(oid)
-- ShipOrder.preはstatus = CONFIRMED を要求 ✗
-- 前提条件違反：AIエージェントは中止する必要がある
```

### AIエージェント解釈ガイド

AIエージェントがVDM-SL仕様に遭遇したとき、それは以下をすべき：

#### 1. コントラクトを抽出する

```python
# 疑似コード：AIエージェントが行うこと
for operation in module.operations:
    precondition = operation.pre    # 前に真である必要がある
    postcondition = operation.post   # 後に真になる

    # このコントラクトを検証用に保存
    contracts[operation.name] = (precondition, postcondition)
```

#### 2. 呼び出す前に前提条件を検証する

```python
def call_operation(op_name, args):
    pre, post = contracts[op_name]

    # 現在の状態で前提条件が保持されているか？
    if not evaluate(pre, current_state):
        raise PreconditionViolated(op_name, pre)

    # 呼び出すのは安全
    result = execute(op_name, args)

    # 検証：事後条件は保持されたか？
    if not evaluate(post, current_state):
        raise PostconditionViolated(op_name, post)

    return result
```

#### 3. 操作を合成する

```python
def compose_operations(ops: List[str], state):
    """操作のシーケンスが安全に実行可能か検証する。"""

    current_state = state
    for op in ops:
        pre, post = contracts[op]

        # 今この操作を呼び出すことができるか？
        if not evaluate(pre, current_state):
            return False  # このシーケンスは危険

        # 実行して状態を更新
        current_state = execute(op, current_state)

        # 事後条件を検証
        if not evaluate(post, current_state):
            return False  # 実装がコントラクトに違反している

    return True  # シーケンスは安全
```

#### 4. 統合問題を検出する

```python
# 例：2つのモジュールを順番に
module_a_post = "entities(id).status = CONFIRMED"
module_b_pre = "entities(id).status = CONFIRMED"

# Aの出力はBの入力を満たすか？
if module_a_post implies module_b_pre:
    # 合成は安全
    pass
else:
    # 統合問題：AはBが必要とすることを保証していない
    raise IntegrationError("モジュールAの事後条件がモジュールBの前提条件を含意していない")
```

### 仕様作成のベストプラクティス

#### 1. 仮定を明示的にする

**悪い例** ❌
```vdmsl
operations
  ProcessOrder(order_id : OrderId)
    result : bool
  pre true
  post (RESULT = true);  -- 何が起こるかについて何も言わない
```

**良い例** ✓
```vdmsl
operations
  ProcessOrder(order_id : OrderId)
    result : OperationResult
  pre
    order_id > 0
    and (let order = GetOrder(order_id)
         in order.status = <PENDING>)
  post
    (RESULT = <SUCCESS>) =>
      (let processed = GetOrder(order_id)
       in processed.status = <PROCESSING>)
    and (RESULT = <ERROR>) =>
      (let unchanged = GetOrder(order_id)
       in unchanged.status = <PENDING>);
```

**理由**: 他のAIエージェントが暗黙のコントラクトについて推論できません。

#### 2. 状態参照にはチルダ（~）を使用する

```vdmsl
-- ~ は「前の状態」を意味する（操作前）
post
  -- 「updated_atが前より大きい」
  orders(order_id).updated_at > orders~(order_id).updated_at
```

#### 3. 純粋関数と操作を区別する

**純粋関数**（副作用なし）：
```vdmsl
functions
  CalculateTotal(items : seq of LineItem) : nat ==
    sum [item.quantity * item.unit_price | item in seq items];
```

**操作**（状態を変更できる）：
```vdmsl
operations
  CreateOrder(...) oid : OrderId
  pre ... post ...;
```

**理由**: AIエージェントは純粋関数を複数回安全に呼び出すことができます。操作は副作用があります。

#### 4. 常に状態不変条件をチェックする

```vdmsl
post
  -- ... 操作固有の事後条件 ...

  -- 常に終わりに：
  and inv_OrderState(OrderState);  -- 不変条件が保持される
```

#### 5. エラーケースを明示的にする

```vdmsl
operations
  CancelOrder(order_id : OrderId)
    result : OperationResult
  pre
    order_id > 0
  post
    -- 成功ケース
    ((order.status = <PENDING> or order.status = <CONFIRMED>) =>
      (RESULT = <SUCCESS> and GetOrder(order_id).status = <CANCELLED>))
    -- 失敗ケース
    and ((order.status = <SHIPPED> or order.status = <DELIVERED>) =>
      (RESULT = <CANCELLED_TOO_LATE> and GetOrder(order_id) = GetOrder~(order_id)));
```

### トラブルシューティング

#### 問題：「操作Xの事後条件が操作Yの前提条件と一致しない」

**原因**: モジュールが合成可能でない。例えば：
- Aの事後条件：`status = PENDING`
- Bの前提条件：`status = CONFIRMED`
- これらは一致しない → A後にBを呼び出せない

**解決策**：
1. Aの事後条件を調整してBが必要とするものを保証するか、または
2. 中間操作を挿入してステータスを遷移させる（例：`ConfirmOrder`）

#### 問題：「操作後に不変条件が違反されている」

**原因**: 操作が状態不変条件を保持していない。

**解決策**：
1. 確認：事後条件に`and inv_ModuleState(...)`が含まれているか？
2. 確認：操作は関連するすべてのフィールドを正しく更新しているか？
3. 検証：不変条件は実際に達成可能か？

#### 問題：「モジュールA → Bが安全か検証できない」

**原因**: Aの事後条件が弱すぎる。例えば：
- A.post：`order exists`（詳細不足）
- B.pre：`order.status = CONFIRMED`（Bはこれを具体的に必要）

**解決策**: Aの事後条件を強化してBが要求するステータス情報を含める。

---

## Summary

These templates provide a structured approach to writing formal specifications that enable:

- **Clear communication** between tech leads and AI agents
- **Verifiable composition** of multi-agent systems
- **Executable contracts** that can be validated against actual implementations
- **Reusable patterns** for any stateful system

Use them as a foundation, adapt them to your domain, and leverage the formal contracts to enable safe, composable AI-driven development.

**Next Steps**:
1. Copy `module-template.vdmsl` and adapt it to your domain
2. Extract the public contract into `interface-contract.vdmsl`
3. Have AI agents verify pre/post conditions during implementation and integration
4. Compose operations only when postconditions imply preconditions
