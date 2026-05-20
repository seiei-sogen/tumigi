# ts-spec-ast-search 準備タスク（ユーザー作業）

> **仕様**: [requirements.md](requirements.md)
> **生成日**: 2026-05-19

**【信頼性レベル凡例】**:
- 🔵 **青信号**: 要件定義書・ヒアリングで明確に必要と判明したタスク
- 🟡 **黄信号**: 要件定義書から妥当に推測されるタスク
- 🔴 **赤信号**: 推測による予防的タスク

## 必須（実装開始前に完了が必要）

以下のタスクが完了していないと、実装フェーズでブロッカーになります。

- [ ] **TypeScript のリポジトリ導入** 🔵 *REQ-002, REQ-403 より*
  - tumigi はこれまで TypeScript を採用していない（package.json に typescript 依存なし）
  - `pnpm add -D typescript @types/node` 等で TypeScript 環境を追加
  - `tsconfig.json` を新設（target: ES2022 等、strict: true 推奨）
  - 関連要件: REQ-002, REQ-403, NFR-201

- [ ] **`specs/` ディレクトリ配置の合意** 🔵 *chatgpt-idea-ts-object.md §1 より*
  - リポジトリルート直下に `specs/` を配置する方針で合意（参考案踏襲）
  - `.gitignore` / `secretlint` の対象範囲確認
  - 関連要件: REQ-001, NFR-101

## 推奨（実装中に用意できればOK）

実装を開始できますが、該当機能の実装前までに準備してください。

- [ ] **ast-grep のローカル/開発者環境へのインストール** 🔵 *REQ-101 より*
  - 公式: https://github.com/ast-grep/ast-grep
  - macOS: `brew install ast-grep` / Linux: バイナリ取得 / npm: `npm install -g @ast-grep/cli`
  - README / MANUAL の AST 検索クエリ集（NFR-202）に「インストール手順」も併記
  - 必要になるフェーズ: Phase 1（spec スキーマ整備直後の動作確認）
  - 関連要件: REQ-101, NFR-202

- [ ] **cocoindex-code（ccc）のセットアップ** 🔵 *REQ-102 より*
  - 公式: https://github.com/cocoindex-io/cocoindex-code
  - インストール後、`specs/` を対象にしたインデックス設定を作成
  - インデックス更新タイミング（手動/CI）を運用ルールで明示
  - 必要になるフェーズ: Phase 1 後半（セマンティック検索動作確認）
  - 関連要件: REQ-102, NFR-202

- [ ] **GitHub Actions 用 `pnpm spec:check` ワークフロー追加** 🔵 *REQ-302 より*
  - `.github/workflows/spec-check.yml` を追加
  - `pnpm install --frozen-lockfile && pnpm spec:check` を PR ごとに実行
  - 必要になるフェーズ: Phase 2（spec:check 実装後）
  - 関連要件: REQ-302

## 確認事項（判断が必要）

実装方針に影響するため、早めの判断・確認が推奨されます。

- [ ] **TypeScript ランタイム選定（tsc / tsx / Bun / Node + esbuild）** 🟡 *REQ-301 より*
  - 生成スクリプト（`pnpm spec:generate`）と検査スクリプト（`pnpm spec:check`）の実行方式
  - 候補: `tsx`（簡易・依存少）、`Bun`（高速）、`ts-node`（古典的）
  - 既存スタック（pnpm + Biome + secretlint）との親和性を考慮して選定
  - 関連要件: REQ-104, REQ-301

- [ ] **spec の Source of Truth レベル** 🟡 *REQ-401, REQ-402 より*
  - 「TS spec は追加レイヤ」と確定（ヒアリング Q1, Q8）したが、将来的に SoT 昇格する場合の互換性ポリシーをドキュメントに残すか確認
  - 関連要件: REQ-401, REQ-402

- [ ] **CLAUDE.md / AGENTS.md への Policy 追記タイミング** 🟡 *NFR-203 より*
  - Phase 1 完了時点で CLAUDE.md に「Tumigi Specification Policy」セクションを追加するか、別ファイル（AGENTS.md 新設）にするか
  - 関連要件: NFR-203

- [ ] **既存 commands/skills の TS 化スコープ範囲（30 件以上）** 🟡 *REQ-005, REQ-301 より*
  - Must Have の「代表コマンド・スキルの TS spec 記述」とは何件・どのコマンドを指すかを kairo-design で確定
  - 候補: Kairo 5 件 + Dev Skills 5 件 + DCS 5 件 + その他から代表抽出
  - 関連要件: REQ-005, REQ-301

---

## サマリー

| 優先度 | 件数 | 🔵 | 🟡 | 🔴 |
|--------|------|-----|-----|-----|
| 必須 | 2 | 2 | 0 | 0 |
| 推奨 | 3 | 3 | 0 | 0 |
| 確認事項 | 4 | 0 | 4 | 0 |
| **合計** | **9** | **5** | **4** | **0** |

## 関連文書

- **要件定義書**: [requirements.md](requirements.md)
- **ヒアリング記録**: [interview-record.md](interview-record.md)
- **ユーザストーリー**: [user-stories.md](user-stories.md)
- **受け入れ基準**: [acceptance-criteria.md](acceptance-criteria.md)
