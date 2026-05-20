# ts-spec-ast-search 要件定義書

## 概要

tumigi のコマンド・スキル・ワークフロー・出力契約を **TypeScript object** として `specs/**/*.ts` 配下に二重表現し、`ast-grep` / `cocoindex-code (ccc search)` / TypeScript 型システムによる **AST レベルでの構造化検索** を可能にする。

これにより LLM が tumigi ドキュメントを参照する際、自然言語マッチング（Markdown 全文検索）に頼らず、`area: "kairo"` や `name: "kairo-requirements"` のようなパターン一致で目的の仕様へ到達できる。既存の Markdown コマンド・スキル定義は **そのまま保持** し、TS spec は **追加レイヤ（メタデータ層）** として並走する。

## 関連文書

- **ヒアリング記録**: [💬 interview-record.md](interview-record.md)
- **ユーザストーリー**: [📖 user-stories.md](user-stories.md)
- **受け入れ基準**: [✅ acceptance-criteria.md](acceptance-criteria.md)
- **コンテキストノート**: [📝 note.md](note.md)
- **準備タスク**: [🔧 prep.md](prep.md)
- **PRD 相当のたたき台**: [📄 ../../../tatakidai/requirements-ast.md](../../../tatakidai/requirements-ast.md)
- **スキーマ参考案**: [📄 ../../../tatakidai/chatgpt-idea-ts-object.md](../../../tatakidai/chatgpt-idea-ts-object.md)

## 機能要件（EARS記法）

**【信頼性レベル凡例】**:
- 🔵 **青信号**: たたき台・ヒアリング結果に直接基づく確実な要件
- 🟡 **黄信号**: たたき台・既存設計から妥当に推測した要件
- 🔴 **赤信号**: たたき台・ヒアリングにない推測による要件

### 通常要件

- **REQ-001**: システムは、`specs/**/*.ts` 配下に TypeScript object として要件（Requirement）・ワークフロー（Workflow）・タスク（Task）・決定（Decision）・検証（Verification）を保持しなければならない 🔵 *tatakidai/requirements-ast.md および ヒアリング Q3 より*
- **REQ-002**: システムは、各 TS object に対し `defineRequirement` / `defineWorkflow` / `defineTask` / `defineDecision` / `defineVerification` の identity 関数を提供し、型補完と型チェックを保証しなければならない 🔵 *chatgpt-idea-ts-object.md §4 より*
- **REQ-003**: システムは、各 spec に一意の ID（`REQ-TUMIGI-001`、`REQ-CMD-KAIRO-001`、`REQ-SKILL-DEV-001` 等の area プレフィックス付き連番）を付与しなければならない 🔵 *ヒアリング Q5 で「tatakidai案を採用」選択*
- **REQ-004**: システムは、各 spec に `area` フィールド（`product` / `kairo` / `tdd` / `dev-skills` / `dcs` / `utility` / `reverse-engineering` / `claude-plugin` / `rulesync` / `documentation` / `quality`）を持たせ、area 単位での横断検索を可能にしなければならない 🔵 *chatgpt-idea-ts-object.md §3 より*
- **REQ-005**: システムは、spec の `implementedBy` / `relatedCommands` / `relatedSkills` フィールドで **commands/*.md**、**skills/**\/SKILL.md**、**docs/\*\* ドキュメント** を参照できなければならない 🔵 *ヒアリング Q6（再質問）より*

### 条件付き要件

- **REQ-101**: ユーザーが `ast-grep run --lang ts -p '<pattern>' specs` を実行する場合、システムは TS spec を AST レベルでマッチし対象 spec を返さなければならない 🔵 *tatakidai/requirements-ast.md「AST レベルでの検索」より*
- **REQ-102**: ユーザーが `ccc search <query>` を実行する場合、システムは cocoindex-code のインデックスを通じて関連 spec をセマンティック検索できなければならない 🔵 *tatakidai/requirements-ast.md より*
- **REQ-103**: ユーザーが TypeScript 言語サーバ（tsc / IDE）を介して ID や型を参照する場合、システムは spec 間の参照関係（`dependsOn`、`implements`、`verifies`）を型エラーなく解決しなければならない 🔵 *ヒアリング Q2「TS 型 System のシグネチャ検索」選択より*
- **REQ-104**: ユーザーが `pnpm spec:check` を実行する場合、システムは spec 検査結果（成功・失敗・違反箇所）を標準出力に表示しなければならない 🟡 *ヒアリング Q3-A「フル検査」選択から妥当な推測*

### 状態要件

- **REQ-201**: spec の `status` が `"draft"` / `"accepted"` / `"implemented"` / `"verified"` / `"deprecated"` のいずれかである場合、システムはその状態を spec フィールドとして保持しなければならない 🔵 *chatgpt-idea-ts-object.md §3 RequirementStatus より*
- **REQ-202**: spec が `deprecated` 状態である場合、システムは README / MANUAL から該当コマンドが削除されていることを `spec:check` で検出できなければならない 🟡 *chatgpt-idea-ts-object.md §12 「deprecated扱いのコマンドがREADMEに残っていない」より*

### オプション要件

- **REQ-301**: システムは、既存 Markdown（`commands/*.md`、`skills/**/SKILL.md`）の frontmatter をパースし、初期 TS spec を自動生成するスクリプト（例: `pnpm spec:generate`）を提供してもよい 🔵 *ヒアリング Q6 で「既存Markdownから生成スクリプト」選択*
- **REQ-302**: システムは、spec チェックを CI（GitHub Actions 等）で自動実行する設定を提供してもよい 🔵 *ヒアリング Q9 「Should Have」リスト内*
- **REQ-303**: システムは、Decision（ADR 相当）と Verification（検証手段）の記述テンプレートを提供してもよい 🔵 *ヒアリング Q9 「Should Have」リスト内*
- **REQ-304**: システムは、pre-commit フック（simple-git-hooks）に spec チェックを組み込んでもよい 🟡 *ヒアリング Q3-B「手動運用」を選択したため pre-commit は任意レベル*

### 制約要件

- **REQ-401**: システムは、既存の `commands/*.md`、`skills/**/SKILL.md`、`.claude-plugin/*.json` のいずれも破壊的変更してはならない（追加レイヤとして動作） 🔵 *ヒアリング Q1「TS specを指し示す層として併存」選択*
- **REQ-402**: システムは、rulesync 経由の generate（`--targets claudecode/geminicli/cursor/copilot/codexcli/roo`）の挙動を変更してはならない 🔵 *ヒアリング Q8「現状の仕組みを維持」選択*
- **REQ-403**: spec ファイルは Node.js + TypeScript 環境で型チェックが通らなければならない（`tsc --noEmit` で 0 件エラー） 🟡 *REQ-002 と既存技術スタック (pnpm) から妥当な推測*
- **REQ-404**: spec ファイルは `pnpm secretlint` の対象に含め、機密情報を含んではならない 🟡 *既存 pre-commit 設定 (CLAUDE.md) から妥当な推測*
- **REQ-405**: spec ID は重複してはならず、`spec:check` で一意性を検証できなければならない 🟡 *chatgpt-idea-ts-object.md §12 検査一覧から妥当な推測*

## 非機能要件

### パフォーマンス

- **NFR-001**: `pnpm spec:check` の実行時間は、spec ファイル 50 件規模で 10 秒以内に完了することが望ましい 🟡 *typical な tsc + glob 走査の経験則から妥当な推測*
- **NFR-002**: `ast-grep run --pattern '<query>' specs` の検索応答は spec 50 件規模で 1 秒以内であることが望ましい 🟡 *ast-grep の一般的な実行速度から妥当な推測*

### セキュリティ

- **NFR-101**: spec ファイルには API キー・トークン・個人情報を記載してはならない 🔵 *既存 secretlint 運用 (CLAUDE.md) より*
- **NFR-102**: spec 検査スクリプトは外部ネットワークアクセスを行ってはならない（オフラインで完結） 🟡 *既存 pnpm scripts (secretlint, prepare) はすべてローカル完結であることから妥当な推測*

### ユーザビリティ

- **NFR-201**: spec の TypeScript 型定義は IDE（VS Code 等）で型補完が効かなければならない 🔵 *REQ-002・ヒアリング Q2 より*
- **NFR-202**: ast-grep / ccc search の代表的な検索クエリ集（最低 5 種類）を README または MANUAL に記載しなければならない 🔵 *ヒアリング Q9「検索サンプル（コマンド集）の提供」Must Have 選択*
- **NFR-203**: spec の記述ルール・ID 命名規則は AGENTS.md または CLAUDE.md に記載しなければならない 🔵 *chatgpt-idea-ts-object.md §14 より*

### 拡張性

- **NFR-301**: 新しい area（例: 新カテゴリのコマンド群）を追加する際、`TumigiArea` 型と ID 型を拡張するだけで spec を追加できなければならない 🟡 *chatgpt-idea-ts-object.md §2 ID設計から妥当な推測*

## Edge ケース

### エラー処理

- **EDGE-001**: spec の `relatedCommands.path` が実在しない場合、`spec:check` はエラーを返し、該当 spec の ID と存在しないパスを表示しなければならない 🔵 *chatgpt-idea-ts-object.md §12 検査一覧より*
- **EDGE-002**: spec の `id` が重複している場合、`spec:check` はエラーを返し、重複 ID を表示しなければならない 🔵 *REQ-405 と整合*
- **EDGE-003**: README / MANUAL に記載のコマンドが `commands/*.md` に存在しない場合、`spec:check` は警告を出さなければならない 🔵 *chatgpt-idea-ts-object.md §12「READMEに載っているコマンドと commands/*.md がズレていない」より*
- **EDGE-004**: spec 生成スクリプト（`spec:generate`）が既存の `specs/*.ts` を上書きしようとする場合、確認プロンプトまたは `--force` フラグを要求しなければならない 🟡 *generic な CLI 設計慣行から妥当な推測*

### 境界値

- **EDGE-101**: 1 つの Requirement に紐づく `dependsOn` / `relatedCommands` / `relatedSkills` の件数に上限を設けない（ただし spec:check は重複参照を検出する） 🟡 *chatgpt-idea-ts-object.md §3 型定義に上限指定なし*
- **EDGE-102**: `area` 値は `TumigiArea` リテラル型に限定し、未定義の area を spec に書いた場合は `tsc` がコンパイルエラーを出す 🔵 *TypeScript リテラル型の言語仕様 + REQ-403 より*

### 互換性

- **EDGE-201**: 既存ユーザが TS spec を導入せずに tumigi を使い続けた場合、コマンド・スキルの動作は一切変わらない 🔵 *REQ-401 制約より*
- **EDGE-202**: rulesync で他ツール（cursor / geminicli 等）に export する際、`specs/**/*.ts` は出力対象から除外される 🟡 *REQ-402 と rulesync の `commands` feature 仕様から妥当な推測*

## 優先順位サマリー

### Must Have（必須）

| ID | 概要 |
|----|------|
| REQ-001, REQ-002, REQ-003, REQ-004, REQ-005 | TS spec スキーマと defineXxx 関数の整備 |
| REQ-101, REQ-102, REQ-103 | ast-grep / ccc search / TS 型 System による検索 |
| REQ-201 | spec status の表現 |
| REQ-401, REQ-402, REQ-403, REQ-404, REQ-405 | 既存仕組みへの非破壊・型安全制約 |
| NFR-101, NFR-201, NFR-202, NFR-203 | セキュリティ・ユーザビリティ・運用ドキュメント |

### Should Have（できればやりたい）

| ID | 概要 |
|----|------|
| REQ-104, REQ-202 | `spec:check` 実装 |
| REQ-301 | 既存 Markdown → TS spec 生成スクリプト |
| REQ-302 | CI（GitHub Actions）での spec:check 自動実行 |
| REQ-303 | Decision / Verification テンプレート |
| EDGE-001, EDGE-002, EDGE-003, EDGE-004 | spec:check のエラー検出機能 |

### Could Have（任意）

| ID | 概要 |
|----|------|
| REQ-304 | pre-commit フック組み込み |
| NFR-001, NFR-002 | パフォーマンス目標 |
| NFR-301 | 拡張性目標 |
