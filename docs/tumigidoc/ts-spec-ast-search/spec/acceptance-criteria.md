# ts-spec-ast-search 受け入れ基準

**作成日**: 2026-05-19
**関連要件定義**: [requirements.md](requirements.md)
**関連ユーザストーリー**: [user-stories.md](user-stories.md)
**ヒアリング記録**: [interview-record.md](interview-record.md)

**【信頼性レベル凡例】**:
- 🔵 **青信号**: たたき台・ヒアリング結果に基づく確実な基準
- 🟡 **黄信号**: たたき台・参考案から妥当な推測による基準
- 🔴 **赤信号**: たたき台・ヒアリングにない推測による基準

---

## REQ-001: spec の TS object 保持 🔵

**信頼性**: 🔵 *tatakidai/requirements-ast.md・ヒアリング Q3 より*

### Given（前提条件）
- リポジトリルートに `specs/` ディレクトリを作成可能
- TypeScript 環境が準備されている

### When（実行条件）
- `specs/_schema/tumigi-types.ts` と `specs/commands/kairo.ts` 等を配置する

### Then（期待結果）
- `specs/**/*.ts` 配下に Requirement / Workflow / Task / Decision / Verification の TS object が存在する
- 各 object が `defineXxx` で wrap されている

### テストケース

#### 正常系

- [ ] **TC-001-01**: 代表 spec ファイルの存在確認 🔵
  - **入力**: `find specs -name "*.ts" -type f` 相当のチェック
  - **期待結果**: 最低 1 件以上の `.ts` ファイルが存在し、`defineRequirement` 等の identity 関数を呼び出している
  - **信頼性**: 🔵 *REQ-001 直接*

#### 異常系

- [ ] **TC-001-E01**: スキーマ未定義での import 失敗 🔵
  - **入力**: `specs/_schema/define.ts` を意図的に削除して `specs/commands/kairo.ts` をコンパイル
  - **期待結果**: tsc が import エラーを返す
  - **信頼性**: 🔵 *TypeScript 言語仕様*

---

## REQ-002: defineXxx identity 関数の提供 🔵

**信頼性**: 🔵 *chatgpt-idea-ts-object.md §4 より*

### Given
- `specs/_schema/define.ts` が存在する

### When
- `import { defineRequirement } from "./_schema/define"` を実行する

### Then
- 型補完が効き、`defineRequirement({...})` の引数に対し `TumigiRequirement` 型のチェックが働く

### テストケース

#### 正常系

- [ ] **TC-002-01**: 全 5 種類の define 関数が export されている 🔵
  - **入力**: `specs/_schema/define.ts` の export 一覧
  - **期待結果**: `defineRequirement`, `defineWorkflow`, `defineTask`, `defineDecision`, `defineVerification` の 5 つが export
  - **信頼性**: 🔵 *chatgpt-idea-ts-object.md §4*

#### 異常系

- [ ] **TC-002-E01**: 必須プロパティ欠落で tsc エラー 🔵
  - **入力**: `defineRequirement({ id: "REQ-CMD-KAIRO-001" })` のように `title` 等を省略
  - **期待結果**: tsc が型エラーを返す
  - **信頼性**: 🔵 *TypeScript リテラル型・必須プロパティ仕様*

---

## REQ-003: 一意 ID 体系 🔵

**信頼性**: 🔵 *ヒアリング Q5「tatakidai 案を採用」より*

### Given
- `specs/_schema/ids.ts` で ID 型が定義されている

### When
- 各 spec ファイルが `id: "REQ-CMD-KAIRO-001"` のように area プレフィックス付き ID を持つ

### Then
- ID 重複時には `pnpm spec:check` がエラーを返す
- ID が型パターン（例: `REQ-CMD-${Uppercase<string>}-${number}`）に違反する場合 tsc がエラーを返す

### テストケース

#### 正常系

- [ ] **TC-003-01**: 全 spec の ID 一意性 🔵
  - **入力**: `specs/**/*.ts` を全件パースし ID をリストアップ
  - **期待結果**: 重複なし
  - **信頼性**: 🔵 *REQ-405・EDGE-002*

#### 異常系

- [ ] **TC-003-E01**: 重複 ID 検出 🔵
  - **入力**: 同じ ID を持つ 2 つの spec を配置し `pnpm spec:check` を実行
  - **期待結果**: スクリプトが重複箇所を表示してエラー終了
  - **信頼性**: 🔵 *EDGE-002*

- [ ] **TC-003-E02**: ID 型違反検出 🔵
  - **入力**: `id: "INVALID-FORMAT"` を spec に書く
  - **期待結果**: tsc が型エラーを返す
  - **信頼性**: 🔵 *TypeScript リテラル型仕様*

#### 境界値

- [ ] **TC-003-B01**: ID の `number` 部分の上限 🟡
  - **入力**: `REQ-CMD-KAIRO-999999` のような大きい数値
  - **期待結果**: tsc がエラーを返さない（`number` リテラルに上限なし）
  - **信頼性**: 🟡 *TypeScript 言語仕様から推測*

---

## REQ-101: ast-grep による AST 検索 🔵

**信頼性**: 🔵 *tatakidai/requirements-ast.md より*

### Given
- ast-grep がインストール済み
- `specs/commands/kairo.ts` が `area: "kairo"` を持つ Requirement を含む

### When
- `ast-grep run --lang ts -p 'area: "kairo"' specs` を実行する

### Then
- 対象 spec のマッチが標準出力に返る

### テストケース

#### 正常系

- [ ] **TC-101-01**: area 検索の動作 🔵
  - **入力**: `ast-grep run --lang ts -p 'area: "kairo"' specs`
  - **期待結果**: `specs/commands/kairo.ts` の該当行がヒット
  - **信頼性**: 🔵 *tatakidai/requirements-ast.md*

- [ ] **TC-101-02**: コマンド名検索 🔵
  - **入力**: `ast-grep run --lang ts -p 'name: "kairo-requirements"' specs`
  - **期待結果**: relatedCommands に `kairo-requirements` を含む全 spec がヒット
  - **信頼性**: 🔵 *chatgpt-idea-ts-object.md §13*

- [ ] **TC-101-03**: 出力契約パターン検索 🔵
  - **入力**: `ast-grep run --lang ts -p 'pathPattern: "docs/tumigidoc/$_"' specs`
  - **期待結果**: 該当する outputContracts を持つ spec がヒット
  - **信頼性**: 🔵 *chatgpt-idea-ts-object.md §13*

---

## REQ-102: cocoindex-code セマンティック検索 🔵

**信頼性**: 🔵 *tatakidai/requirements-ast.md より*

### Given
- cocoindex-code がセットアップ済み
- `specs/` がインデックス対象に登録されている

### When
- `ccc search "EARS 要件定義"` を実行する

### Then
- 関連 spec の上位候補が返る

### テストケース

#### 正常系

- [ ] **TC-102-01**: セマンティック検索の動作 🔵
  - **入力**: `ccc search "EARS 要件定義"`
  - **期待結果**: Kairo 要件関連の spec が上位にランクイン
  - **信頼性**: 🔵 *tatakidai/requirements-ast.md*

---

## REQ-103: TypeScript 型 System 経由の参照解決 🔵

**信頼性**: 🔵 *ヒアリング Q2 より*

### Given
- `specs/_schema/ids.ts` で ID 型が定義されている
- 各 spec が `dependsOn: ["REQ-CMD-KAIRO-001"]` を含む

### When
- VS Code（または tsc）で spec ファイルを開く

### Then
- `dependsOn` の値が型補完候補に出る
- 存在しない ID は型エラーになる

### テストケース

#### 正常系

- [ ] **TC-103-01**: ID 型補完 🔵
  - **入力**: VS Code で `dependsOn: [...]` を入力
  - **期待結果**: 既存 ID が補完候補に表示される
  - **信頼性**: 🔵 *TypeScript Language Service*

#### 異常系

- [ ] **TC-103-E01**: 存在しない ID 検出 🔵
  - **入力**: `dependsOn: ["REQ-NONEXISTENT-001"]`
  - **期待結果**: spec:check スクリプトがエラーを返す（型レベルでは `Uppercase<string>` パターンを満たすため tsc は通る場合あり）
  - **信頼性**: 🔵 *REQ-405・EDGE-001*

---

## REQ-104: pnpm spec:check の表示 🟡

**信頼性**: 🟡 *ヒアリング Q3-A「フル検査」選択から妥当な推測*

### Given
- `specs/**/*.ts` が存在する

### When
- `pnpm spec:check` を実行する

### Then
- 検査結果（成功・失敗・違反箇所）が標準出力に表示される
- 失敗時は exit code が 0 以外

### テストケース

#### 正常系

- [ ] **TC-104-01**: 全件成功時の出力 🟡
  - **入力**: 整合の取れた spec で `pnpm spec:check`
  - **期待結果**: `OK` 表示と exit 0
  - **信頼性**: 🟡 *generic CLI 設計から推測*

#### 異常系

- [ ] **TC-104-E01**: パス不在検出 🔵
  - **入力**: `relatedCommands.path: "commands/nonexistent.md"` を含む spec で実行
  - **期待結果**: 該当 ID とパスを表示しエラー終了
  - **信頼性**: 🔵 *EDGE-001*

- [ ] **TC-104-E02**: frontmatter 欠落検出 🔵
  - **入力**: `commands/*.md` で description を欠落させた状態
  - **期待結果**: 該当ファイルを表示しエラー終了
  - **信頼性**: 🔵 *chatgpt-idea-ts-object.md §12*

---

## REQ-301: 既存 Markdown → TS spec 生成スクリプト 🔵

**信頼性**: 🔵 *ヒアリング Q7 より*

### Given
- `commands/*.md` に frontmatter が存在する

### When
- `pnpm spec:generate` を実行する

### Then
- `specs/commands/<name>.ts` に `defineRequirement({...})` の雛形が出力される

### テストケース

#### 正常系

- [ ] **TC-301-01**: 雛形生成 🔵
  - **入力**: `commands/kairo-requirements.md` を対象に `pnpm spec:generate`
  - **期待結果**: `specs/commands/kairo-requirements.ts` が生成される
  - **信頼性**: 🔵 *REQ-301*

#### 異常系

- [ ] **TC-301-E01**: 既存 spec 上書き保護 🟡
  - **入力**: 既に `specs/commands/kairo-requirements.ts` が存在する状態で再実行
  - **期待結果**: 確認プロンプトまたは `--force` 要求
  - **信頼性**: 🟡 *EDGE-004*

---

## REQ-401: 既存仕様の非破壊保証 🔵

**信頼性**: 🔵 *ヒアリング Q1 より*

### Given
- TS spec 導入前後の commit を比較できる

### When
- 比較対象は `commands/*.md`、`skills/**/SKILL.md`、`.claude-plugin/*.json`

### Then
- いずれのファイルも内容変更されていない（追加ファイルとして spec のみが入る）

### テストケース

#### 正常系

- [ ] **TC-401-01**: 既存 Markdown の非変更確認 🔵
  - **入力**: `git diff <before>..<after> -- commands/ skills/ .claude-plugin/`
  - **期待結果**: 変更なし（または極小の typo 修正のみ）
  - **信頼性**: 🔵 *REQ-401・EDGE-201*

---

## REQ-402: rulesync 互換性 🔵

**信頼性**: 🔵 *ヒアリング Q8 より*

### Given
- TS spec 導入前と導入後で同じ rulesync コマンドを実行する

### When
- `npx -y rulesync generate --targets cursor --features commands --experimental-simulate-commands` を実行

### Then
- 生成結果が spec 導入前と一致する（specs/ は出力対象に含まれない）

### テストケース

#### 正常系

- [ ] **TC-402-01**: rulesync 出力比較 🔵
  - **入力**: 導入前後の rulesync generate 出力を diff
  - **期待結果**: 差分ゼロ
  - **信頼性**: 🔵 *REQ-402・EDGE-202*

---

## REQ-403: tsc 型チェック通過 🟡

**信頼性**: 🟡 *REQ-002 と既存技術スタックから妥当な推測*

### Given
- `specs/**/*.ts` と `tsconfig.json` が存在する

### When
- `pnpm tsc --noEmit` を実行する

### Then
- 型エラーゼロ

### テストケース

#### 正常系

- [ ] **TC-403-01**: 型チェック通過 🟡
  - **入力**: `pnpm tsc --noEmit`
  - **期待結果**: エラー 0 件、exit 0
  - **信頼性**: 🟡 *REQ-403*

---

## 非機能要件テスト

### NFR-001: spec:check のパフォーマンス 🟡

**信頼性**: 🟡 *generic な経験則*

- [ ] **TC-NFR-001-01**: 50 件規模の検査速度
  - **測定項目**: `pnpm spec:check` の実行時間
  - **目標値**: 10 秒以内
  - **測定条件**: spec ファイル 50 件、ローカル開発機
  - **信頼性**: 🟡 *経験則からの推測*

### NFR-002: ast-grep の検索速度 🟡

- [ ] **TC-NFR-002-01**: AST 検索の応答時間
  - **測定項目**: `ast-grep run --pattern '...' specs` の応答時間
  - **目標値**: 1 秒以内
  - **測定条件**: spec ファイル 50 件、ローカル開発機
  - **信頼性**: 🟡 *ast-grep 経験則*

### NFR-101: spec ファイル機密情報フリー 🔵

- [ ] **TC-NFR-101-01**: secretlint 通過
  - **検証内容**: `pnpm secretlint **/*.ts` で機密情報検出ゼロ
  - **期待結果**: エラー 0 件
  - **信頼性**: 🔵 *既存 secretlint 運用*

### NFR-201: IDE 型補完 🔵

- [ ] **TC-NFR-201-01**: VS Code 型補完
  - **検証内容**: `specs/commands/kairo.ts` で `area: ` を入力
  - **期待結果**: `"product" | "kairo" | "tdd" | ...` のリテラル補完
  - **信頼性**: 🔵 *TypeScript Language Service*

### NFR-202: AST 検索クエリ集の記載 🔵

- [ ] **TC-NFR-202-01**: README 記載
  - **検証内容**: README または MANUAL に「AST 検索クエリ」セクションが存在
  - **期待結果**: 最低 5 件の代表クエリ例が記載
  - **信頼性**: 🔵 *ヒアリング Q9 Must Have*

### NFR-203: Specification Policy 文書 🔵

- [ ] **TC-NFR-203-01**: CLAUDE.md / AGENTS.md 記載
  - **検証内容**: 仕様確認手順・出力契約・禁止事項が記載
  - **期待結果**: 全項目が記載されている
  - **信頼性**: 🔵 *chatgpt-idea-ts-object.md §14*

---

## Edge ケーステスト

### EDGE-001: relatedCommands.path 不在検出 🔵

- [ ] **TC-EDGE-001-01**: パス不在エラー表示
  - **条件**: 存在しない `commands/foo.md` を `relatedCommands.path` に指定
  - **期待結果**: spec:check が該当 ID とパスを表示しエラー終了
  - **信頼性**: 🔵 *EDGE-001*

### EDGE-002: ID 重複検出 🔵

- [ ] **TC-EDGE-002-01**: 重複 ID 警告
  - **条件**: 同一 ID を持つ 2 spec を配置
  - **期待結果**: spec:check が重複 ID をリストアップしエラー終了
  - **信頼性**: 🔵 *EDGE-002*

### EDGE-003: README コマンド名ズレ検出 🔵

- [ ] **TC-EDGE-003-01**: 名前ズレ警告
  - **条件**: README に記載のあるコマンドが `commands/*.md` に存在しない
  - **期待結果**: spec:check が警告を出す
  - **信頼性**: 🔵 *EDGE-003*

### EDGE-004: 生成スクリプトの上書き保護 🟡

- [ ] **TC-EDGE-004-01**: 確認プロンプト
  - **条件**: 既存 `specs/commands/kairo.ts` がある状態で `pnpm spec:generate` 実行
  - **期待結果**: 確認プロンプトまたは `--force` 要求
  - **信頼性**: 🟡 *EDGE-004*

### EDGE-101: 多数の依存関係 🟡

- [ ] **TC-EDGE-101-01**: 大量 dependsOn
  - **条件**: `dependsOn` に 100 件の ID を列挙
  - **期待結果**: tsc / spec:check ともに正常動作
  - **信頼性**: 🟡 *EDGE-101*

### EDGE-201: TS spec 導入なしでの動作不変 🔵

- [ ] **TC-EDGE-201-01**: spec 未配置時の挙動
  - **条件**: `specs/` ディレクトリ自体がない状態
  - **期待結果**: tumigi スラッシュコマンドはすべて従来通り動作
  - **信頼性**: 🔵 *REQ-401・EDGE-201*

### EDGE-202: rulesync 出力からの除外 🟡

- [ ] **TC-EDGE-202-01**: rulesync で specs/ が除外
  - **条件**: `npx -y rulesync generate --targets geminicli --features commands`
  - **期待結果**: 出力に `specs/` 由来のファイルが含まれない
  - **信頼性**: 🟡 *EDGE-202*

---

## テストケースサマリー

### カテゴリ別件数

| カテゴリ | 正常系 | 異常系 | 境界値 | 合計 |
|---------|--------|--------|--------|------|
| 機能要件 | 9 | 6 | 1 | 16 |
| 非機能要件 | 6 | 0 | 0 | 6 |
| Edge ケース | 4 | 0 | 3 | 7 |
| **合計** | **19** | **6** | **4** | **29** |

### 信頼性レベル分布

- 🔵 青信号: 22 件 (75.9%)
- 🟡 黄信号: 7 件 (24.1%)
- 🔴 赤信号: 0 件 (0.0%)

**品質評価**: **高品質**（赤信号 0 件、青信号 76%）

### 優先度別テストケース

- **Must Have**（REQ-001 〜 REQ-005, REQ-101 〜 REQ-103, REQ-401 〜 REQ-405, NFR-101 〜 NFR-203）: 19 件
- **Should Have**（REQ-104, REQ-301, REQ-302, REQ-303, EDGE-001 〜 EDGE-004）: 10 件
- **Could Have**（REQ-304, NFR-001, NFR-002, NFR-301）: 0 件（このフェーズではテストケース化していない）

---

## テスト実施計画

### Phase 1: Must Have 基本機能テスト
- REQ-001, REQ-002, REQ-003, REQ-101, REQ-102, REQ-103, REQ-401, REQ-402, REQ-403
- NFR-101, NFR-201, NFR-202, NFR-203
- 実施タイミング: TS spec スキーマ整備直後

### Phase 2: Should Have 拡張機能テスト
- REQ-104, REQ-301
- EDGE-001, EDGE-002, EDGE-003, EDGE-004
- 実施タイミング: spec:check / spec:generate スクリプト実装後

### Phase 3: パフォーマンス・互換性テスト
- NFR-001, NFR-002
- EDGE-201, EDGE-202
- 実施タイミング: 全実装完了後、リリース前
