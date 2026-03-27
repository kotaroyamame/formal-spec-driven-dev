# Contributing to Formal Spec-Driven Development

[English](#english) | [日本語](#japanese)

---

## English

Thank you for your interest in contributing to the Formal Spec-Driven Development project! This project demonstrates how to leverage AI-assisted formal specification with VDM-SL to build reliable software systems. We welcome contributions from everyone, regardless of experience level.

### How to Contribute

We appreciate contributions in the following areas:

#### Documentation
- Improve existing guides, README files, and tutorials
- Add clarifications to specifications and examples
- Write blog posts or articles about formal spec-driven development
- Create quick-start guides for new users

#### Templates and Examples
- Develop domain-specific VDM-SL templates (e.g., healthcare, fintech, logistics, IoT, e-commerce)
- Create new prompt templates for different phases of development
- Build example projects demonstrating best practices
- Add case studies showing real-world applications

#### Translations
- Translate documentation into additional languages
- Localize examples and guides for different regions
- Translate VDM-SL comments and specifications

#### Tooling and Integrations
- Integrate with Overture Tool for automated verification
- Create plugins for popular IDEs and editors
- Build CI/CD pipeline examples
- Develop helper scripts for common workflows
- Create integrations with orchestration tools (Kubernetes, Docker, etc.)

#### Code of Conduct

This project is committed to providing a welcoming and inclusive environment for all contributors. We expect all participants to:

- **Be respectful** and considerate in all interactions
- **Be inclusive** of diverse backgrounds, experiences, and perspectives
- **Be collaborative** and constructive in feedback
- **Focus on the work**, not personal attributes
- **Report issues** of harassment or misconduct promptly

Instances of abusive, harassing, or otherwise unacceptable behavior may be reported to the project maintainers. All complaints will be reviewed and investigated fairly.

### Issue and Pull Request Workflow

#### Reporting Issues

1. **Check existing issues** before opening a new one to avoid duplicates
2. **Use issue templates** (bug report, feature request, case study)
3. **Provide clear context**: What were you trying to do? What happened? What did you expect?
4. **Include examples** where applicable (code snippets, specifications, error messages)
5. **Label appropriately**: bug, enhancement, documentation, template-request, translation, etc.

#### Creating Pull Requests

1. **Fork the repository** and create a descriptive branch name:
   - `feature/new-template-healthcare`
   - `docs/improve-getting-started`
   - `fix/vdm-sl-syntax-example`
   - `translation/french-documentation`

2. **Write clear commit messages**:
   - First line: concise summary (50 characters max)
   - Blank line
   - Detailed explanation if needed

3. **Follow the style guidelines** (see below)

4. **Update documentation** if your changes affect how users work with the project

5. **Submit your PR** with a clear description of what you've changed and why

### Style Guidelines

#### VDM-SL Specifications

- **Naming conventions**: Use descriptive names in CamelCase for types and values
  ```vdml
  types
    PatientRecord :: id: nat
                    name: String
                    age: nat
  end
  ```

- **Comments**: Include comments explaining the business logic and invariants
  ```vdml
  -- Ensure age is within reasonable bounds
  inv mk_PatientRecord(id, name, age) == age <= 150
  ```

- **Formatting**: Use consistent indentation (2 spaces), align operators clearly
- **Documentation**: Include a brief specification header explaining the module's purpose

#### Prompt Templates

- **Clear structure**: Separate phases logically with distinct sections
- **Context first**: Provide background before asking for work
- **Concrete examples**: Include sample input/output when helpful
- **Constraints**: Explicitly state any constraints or requirements
- **Validation**: Ask for confirmation of critical decisions

#### Documentation

- **Plain language**: Write for an audience with formal methods background but new to this approach
- **Concrete examples**: Show actual VDM-SL code and prompts
- **Consistent formatting**: Use headings, code blocks, and lists appropriately
- **Links**: Cross-reference related sections and templates
- **Bilingual content**: When translating, maintain parallel structure in both languages

### Areas Where Help is Especially Wanted

#### 1. Domain-Specific VDM-SL Templates

We're seeking templates for these industries:
- **Healthcare**: Patient records, medication management, appointment scheduling
- **Fintech**: Transaction processing, account management, regulatory compliance
- **Logistics**: Inventory management, route optimization, shipment tracking
- **E-Commerce**: Shopping cart, order fulfillment, payment processing
- **IoT/Embedded Systems**: Sensor data processing, device state management
- **Telecommunications**: Call routing, billing systems, network management

#### 2. Orchestration Tool Integrations

Help us integrate with:
- **Kubernetes**: Deployment specifications, StatefulSet examples
- **Docker**: Container orchestration patterns
- **Terraform**: Infrastructure as code examples
- **CI/CD Pipelines**: GitHub Actions, GitLab CI, Jenkins workflows

#### 3. Case Studies

We welcome:
- Real project experiences using formal specs
- Lessons learned from formal specification projects
- Performance and reliability metrics
- Team size and timeline information
- Before/after comparisons with traditional approaches

Please submit case studies using the case-study issue template.

#### 4. Translations

Current support needed for:
- French
- German
- Spanish
- Mandarin Chinese
- Other languages as interest develops

#### 5. Tooling Improvements

We're looking for contributions in:
- **Overture Tool integration**: Automated specification verification
- **IDE extensions**: VS Code, JetBrains, Vim/Neovim plugins
- **Linting tools**: VDM-SL style checkers
- **Documentation generators**: Auto-generate specs from VDM-SL code
- **Test utilities**: Helper functions for property-based testing

### Getting Started

1. **Read the main README** to understand the project philosophy
2. **Explore the templates** directory to see existing examples
3. **Check open issues** tagged "good-first-issue" or "help-wanted"
4. **Comment on an issue** before starting work to ensure alignment
5. **Join discussions** about the project's direction and priorities

### Questions?

- Open a discussion on GitHub if you have questions
- Check existing documentation and examples
- Look at similar pull requests for patterns
- Don't hesitate to ask for clarification in an issue

Thank you for contributing to better software development practices!

---

## 日本語

Formal Spec-Driven Development プロジェクトへのご貢献に感謝します！このプロジェクトは、AI支援形式仕様とVDM-SLを活用して信頼性の高いソフトウェアシステムを構築する方法を示しています。経験レベルに関わらず、誰からの貢献も歓迎します。

### 貢献方法

以下の分野での貢献をお待ちしています：

#### ドキュメント
- 既存ガイド、READMEファイル、チュートリアルの改善
- 仕様とサンプルの説明追加
- 形式仕様駆動開発に関するブログ記事の執筆
- 新規ユーザー向けクイックスタートガイドの作成

#### テンプレートとサンプル
- ドメイン固有のVDM-SLテンプレート開発（医療、金融、ロジスティクスなど）
- 開発フェーズ別の新しいプロンプトテンプレート作成
- ベストプラクティスを示すサンプルプロジェクト
- 実世界での応用例を示すケーススタディ追加

#### 翻訳
- ドキュメントの追加言語への翻訳
- サンプルとガイドの地域ローカライズ
- VDM-SLコメントと仕様の翻訳

#### ツール連携とintegrations
- Overture Toolとの統合による自動検証
- 人気IDEおよびエディタへのプラグイン開発
- CI/CDパイプラインの例
- 一般的なワークフロー用ヘルパースクリプト開発
- オーケストレーションツール（Kubernetes、Dockerなど）との統合

#### 行動規範

このプロジェクトは、全ての貢献者に対して歓迎的で包括的な環境を提供することを約束しています。全ての参加者は以下を守ることを期待しています：

- **尊重と思慮深さ**をもつ対話
- **多様な背景、経験、観点の包括**
- **建設的で協力的な**フィードバック
- **仕事に焦点**を当てた議論
- **問題の迅速な報告**

虐待的、嫌がらせ的、または受け入れられない行動が報告された場合、プロジェクト保守者によって公正に調査されます。

### Issue および Pull Request ワークフロー

#### Issue報告

1. **既存のissueを確認**して重複を避ける
2. **issue テンプレートを使用**（バグ報告、機能リクエスト、ケーススタディ）
3. **明確なコンテキスト提供**：何をしようとしていた？何が起きた？期待していた動作は？
4. **例を含める**：コードスニペット、仕様、エラーメッセージなど
5. **適切にラベル付け**：bug、enhancement、documentation、template-request、translation など

#### Pull Requestの作成

1. **リポジトリをフォーク**して説明的なブランチ名を作成：
   - `feature/new-template-healthcare`
   - `docs/improve-getting-started`
   - `fix/vdm-sl-syntax-example`
   - `translation/french-documentation`

2. **明確なコミットメッセージを作成**：
   - 最初の行：簡潔なサマリー（50文字以内）
   - 空行
   - 必要に応じて詳細説明

3. **スタイルガイドラインに従う**（下記参照）

4. **変更がユーザーの使用方法に影響する場合、ドキュメントを更新**

5. **明確な説明を含むPRを提出**

### スタイルガイドライン

#### VDM-SL仕様

- **命名規則**：CamelCase でわかりやすい名前を使用
  ```vdml
  types
    PatientRecord :: id: nat
                    name: String
                    age: nat
  end
  ```

- **コメント**：ビジネスロジックと不変量を説明するコメントを含める
  ```vdml
  -- 年齢が妥当な範囲内であることを確認
  inv mk_PatientRecord(id, name, age) == age <= 150
  ```

- **フォーマット**：一貫したインデント（2スペース）、演算子の整列
- **ドキュメント**：モジュールの目的を説明する簡潔なヘッダーを含める

#### プロンプトテンプレート

- **明確な構造**：フェーズを論理的に区切る
- **コンテキスト優先**：作業を依頼する前に背景を提供
- **具体例**：入出力例を含める
- **制約の明示**：制約や要件を明確に記述
- **確認**：重要な決定確認を求める

#### ドキュメント

- **平易な言語**：形式手法の背景を持ちながら、このアプローチに新しいオーディエンスを想定
- **具体例**：実際のVDM-SLコードとプロンプトを示す
- **一貫したフォーマット**：見出し、コードブロック、リストを適切に使用
- **リンク**：関連セクションとテンプレートへのクロスリファレンス
- **バイリンガルコンテンツ**：翻訳時、両言語で平行構造を維持

### 特に支援が必要な分野

#### 1. ドメイン固有VDM-SLテンプレート

以下の業界向けテンプレートを募集中：
- **ヘルスケア**：患者記録、投薬管理、予約スケジューリング
- **金融技術**：トランザクション処理、アカウント管理、規制準拠
- **ロジスティクス**：在庫管理、ルート最適化、出荷追跡
- **Eコマース**：ショッピングカート、注文処理、決済処理
- **IoT/組込システム**：センサーデータ処理、デバイス状態管理
- **通信**：コール ルーティング、課金システム、ネットワーク管理

#### 2. オーケストレーションツール統合

以下との統合に支援が必要：
- **Kubernetes**：デプロイメント仕様、StatefulSet例
- **Docker**：コンテナオーケストレーションパターン
- **Terraform**：Infrastructure as Code例
- **CI/CDパイプライン**：GitHub Actions、GitLab CI、Jenkins ワークフロー

#### 3. ケーススタディ

以下を募集中：
- 形式仕様を使用した実プロジェクト経験
- 形式仕様プロジェクトから得た教訓
- パフォーマンスと信頼性指標
- チーム規模とタイムライン情報
- 従来アプローチとの比較

ケーススタディテンプレートを使用して提出してください。

#### 4. 翻訳

現在以下の言語でのサポートが必要：
- フランス語
- ドイツ語
- スペイン語
- 中国語（標準中国語）
- その他の言語

#### 5. ツール改善

以下の貢献を求めています：
- **Overture Tool統合**：仕様の自動検証
- **IDE拡張**：VS Code、JetBrains、Vim/Neovim プラグイン
- **リントツール**：VDM-SLスタイルチェッカー
- **ドキュメント生成**：VDM-SLコードから自動生成
- **テストユーティリティ**：プロパティベースのテスト用ヘルパー関数

### はじめ方

1. **メインREADMEを読む**：プロジェクト哲学の理解
2. **テンプレートディレクトリを参照**：既存例の確認
3. **Openissueを確認**："good-first-issue" または "help-wanted" タグを探す
4. **作業前にissueにコメント**：方向性確認
5. **ディスカッションに参加**：プロジェクト方向性と優先事項について

### 質問がある場合

- GitHub でディスカッションを開く
- 既存ドキュメントと例を確認
- 類似の pull request からパターンを学ぶ
- issue内で遠慮なく説明を求める

より良いソフトウェア開発慣行に貢献してくれてありがとうございます！
