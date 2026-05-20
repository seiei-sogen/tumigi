# ts-spec-ast-search コンテキストノート

> Kairo 要件定義開始時に収集したコンテキスト情報

## 1. 要件の出典・たたき台

- **たたき台ファイル**: [docs/tatakidai/requirements-ast.md](../../../tatakidai/requirements-ast.md)
  - tumigi のコマンドが LLM 経由でドキュメントを検索する際、AST レベルの検索 (ast-grep, cocoindex-code `ccc search`) で効率化したい
  - markdown ではなく **TypeScript の object** で構造化したい
- **構造化スキーマの参考案**: [docs/tatakidai/chatgpt-idea-ts-object.md](../../../tatakidai/chatgpt-idea-ts-object.md)
  - `specs/**/*.ts` 配下に `Requirement / Workflow / Task / Decision / Verification` を TS object として定義
  - `defineRequirement()` 等の identity 関数で型補完
  - `implementedBy` は code symbol ではなく **commands/*.md / skills/* / docs/** / .claude-plugin / 出力ディレクトリ**を指す
  - ast-grep で `area: "kairo"` のように構造ベース検索が可能
  - `pnpm spec:check:commands` / `pnpm spec:check:docs` でドキュメント整合性検査

## 2. プロジェクト基本情報

| 項目 | 内容 |
|------|------|
| プロジェクト名 | tumigi |
| 概要 | AI 駆動開発フレームワーク (Claude Code Plugin) |
| バージョン (package.json) | 0.0.7 |
| バージョン (plugin.json) | 1.4.1 |
| Package Manager | pnpm@10.13.1 |
| ライセンス | MIT |

## 3. 技術スタック

- **言語**: 主に Markdown（コマンドファイル）。Node.js 系の dev tooling のみ存在
- **dev tooling**: `secretlint`, `@biomejs/biome`, `simple-git-hooks`
- **テスト/ビルド**: なし（コマンド定義中心リポジトリ）
- **TypeScript**: 現状未導入（**新規導入が要件のスコープ**）

## 4. リポジトリ構成（要件に直結する範囲）

```
tumigi/
├── .claude-plugin/         # plugin.json, marketplace.json
├── commands/               # 31 ファイル（kairo, tdd, dcs, utility, rev）
├── skills/                 # 14 SKILL.md（dev-*, ipa-*, kairo-implement）
├── agents/                 # （現状ファイルなし）
├── book/                   # 利用ガイド・ケーススタディ
├── docs/
│   ├── external-sources.md # 外部情報源リスト
│   └── tatakidai/          # 本要件のたたき台
└── package.json
```

### 既存コマンド一覧 (31)

`commands/*.md`:
- kairo: init-tech-stack, kairo-requirements, kairo-design, kairo-tasks, kairo-loop, kairo-tasknote
- tdd: tdd-requirements, tdd-testcases, tdd-red, tdd-green, tdd-refactor, tdd-verify-complete, tdd-todo, tdd-tasknote
- direct: direct-setup, direct-verify
- rev: rev-tasks, rev-design, rev-specs, rev-requirements
- dcs/: feature-rubber-duck, sequence-diagram-analysis, state-transition-analysis, impact-analysis, incremental-dev, bug-analysis, performance-analysis
- ユーティリティ: help, orchestrate, refine-plan, refine-execute, auto-debug, build-fix, env-fix, flaky-fix, timeout-fix, test-optimization-patterns, tech-stack

### 既存スキル一覧 (14)

`skills/*/SKILL.md`:
- dev-*: dev-context, dev-init, dev-plan, dev-impl, dev-run, dev-verify, dev-debug, dev-navigate, dev-screen-spec, dev-webtest-plan, dev-webtest
- ipa-*: ipa-security-check, ipa-security-guide
- kairo-implement

## 5. 出力契約（既存）

- **Kairo**: `docs/tumigidoc/{要件名}/spec/`（本ファイルの所属）
- **Dev Skills**: `docs/dev/context.md`, `docs/dev/plans/{planName}/`
- **DCS**: `.dcs/{timestamp}_{targetName}/`

## 6. 開発ルール

- CLAUDE.md: pnpm secretlint を pre-commit で実行。機密情報チェック必須
- 出力ファイルパスはプロジェクトルート基準の相対パス
- 信頼性レベル（🔵🟡🔴）を要件・テストケースごとに記載

## 7. 外部情報源（参考）

- ast-grep: https://github.com/ast-grep/ast-grep
- cocoindex-code: https://github.com/cocoindex-io/cocoindex-code
  - `ccc search {{ARGS}}` で AST レベル検索

## 8. 注意事項・既知の制約

- TypeScript ベースの spec を新設する場合、tsc / type-check / lint パイプラインの新設が必要
- 既存の `commands/*.md` を spec から「指し示す」設計のため、両者の **整合性検査** が必須
- rulesync 経由で Claude Code 以外 (cursor, geminicli, codexcli, roo) にも展開している関係上、spec/検査スクリプトが各ツール環境に依存しないこと
- AST 検索ツールの導入は **必須/推奨/任意** かを要件で明確化する必要あり
- 既存ユーザの learning curve（markdown → TS object）への配慮

## 9. ギャップ・未確定事項（ヒアリング候補）

1. **既存 Markdown の扱い**: 完全置換か、TS spec と Markdown を併存させるか
2. **AST 検索ツールの選定**: ast-grep のみ / cocoindex-code のみ / 両方サポート
3. **検査スクリプトの範囲**: コマンド構造検査・出力契約検査・README 整合性検査をどこまで含めるか
4. **対象範囲**: commands / skills / agents / book / docs のうちどれを TS object 化するか
5. **CI 連携**: pre-commit / GitHub Actions での spec チェック自動化の要否
6. **rulesync 互換性**: TS spec を rulesync が import/generate 可能な形式に変換する必要があるか
7. **段階導入**: 一括移行 vs 段階的移行
