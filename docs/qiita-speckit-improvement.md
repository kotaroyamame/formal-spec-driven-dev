---
title: "GitHub Spec Kitは80%正しい——残り20%を埋める「形式仕様」という提案"
tags: AI駆動開発 GitHub SpecKit 形式手法 ソフトウェアアーキテクチャ
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## Spec Kitの思想は正しい。だからこそ、もう一歩踏み込みたい。

2025年にGitHubがオープンソース化した[Spec Kit](https://github.com/github/spec-kit)は、AI駆動開発ツールの中で最も知的に誠実なアプローチだと考えています。多くのツールが「コード生成の高速化」に注力する中、Spec Kitはもっと根本的な問いを立てました。

> "We're moving from 'code is the source of truth' to 'intent is the source of truth.'"
> （「コードが真実の源泉」から「意図が真実の源泉」へ）

これはまさに正しい問いです。Constitution → Specify → Plan → Tasks → Implement のワークフロー、各ゲートで人間が介入可能な「ステアラブル」設計、25以上のAIエージェント対応（Claude Code、Copilot、Gemini CLI、Cursor等）、40以上のコミュニティ拡張——いずれも実用的でよく設計されています。

私は[形式仕様駆動開発フレームワーク](https://github.com/kotaroyamame/formal-spec-driven-dev)を構築する中で、同じ出発点——**仕様が開発を駆動すべきであり、その逆ではない**——に立っています。しかしリサーチと実装を重ねるうちに、Spec Kitには一つの構造的ギャップがあると確信するに至りました。

そのギャップとは、**仕様自体の形式的検証可能性**です。

## 問題：「構造化された曖昧性」

Spec Kitの仕様は構造化されたMarkdownで記述されます。これは他の手法と比較して大きな前進です。

| 手法 | 正しさの根拠 | 構造的欠陥 |
|:---|:---|:---|
| MetaGPT | SOP＋自然言語PRD | エージェント間で曖昧性が蓄積 |
| ChatDev | Chat Chain対話合意 | 非決定的・再現不可能 |
| Devin/SWE-Agent | 既存コードの挙動 | コードから仕様を推測する循環論法 |
| **Spec Kit** | 構造化Markdown仕様 | 曖昧性を低減するが**排除しない** |

参照元：各手法の詳細な構造的欠陥分析は[比較分析ドキュメント](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/master/docs/comparison.md)を参照。

Spec Kitは上記の他手法の欠陥をすべて回避しています。しかし、**構造化された自然言語は曖昧性を低減するものの、排除はしません**。

具体例を見てみましょう。Spec Kitの仕様として以下を書いたとします。

```markdown
## ユーザーストーリー：カートに追加
ユーザーは商品をショッピングカートに追加できる。
カートには更新された数量が反映される。
```

人間の読者にとっては明快です。しかし、いくつかの重要な問いが未回答のまま残ります。

- 0個や負の数を追加できるか？
- 在庫不足の場合はどうなるか？
- 既にカートにある商品の場合、数量は**置換**か**加算**か？
- カートの上限はあるか？

これらはエッジケースではなく**境界条件**——モジュールが接合し、バグが潜む場所——です。仕様の記述者が意識的に列挙しない限り、仕様に含まれません。

## 形式仕様が加えるもの

同じ要件を[VDM-SL](https://github.com/kotaroyamame/formal-spec-driven-dev/tree/main/templates/vdm-sl)（Vienna Development Method - Specification Language）で記述すると、こうなります。

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
     and carts(cid)(pid) = carts~(cid)(pid) + qty  -- 加算（置換ではない）
```

何が起きているかを整理します。

1. **`nat1`型（正の自然数）** — 0や負の数量は型レベルで不可能。テストで弾くのではなく、構造的に排除される。
2. **`pre`条件** — 在庫十分性、カートサイズ上限が**明示的な事前条件**。AIエージェントがこれを「忘れる」ことは不可能。
3. **`post`条件** — 振る舞いが一義的：数量は加算される（置換ではない）。どのエージェントが実装しても同じ結果になる。
4. **機械的検証** — 別のモジュールの事後条件が `inventory(pid) >= qty` を保証していれば、このモジュールの事前条件を満たすことを**機械的に証明**できる。人間のレビューは不要。

## Spec Kitに形式仕様が埋める3つのギャップ

### ギャップ1：境界条件が自然言語では不可視

Spec Kitでは、境界条件の網羅性は仕様記述者の注意力に依存します。これは人間の信頼性問題——まさにシステムから排除しようとしている問題——です。

形式仕様では、型システムと事前条件が境界条件を構造的に強制します。`qty: nat`と書いて在庫制約の事前条件を書かなければ、**仕様が不完全であることが型検査で可視化**されます。

### ギャップ2：モジュール間整合性が検証不可能

これはスケーリングにおける決定的問題です。Spec Kitはタスク（≒モジュール）単位で仕様を記述しますが、**タスクAの出力がタスクBの入力要件を満たすか**を機械的に検証する手段がありません。

| モジュール数 | ペアワイズIF数 | 人間レビューの実現性 |
|:---:|:---:|:---|
| 3 | 3 | 容易 |
| 5 | 10 | 可能 |
| 10 | 45 | 困難 |
| 20 | 190 | 非現実的 |

形式仕様では**合成可能性の検証**でこれを解決します。OrderモジュールのConfirmOrderの事後条件と、InventoryモジュールのReserveStockの事前条件がある場合、`ConfirmOrder.post ⇒ ReserveStock.pre` を機械的に検査できます。検証コストはO(n²)ではなくO(n)で、モジュール数に対して線形に増加します。

```
Module A (Order)                Module B (Inventory)
┌─────────────────┐            ┌─────────────────┐
│  ConfirmOrder()  │            │  ReserveStock()  │
│  post:           │───検証────│  pre:            │
│   order.status   │            │   productId ∈    │
│    = <CONFIRMED> │   A.post   │    dom inventory │
│   ∧ stock        │    ⇒       │   ∧ quantity > 0 │
│    reserved      │   B.pre    │   ∧ available ≥  │
│                  │            │     quantity     │
└─────────────────┘            └─────────────────┘
```

### ギャップ3：仕様自体の内部矛盾を検出できない

Spec Kitの哲学は「意図が真実の源泉」です。しかしメタレベルの問題があります。**仕様自体が内部的に矛盾していないことをどう検証するか？**

自然言語仕様は、実装段階になるまで矛盾が発見されないケースがあります。

```markdown
- チェックアウトは30分以内に完了すること
- カートを保存して後日再開できること
- チェックアウト開始時に在庫を確保すること
```

これらは矛盾していないでしょうか？ユーザーが29分目にカートを保存し、3時間後に再開した場合、在庫は確保されたままか？自然言語ではこの矛盾は目視でしか発見できません。VDM-SLでは、在庫確保期間の不変条件と「後日再開」の事後条件が型検査レベルで衝突します。

## 具体的提案：Spec Kitのワークフローに形式仕様層を追加

Spec Kitの自然言語層を置換するのではなく、**その下に形式検証層を追加**する提案です。

### Constitutionフェーズ（変更なし）
プロジェクト原則・ガイドラインは自然言語のまま。ガバナンスであり計算対象ではない。

### Specifyフェーズ（拡張）
人間が自然言語で意図を記述する（従来通り）。続けて、AIエージェントが**VDM-SL形式化を生成**し、人間に説明する。

> 「あなたの仕様では、ユーザーがカートに商品を追加できます。以下の制約を形式化しました。数量は正の整数、在庫が十分であること、カート上限は50個。数量の振る舞いは加算——カートに既に2個あり3個追加すると5個になります（3個に置き換わるのではなく）。この理解で正しいですか？」

人間が意味を検証し、機械が整合性を検証する。

### Planフェーズ（拡張）
技術計画にモジュール間の**インターフェース契約**を含める。どのモジュールの事後条件がどのモジュールの事前条件に接続するか。これらの契約はVDM-SLで記述され、機械的に検証可能。

### Tasksフェーズ（拡張）
各タスクに自然言語の説明だけでなく、**形式モジュール仕様**——型・状態・操作・事前/事後条件——を付与。実装を担当するAIエージェントは一義的な契約を持つ。

### Implementフェーズ（変更なし）
エージェントは自然言語ではなく形式仕様に基づいて実装。Spec Kitが対応する25以上のエージェントはいずれもVDM-SLを読み取り可能。

### 新規：Verifyフェーズ
専用の検証ステップで全モジュール境界の `A.post ⇒ B.pre` を検査。**仕様レベル**で、コード実行前に実施。統合問題を実装前に発見。

## なぜ今これが重要か——マルチエージェント時代

単一エージェント開発（1人の開発者＋1つのAI）であれば、自然言語仕様でも問題は少ない。人間がリアルタイムで曖昧性をキャッチできるからです。

しかし業界は明らかに**マルチエージェント開発**——複数のAIエージェントが異なるモジュールを並列構築する——へ向かっています。マルチエージェント環境では、自然言語の曖昧性はスケーリング危機になります。

形式仕様のインターフェース契約は、スケーリングに耐えうる唯一の既知のメカニズムです。各契約が独立に検証されるため、検証コストはモジュール数に対して線形に増加します。

## この提案が意味しないこと

明確にしておきます。

- **「Spec Kitが悪い」ではない。** Spec Kitは現存する仕様駆動フレームワークの中で最も良く設計されています。思想もUXもコミュニティも正しい。
- **「全員がVDM-SLを学ぶべき」ではない。** AIが形式記法を読み書きし、人間は自然言語で意味を検証する。形式手法の専門知識は不要。
- **「形式仕様がテストを置換する」ではない。** 統合テスト・性能テスト・E2Eテストは引き続き重要。形式仕様が置換するのは、正しさの**中心**としてのテストの地位。
- **「すべてに適用可能」ではない。** UI/UX、パフォーマンスチューニング、軽量バグ修正には過剰。対象は**モジュール間境界の正しさが重要なビジネスロジックシステム**。

## オープンソースプロジェクト

この提案の思想と実践テンプレートをオープンソースで公開しています。

**[`formal-spec-driven-dev`](https://github.com/kotaroyamame/formal-spec-driven-dev)** — Apache 2.0ライセンス

VDM-SLテンプレート、4フェーズ全体のAIプロンプトテンプレート、マルチエージェントオーケストレーション設定、ECサイトの実装例（注文・在庫・決済モジュール間の形式的インターフェース契約付き）を含みます。

MetaGPT、ChatDev、Devin、Spec Kit、Claude Code等との[体系的比較分析](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/master/docs/comparison.md)では、各手法の「正しさの根拠」の構造的欠陥を統一的フレームワークで分析しています。

Spec Kitを現在使用中で、マルチエージェント・マルチモジュールプロジェクトへのスケーリングを検討されている方は、ぜひフィードバックをお寄せください。Spec Kitの優れた開発者体験と形式検証の数学的保証を組み合わせることが、エコシステムが必要とする次のステップだと考えています。

---

*安藤光太郎 — [IID Systems](https://iid.systems)*
*[GitHub: formal-spec-driven-dev](https://github.com/kotaroyamame/formal-spec-driven-dev) | Apache 2.0*

---

## 参照元

- [GitHub Spec Kit](https://github.com/github/spec-kit) — 本記事が改善を提案するツールキット
- [Spec-Driven Development with AI（GitHub Blog）](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
- [Diving Into Spec-Driven Development（Microsoft Developer Blog）](https://developer.microsoft.com/blog/spec-driven-development-spec-kit)
- [formal-spec-driven-dev](https://github.com/kotaroyamame/formal-spec-driven-dev) — 形式仕様駆動開発フレームワーク
- [AI駆動開発アーキテクチャの比較分析](https://github.com/kotaroyamame/formal-spec-driven-dev/blob/master/docs/comparison.md) — 各手法の構造的欠陥の統一的分析
- Dijkstra, E. W. (1969). "Notes on Structured Programming" — 「テストはバグの存在を示せるがバグの不在を示せない」
