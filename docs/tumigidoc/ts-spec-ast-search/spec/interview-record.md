# ts-spec-ast-search ヒアリング記録

**作成日**: 2026-05-19
**ヒアリング実施**: step4 既存情報ベースの差分ヒアリング
**実施方法**: AskUserQuestion ツールによる対話的選択肢提示

## ヒアリング目的

たたき台（`docs/tatakidai/requirements-ast.md`）および参考案（`docs/tatakidai/chatgpt-idea-ts-object.md`）の内容を確認し、tumigi リポジトリへの導入スコープ・優先順位・既存資産との関係を明確化するためのヒアリングを実施した。

## 質問と回答

### Q1: Markdown と TypeScript spec の関係

**質問日時**: 2026-05-19
**カテゴリ**: 既存設計確認・追加要件
**背景**: たたき台では「Markdown ではなく TypeScript object」とあるが、既存の `commands/*.md` を破棄するか並走させるかで設計が大きく変わるため。

**選択肢と回答**:
- ✅ **TS spec を「指し示す」層として併存**
- ⬜ Markdown をすべて TS object に完全置換
- ⬜ 段階的移行

**信頼性への影響**:
- REQ-401（破壊的変更禁止）の信頼性が 🟡 → 🔵 に向上
- 「Markdown 全文置換」シナリオを設計上排除

---

### Q2: AST 検索ツールの選定

**質問日時**: 2026-05-19
**カテゴリ**: 追加要件
**背景**: たたき台では ast-grep と cocoindex-code が候補として挙がっていたが、どれをサポートするかが要件文に直結するため。

**選択肢と回答**（複数選択）:
- ✅ **ast-grep**
- ✅ **cocoindex-code (ccc search)**
- ✅ **TS 型 System のシグネチャ検索**

**信頼性への影響**:
- REQ-101, REQ-102, REQ-103 が新規追加で 🔵
- 3 種類のツール全てがサポート対象として確定

---

### Q3: TS 化スコープ

**質問日時**: 2026-05-19
**カテゴリ**: スコープ確定
**背景**: chatgpt-idea-ts-object.md ではスキーマが Requirement/Workflow/Task/Decision/Verification の 5 種類あるが、すべてを spec 化するか確認するため。

**選択肢と回答**（複数選択）:
- ✅ **Requirement**
- ✅ **Workflow**
- ✅ **Task**
- ✅ **Decision/Verification**

**信頼性への影響**:
- REQ-001 が 🔵 で確定（5 種類すべて）

---

### Q3-A: 検査スクリプトのスコープ

**質問日時**: 2026-05-19
**カテゴリ**: 追加要件
**背景**: spec ファイルがあっても検査されなければ整合性が崩れるため、検査の範囲を確認した。

**選択肢と回答**:
- ✅ **フル検査（推奨）**: パス実在・frontmatter・ドキュメント整合性・プラグイン互換
- ⬜ 最小限
- ⬜ 今回は見送り

**信頼性への影響**:
- REQ-104, EDGE-001, EDGE-002, EDGE-003 を新規追加で 🔵

---

### Q3-B: spec チェックの自動実行

**質問日時**: 2026-05-19
**カテゴリ**: 運用方針
**背景**: 検査の実行タイミングが要件レベルか付帯機能かを確認するため。

**選択肢と回答**:
- ⬜ pre-commit + GitHub Actions 両方
- ⬜ GitHub Actions のみ
- ✅ **手動で pnpm spec:check を叩く運用**

**信頼性への影響**:
- REQ-304（pre-commit 組み込み）を Could Have に降格 🟡
- REQ-302（CI 自動実行）を Should Have に分類 🔵

---

### Q5: ID 体系

**質問日時**: 2026-05-19
**カテゴリ**: 設計方針
**背景**: ID 命名が検索効率と整合性検査の両方に影響するため。

**選択肢と回答**:
- ✅ **tatakidai 案を採用（推奨）**: `REQ-CMD-KAIRO-001` 形式の area プレフィックス付き
- ⬜ フラットな REQ-NNN
- ⬜ オープンにして後で詳細設計

**信頼性への影響**:
- REQ-003（ID 体系）の信頼性が 🟡 → 🔵 に向上

---

### Q6: 参照アーティファクト（再質問）

**質問日時**: 2026-05-19
**カテゴリ**: スコープ確定
**背景**: 当初の質問が曖昧で `docs/book/**` のみ選択された。「TS spec が指し示す対象」と「TS spec 自体に記述する対象」を区別して聞き直し。

**選択肢と回答**（複数選択）:
- ✅ **commands/*.md（すべてのスラッシュコマンド）**
- ✅ **skills/**\/SKILL.md（dev-*, ipa-* 等）**
- ✅ **docs/** ドキュメント**
- ⬜ .claude-plugin / agents / 出力ディレクトリ

**信頼性への影響**:
- REQ-005（参照対象）が確定 🔵
- agents/ は現状ファイルなしのため除外

---

### Q7: 保守ツール

**質問日時**: 2026-05-19
**カテゴリ**: 追加要件
**背景**: spec の初期生成・継続保守の工数が要件の優先順位に影響するため。

**選択肢と回答**:
- ⬜ 人手で書き、コマンド追加時に spec も追加
- ✅ **既存 Markdown から TS spec を生成するスクリプト**
- ⬜ 両方をサポート

**信頼性への影響**:
- REQ-301（生成スクリプト）が Should Have として確定 🔵

---

### Q8: rulesync 互換性

**質問日時**: 2026-05-19
**カテゴリ**: 影響範囲確認
**背景**: tumigi は rulesync 経由で他ツールにも展開されているため、TS spec 導入が他ツール向け生成に影響しないか確認。

**選択肢と回答**:
- ✅ **現状の仕組みを維持し、TS spec は追加レイヤとして動作**
- ⬜ rulesync 向けに spec から commands/*.md を生成
- ⬜ 今回はスコープ外

**信頼性への影響**:
- REQ-402（rulesync 非干渉）が 🔵 で確定
- 「spec を Single Source of Truth に昇格」シナリオを排除

---

### Q9: 優先順位（Must / Should）

**質問日時**: 2026-05-19
**カテゴリ**: 優先順位
**背景**: フル機能開発スコープのなかで、必須実装と任意実装を区分するため。

**Must Have（複数選択）**:
- ✅ TS spec スキーマと defineXxx 関数の定義
- ✅ 代表コマンド・スキルの TS spec 記述
- ✅ ast-grep / cocoindex-code の検索手順ドキュメント
- ✅ 検索サンプル（コマンド集）の提供

**Should Have（複数選択）**:
- ✅ 既存 Markdown から TS spec を生成するスクリプト
- ✅ pnpm spec:check スクリプト（フル検査）
- ✅ spec チェックを CI で自動実行
- ✅ Decision / Verification 記述と ADR 体裁の整備

**信頼性への影響**:
- 全要件の優先度分類が確定
- requirements.md の優先順位サマリーが構築可能に

---

## ヒアリング結果サマリー

### 確認できた事項

1. **設計方針**: TS spec は既存 Markdown を「指し示すメタデータ層」として並走（破壊的変更なし）
2. **対象範囲**: commands/*.md + skills/**\/SKILL.md + docs/** ドキュメント
3. **検索手段**: ast-grep + cocoindex-code + TS 型 System の 3 系統すべてサポート
4. **TS object 種別**: Requirement / Workflow / Task / Decision / Verification の 5 種類
5. **ID 体系**: `REQ-CMD-KAIRO-001` のような area プレフィックス付き
6. **検査方針**: フル検査スクリプトを Should Have、手動運用を基本（CI 自動化は Should Have）
7. **rulesync**: 現状維持・spec は追加レイヤのみ

### 追加/変更要件

- AST 検索の代表クエリ集（最低 5 種類）を README/MANUAL に記載（NFR-202）
- 既存 Markdown → TS spec 生成スクリプトの提供（REQ-301）
- Decision / Verification テンプレートの提供（REQ-303）

### 残課題（kairo-design 以降で詳細化）

- TS プロジェクト構成（tsconfig.json、package.json scripts、依存関係）
- 生成スクリプトの実装言語・利用ライブラリ（node:fs + gray-matter / Bun / tsx 等）
- spec:check スクリプトの具体的なチェック項目の優先順位
- ast-grep / ccc search のセットアップ手順（環境別: macOS / Linux / Windows）
- CI 設定の具体（GitHub Actions のワークフロー定義）
- 既存 Markdown → TS spec 生成における手動補正フロー

### 信頼性レベル分布

**ヒアリング前**（たたき台のみ参照）:
- 🔵 青信号: 約 5 件（直接記述あり）
- 🟡 黄信号: 約 20 件（妥当な推測）
- 🔴 赤信号: 約 10 件（推測ベース）

**ヒアリング後**:
- 🔵 青信号: 30 件（+25）
- 🟡 黄信号: 12 件（-8）
- 🔴 赤信号: 0 件（-10）

**品質評価**: **高品質**（赤信号 0 件、青信号 71%）

## 関連文書

- **要件定義書**: [requirements.md](requirements.md)
- **ユーザストーリー**: [user-stories.md](user-stories.md)
- **受け入れ基準**: [acceptance-criteria.md](acceptance-criteria.md)
- **準備タスク**: [prep.md](prep.md)
