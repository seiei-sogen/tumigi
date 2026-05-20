# ts-spec-ast-search ユーザストーリー

**作成日**: 2026-05-19
**関連要件定義**: [requirements.md](requirements.md)
**ヒアリング記録**: [interview-record.md](interview-record.md)

**【信頼性レベル凡例】**:
- 🔵 **青信号**: たたき台・ヒアリング結果に基づく確実なストーリー
- 🟡 **黄信号**: たたき台・参考案から妥当な推測によるストーリー
- 🔴 **赤信号**: たたき台・ヒアリングにない推測によるストーリー

---

## エピック 1: AST レベル検索基盤の整備

### ストーリー 1.1: tumigi の仕様を構造化された TypeScript object として参照する 🔵

**信頼性**: 🔵 *tatakidai/requirements-ast.md・ヒアリング Q1, Q3 より*

**私は** tumigi を利用する LLM コーディングエージェント **として**
**自然言語による曖昧な検索ではなく、構造化された TS object を AST レベルで検索したい**
**そうすることで** 目的のコマンド・スキル・要件に最短経路でアクセスでき、文脈ウィンドウの消費を抑えられる

**関連要件**: REQ-001, REQ-002, REQ-004, REQ-101

**詳細シナリオ**:
1. エージェントは `specs/**/*.ts` 配下に仕様 TS object が存在することを `specs/` を参照して確認する
2. エージェントは `ast-grep run --lang ts -p 'area: "kairo"' specs` を実行し、Kairo 関連 spec を一覧取得する
3. エージェントは取得した spec から `relatedCommands.path` を読み取り、必要な Markdown だけ読み込む

**前提条件**:
- `specs/_schema/tumigi-types.ts` で `TumigiArea` 型と各 ID 型が定義されている
- `specs/**/*.ts` 配下に最低 1 件の代表的な Requirement（例: `REQ-CMD-KAIRO-001`）が存在する

**制約事項**:
- 既存の `commands/*.md` を破壊しない

**優先度**: Must Have

---

### ストーリー 1.2: 仕様 ID を tsc 経由で参照解決する 🔵

**信頼性**: 🔵 *ヒアリング Q2「TS 型 System のシグネチャ検索」選択より*

**私は** spec を編集する開発者 **として**
**`dependsOn: ["REQ-CMD-KAIRO-001"]` のような ID 参照を IDE 上で型補完したい**
**そうすることで** 存在しない ID を書いてしまうことを未然に防げる

**関連要件**: REQ-002, REQ-003, REQ-103, NFR-201

**詳細シナリオ**:
1. 開発者は VS Code で `specs/commands/kairo.ts` を開く
2. `dependsOn: [...]` を入力したとき、IDE が既存の `TumigiRequirementId` 値（または定数）を補完候補に出す
3. 存在しない ID を書いた場合、`tsc --noEmit` が型エラーで検出する

**前提条件**:
- `specs/_schema/ids.ts` が `Uppercase<string>` リテラル型で ID パターンを表現している
- すべての spec ファイルが TypeScript として型チェックされる

**優先度**: Must Have

---

### ストーリー 1.3: ast-grep の検索クエリ例をドキュメントから引ける 🔵

**信頼性**: 🔵 *ヒアリング Q9「検索サンプル（コマンド集）の提供」Must Have より*

**私は** tumigi に初めて触れる開発者・エージェント **として**
**README または MANUAL に AST 検索の代表クエリ集が記載されていてほしい**
**そうすることで** ツールチェーンの使い方を別途調べる必要なく即座に検索できる

**関連要件**: NFR-202, NFR-203

**詳細シナリオ**:
1. 開発者が README の「AST 検索クエリ」セクションを開く
2. `area: "kairo"` 検索、`name: "kairo-requirements"` 検索、`pathPattern: "docs/tumigidoc/..."` 検索など最低 5 例が掲載されている
3. 開発者はクエリをコピー＆ペーストし、すぐに動作確認できる

**前提条件**:
- README または MANUAL に「AST 検索クエリ」セクションが新設されている

**優先度**: Must Have

---

### ストーリー 1.4: cocoindex-code でセマンティック検索する 🔵

**信頼性**: 🔵 *tatakidai/requirements-ast.md・ヒアリング Q2 より*

**私は** 用語ゆれを含む検索を行いたいエージェント **として**
**`ccc search "要件定義 EARS"` で AST マッチでは取れない関連 spec を取得したい**
**そうすることで** ast-grep の構造マッチと、ccc のセマンティック検索を補完的に使える

**関連要件**: REQ-102

**詳細シナリオ**:
1. エージェントは ast-grep で構造的に絞り込めない場合、`ccc search "<keywords>"` を実行
2. cocoindex-code が `specs/` のインデックスから関連 spec を返す
3. エージェントは返ってきた spec を読み解く

**前提条件**:
- cocoindex-code が tumigi リポジトリにセットアップ済み
- インデックスが `specs/` を対象として更新済み

**優先度**: Must Have

---

## エピック 2: Markdown と spec の整合性確保

### ストーリー 2.1: 既存 Markdown を破壊せずに spec を追加する 🔵

**信頼性**: 🔵 *ヒアリング Q1, Q8 より*

**私は** 既存ユーザ・rulesync 経由の他ツール利用者 **として**
**TS spec 導入後も `commands/*.md` のスラッシュコマンドが全く同じように動作してほしい**
**そうすることで** 移行コストなく恩恵を受けられる

**関連要件**: REQ-401, REQ-402, EDGE-201, EDGE-202

**詳細シナリオ**:
1. ユーザは TS spec を一切意識せず `/tumigi:kairo-requirements` を実行
2. Claude Code は従来通り `commands/kairo-requirements.md` を読み込み挙動する
3. rulesync で `--targets cursor --features commands` を実行しても、出力結果は spec 導入前と一致する

**前提条件**:
- spec ファイルが Markdown の挙動を一切上書きしない

**優先度**: Must Have

---

### ストーリー 2.2: README とコマンド実体の名前ズレを検出する 🔵

**信頼性**: 🔵 *chatgpt-idea-ts-object.md §12・ヒアリング Q3-A「フル検査」選択より*

**私は** リポジトリのメンテナ **として**
**README に書かれているコマンド名と `commands/*.md` の実体ファイルがズレていないかを自動検査したい**
**そうすることで** ドキュメントと実装の不整合によるユーザ混乱を防げる

**関連要件**: REQ-104, REQ-202, EDGE-001, EDGE-003

**詳細シナリオ**:
1. メンテナは `pnpm spec:check` を実行
2. スクリプトは spec の `relatedCommands.path` を全件確認し、実在しないパスを警告
3. README に記載されているコマンド名と `commands/*.md` の対応を照合し、片方にしか存在しないコマンドを警告
4. すべての spec ID が一意であることを確認

**前提条件**:
- spec:check スクリプトが `package.json` に登録されている
- `specs/**/*.ts` が型チェックを通る状態である

**優先度**: Should Have

---

### ストーリー 2.3: 既存 Markdown から spec を初期生成する 🔵

**信頼性**: 🔵 *ヒアリング Q7「既存 Markdown から生成スクリプト」選択より*

**私は** spec 導入を進めるメンテナ **として**
**`commands/*.md`・`skills/**/SKILL.md` の frontmatter から TS spec を一括生成したい**
**そうすることで** 30 件以上ある既存定義を手作業で書き写す必要がなくなる

**関連要件**: REQ-301, EDGE-004

**詳細シナリオ**:
1. メンテナは `pnpm spec:generate` を実行
2. スクリプトは `commands/*.md` の frontmatter（description, allowed-tools, argument-hint）をパース
3. `specs/commands/<name>.ts` に `defineRequirement({...})` または `defineCommand({...})` の雛形を出力
4. 既存ファイルがある場合は確認プロンプトを出す（または `--force` を要求）
5. 生成後、メンテナは EARS 記述や outputContracts を手動補完する

**前提条件**:
- `commands/*.md` に frontmatter が存在する
- TS spec のスキーマが確定している

**優先度**: Should Have

**備考**: 初期生成後の手動補正フローは kairo-design で詳細化が必要

---

## エピック 3: ドキュメントと運用基盤

### ストーリー 3.1: spec の書き方ルールを AGENTS.md / CLAUDE.md で参照する 🔵

**信頼性**: 🔵 *chatgpt-idea-ts-object.md §14・ヒアリング NFR-203 関連*

**私は** 新しい開発者・エージェント **として**
**「コマンド追加時にどう spec を更新するか」が AGENTS.md または CLAUDE.md に明記されていてほしい**
**そうすることで** 規約から外れた spec を書かずに済む

**関連要件**: NFR-203

**詳細シナリオ**:
1. 新しい開発者がリポジトリを clone
2. CLAUDE.md を開くと「Tumigi Specification Policy」セクションに以下が記載されている:
   - `specs/**/*.ts` の役割
   - 仕様確認手順（実装前のチェックリスト）
   - 重要な出力契約
   - 禁止事項（コマンド名変更時の不整合等）
3. 開発者は規約に従い spec を追加・更新する

**前提条件**:
- CLAUDE.md（または AGENTS.md）に Specification Policy セクションが追記されている

**優先度**: Must Have

---

### ストーリー 3.2: CI で spec の整合性を継続的に保証する 🔵

**信頼性**: 🔵 *ヒアリング Q9「Should Have」リスト内*

**私は** リポジトリのメンテナ **として**
**PR のたびに spec:check が自動実行されてほしい**
**そうすることで** マージ後に整合性崩壊を発見するリスクを下げられる

**関連要件**: REQ-302

**詳細シナリオ**:
1. 開発者が PR を出す
2. GitHub Actions が `pnpm install && pnpm spec:check` を実行
3. 失敗時は PR にチェック失敗が表示される

**前提条件**:
- `.github/workflows/spec-check.yml` が追加されている
- spec:check スクリプトが実装済み

**優先度**: Should Have

---

### ストーリー 3.3: Decision（ADR 相当）で重要な設計判断を追跡する 🔵

**信頼性**: 🔵 *ヒアリング Q9「Should Have」リスト内 + chatgpt-idea-ts-object.md §3 TumigiDecision*

**私は** リポジトリのメンテナ・新規開発者 **として**
**「なぜこのコマンド構造になったか」を Decision spec で振り返りたい**
**そうすることで** 過去の判断理由を踏まえた一貫した拡張ができる

**関連要件**: REQ-303

**詳細シナリオ**:
1. メンテナは設計判断時に `specs/decisions/<name>.ts` に `defineDecision({...})` で context / decision / consequences を記録
2. spec から `affects: ["REQ-CMD-KAIRO-001"]` で影響範囲を明示
3. 新規開発者は ast-grep で `kind: "decision"` を検索して過去判断を参照する

**前提条件**:
- Decision テンプレートとサンプルが用意されている

**優先度**: Should Have

---

## ストーリーマップ

```
エピック 1: AST レベル検索基盤の整備
├── ストーリー 1.1 (🔵 Must Have) — TS object 構造化検索
├── ストーリー 1.2 (🔵 Must Have) — tsc 経由の参照解決
├── ストーリー 1.3 (🔵 Must Have) — ast-grep クエリ例ドキュメント
└── ストーリー 1.4 (🔵 Must Have) — cocoindex-code セマンティック検索

エピック 2: Markdown と spec の整合性確保
├── ストーリー 2.1 (🔵 Must Have) — 既存 Markdown 非破壊
├── ストーリー 2.2 (🔵 Should Have) — README 不整合検出
└── ストーリー 2.3 (🔵 Should Have) — spec 初期生成スクリプト

エピック 3: ドキュメントと運用基盤
├── ストーリー 3.1 (🔵 Must Have) — Specification Policy 文書化
├── ストーリー 3.2 (🔵 Should Have) — CI 自動 spec:check
└── ストーリー 3.3 (🔵 Should Have) — Decision (ADR) 追跡
```

## 信頼性レベルサマリー

- 🔵 青信号: 10 件 (100%)
- 🟡 黄信号: 0 件 (0%)
- 🔴 赤信号: 0 件 (0%)

**品質評価**: **高品質**（全ストーリーがヒアリングまたはたたき台に直接根拠）
