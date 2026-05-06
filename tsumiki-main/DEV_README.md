# Dev Skills ドキュメント

## 全体概要

Dev Skillsは、プロジェクトのコンテキスト分析から実装計画・テストファースト実装・検証・デバッグ・Webテストまでをカバーする統合的な開発スキル群です。

### ワークフロー

```
dev-init（新規向け）  ─┐
                      ├→ docs/dev/context.md
dev-context（既存向け）┘
        │
        ▼
    dev-plan ──→ docs/dev/plans/<plan-name>/
        │
        ├──→ dev-impl ──→ dev-verify
        │        ↘ dev-debug（失敗時）
        │
        ├──→ dev-run（自動連続実装: impl→verify→debug）
        │
        ├──→ dev-screen-spec ──→ dev-webtest-plan ──→ dev-webtest
        │                                                ↘ dev-debug webtest（問題修正）
        │
        └──→ dev-navigate（やりたいことからスキルを選択）
```

### スキル一覧

| スキル | 説明 |
|-------|------|
| `dev-init` | 新規プロジェクトの技術スタック選定・初期化 |
| `dev-context` | 既存プロジェクトのコンテキスト自動分析 |
| `dev-navigate` | やりたいことから最適なスキルをナビゲーション |
| `dev-plan` | 要件をタスク分解して実装計画を作成 |
| `dev-impl` | TDDをガードレールとしたテストファースト実装 |
| `dev-run` | タスク範囲の自動連続実装（impl→verify→debug） |
| `dev-verify` | Plan単位のテスト・ビルド・Lint一括検証 |
| `dev-debug` | エラーカテゴリ別の診断・修正 |
| `dev-screen-spec` | ソースコードから画面仕様を自動生成・差分更新 |
| `dev-webtest-plan` | Playwright用Webテスト計画の生成・差分更新 |
| `dev-webtest` | Playwrightによる画面テスト実行 |

---

## スキル詳細

### 1. dev-init

**用途**
新規プロジェクトやまだ構成が固まっていないプロジェクトの技術スタックを対話的に決定し、コンテキストファイルを生成します。

**実行方法**
```
/tsumiki:dev-init
```

**特徴**:
- インタラクティブなヒアリングで技術スタックを決定
- `docs/dev/context.md` を生成（dev-context互換フォーマット）
- 承認制でプロジェクトスキャフォールディング（package.json、設定ファイル、ディレクトリ構造）も実行可能
- CLAUDE.mdの自動生成
- 信号機システムによる確信度表示（前工程指示/妥当な推測/AI推論補完）

**出力**:
- `docs/dev/context.md` - プロジェクトコンテキスト
- プロジェクトファイル群（承認後）

---

### 2. dev-context

**用途**
既存プロジェクトの技術スタック・テストフレームワーク・コーディング規約・アーキテクチャを自動分析し、コンテキストファイルを生成します。

**実行方法**
```
/tsumiki:dev-context
```

**特徴**:
- 4領域の並列探索（プロジェクトルート/テスト環境/ディレクトリ構造/コーディング規約）
- 500行以内のコンパクトなファイルに集約
- 後続スキル（dev-plan/dev-impl/dev-verify/dev-debug）の共通基盤として機能

**出力**:
- `docs/dev/context.md` - プロジェクトコンテキスト（技術スタック、テスト設定、ビルドコマンド、規約等）

---

### 3. dev-navigate

**用途**
やりたいことを対話で把握し、最適なtsumikiスキルの開始ポイントを提案します。

**実行方法**
```
/tsumiki:dev-navigate
```

**特徴**:
- 対話型のヒアリングでユーザーの目的を把握
- tsumikiスキル全体から最適なスキルと実行順序を提案
- 提案後、ユーザーの承認があればそのスキルを起動

---

### 4. dev-plan

**用途**
ユーザーの要件をインターフェースファースト設計とテスト可能なタスクに分解し、実装計画を作成します。

**実行方法**
```
# Lightweight モード（素早い計画）
/tsumiki:dev-plan auth "ユーザー認証機能を実装"

# PRDファイルを入力とする
/tsumiki:dev-plan auth ./docs/prd.md

# Full-spec モード（EARS要件定義付き）
# 実行時にモード選択
```

**実行モード**:
| モード | 説明 | 出力 |
|-------|------|------|
| Lightweight | 素早い要件明確化→設計→タスク分解 | plan.md + tasks/ |
| Full-spec | EARS要件定義→ユーザーストーリー→受入基準→設計→タスク分解 | requirements.md + user-stories.md + acceptance-criteria.md + plan.md + tasks/ |

**前提条件**: `docs/dev/context.md` が存在すること

**出力**: `docs/dev/plans/<plan-name>/`

---

### 5. dev-impl

**用途**
TDDをガードレールとして、テストを先に書いてから実装するテストファースト実装を行います。

**実行方法**
```
# 通常モード（Plan+タスク指定）
/tsumiki:dev-impl auth 001

# クイックモード（Plan不要の軽量修正）
/tsumiki:dev-impl "バリデーションメッセージを日本語に変更"
```

**実行モード**:
| モード | 引数 | 用途 |
|-------|------|------|
| 通常モード | `<plan-name> <task-id>` | Planのタスクを1つ実装 |
| クイックモード | `"修正指示"` | 軽量な修正・調整（Plan不要） |

**特徴**:
- 品質 > 速度 > トークン消費の優先順位
- インターフェースファーストでの最小限コンテキスト構築
- Red→Green→Refactorのフロー
- 失敗時はdev-debugに委譲

---

### 6. dev-run

**用途**
Plan内の指定範囲タスクをdev-impl/dev-verify/dev-debugのワークフローで自動連続実行します。

**実行方法**
```
/tsumiki:dev-run auth 001 005
```

**引数**:
- `plan-name`: 既存のPlan名
- `from-task-id`: 開始タスクID（例: "001"）
- `to-task-id`: 終了タスクID（例: "005"）

**特徴**:
- TaskCreate/TaskUpdateによる依存関係付き進捗管理
- 各タスクをサブエージェントに委託
- impl→verify→debugのループを自動実行

**前提条件**: `docs/dev/context.md` と `docs/dev/plans/<plan-name>/` が存在すること

---

### 7. dev-verify

**用途**
Plan単位で全タスクの完了状態とテスト・ビルド・Lintの整合性を検証し、レポートを出力します。

**実行方法**
```
/tsumiki:dev-verify auth
```

**特徴**:
- タスク完了状態のチェック
- 全テスト実行
- ビルド・Lint確認
- ファイルサイズチェック
- 検証レポートの出力

**出力**: `docs/dev/plans/<plan-name>/reports/`

---

### 8. dev-debug

**用途**
テスト失敗、ビルドエラー、環境問題など様々なエラーパターンをカテゴリ別に診断し、最小コンテキストで修正します。

**実行方法**
```
# 自動検出モード
/tsumiki:dev-debug

# 手動指定モード
/tsumiki:dev-debug "TypeError: Cannot read properties of undefined"

# webtestエラー修正モード
/tsumiki:dev-debug webtest
```

**エラーカテゴリ**:
| カテゴリ | 例 |
|---------|---|
| コンパイル/型エラー | 型不一致、missing import |
| テスト失敗 | assertion失敗、タイムアウト |
| ランタイムエラー | null参照、未処理例外 |
| 環境・設定 | 依存関係エラー、設定ミス |
| Lint/フォーマット | スタイル違反、未使用変数 |
| 依存関係 | バージョン競合、パッケージ不足 |
| webtestエラー | 視覚崩れ、a11y違反、レスポンシブ不備 |

---

### 9. dev-screen-spec

**用途**
ソースコードから画面仕様ドキュメントを自動生成・差分更新します。webtest計画の差分更新パイプラインで、ソースコード変更を画面単位の変更に翻訳する中間レイヤーとして機能します。

**実行方法**
```
# 自動判定（初回 or 差分更新）
/tsumiki:dev-screen-spec

# 強制的に初回生成
/tsumiki:dev-screen-spec init

# 差分更新（特定画面のみ）
/tsumiki:dev-screen-spec update login

# Planの受け入れ条件から事前生成
/tsumiki:dev-screen-spec from-plan auth
```

**出力**: `docs/dev/screen-specs/`

---

### 10. dev-webtest-plan

**用途**
dev-planの出力からPlaywright用のWebテスト計画ファイルを自動生成します。画面仕様の変更差分からテスト計画を更新する差分更新モードにも対応します。

**実行方法**
```
# 新規生成
/tsumiki:dev-webtest-plan auth

# 差分更新（全計画対象）
/tsumiki:dev-webtest-plan update

# 差分更新（特定計画のみ）
/tsumiki:dev-webtest-plan update auth
```

**前提条件**: `docs/dev/plans/<plan-name>/` が存在すること

**出力**: `docs/dev/webtests/plans/*.md`

---

### 11. dev-webtest

**用途**
Playwright CLIを使ってWebアプリケーションの動作確認・視覚テスト・アクセシビリティチェック等を実行し、検出した問題を記録します。

**実行方法**
```
# 計画テスト（Markdownテスト計画に沿って自動実行）
/tsumiki:dev-webtest auth

# 並列実行
/tsumiki:dev-webtest auth --parallel 3

# モンキーテスト
/tsumiki:dev-webtest monkey http://localhost:3000

# クイックチェック（単一ページ）
/tsumiki:dev-webtest check http://localhost:3000/login

# 再テスト（未解決エラーの再確認）
/tsumiki:dev-webtest retest
```

**実行モード**:
| モード | 引数 | 用途 |
|-------|------|------|
| 計画テスト | `<plan-name> [--parallel N]` | Markdownテスト計画に沿って自動テスト |
| モンキーテスト | `monkey <url>` | ランダム操作でエラー・崩れを検出 |
| クイックチェック | `check <url>` | 単一ページの視覚・アクセシビリティ確認 |
| プラン選択 | (引数なし) | 利用可能なプラン一覧から選択して実行 |
| 再テスト | `retest` | 未解決エラーの再現手順を再実行 |

**特徴**:
- Playwright CLI（メイン）/ Playwright MCP（フォールバック）
- スクリーンショット・スナップショットの保存
- 検出した問題はエラーディレクトリに記録、修正はdev-debugに委譲

---

## クイックスタート

### 新規プロジェクト

```bash
# 1. プロジェクト初期化
/tsumiki:dev-init

# 2. 実装計画
/tsumiki:dev-plan auth "ユーザー認証機能"

# 3. 自動実装
/tsumiki:dev-run auth 001 005

# 4. 検証
/tsumiki:dev-verify auth
```

### 既存プロジェクト

```bash
# 1. コンテキスト分析
/tsumiki:dev-context

# 2. 何をすべきかわからない場合
/tsumiki:dev-navigate

# 3. 実装計画
/tsumiki:dev-plan payment "決済機能の追加"

# 4. タスクごとに実装
/tsumiki:dev-impl payment 001

# 5. 検証
/tsumiki:dev-verify payment
```

### Webテストフロー

```bash
# 1. 画面仕様の生成
/tsumiki:dev-screen-spec

# 2. テスト計画の生成
/tsumiki:dev-webtest-plan auth

# 3. テスト実行
/tsumiki:dev-webtest auth

# 4. 問題があればデバッグ
/tsumiki:dev-debug webtest
```

---

## 出力ディレクトリ構造

```
docs/dev/
├── context.md                    # プロジェクトコンテキスト
├── plans/                        # 実装計画
│   └── <plan-name>/
│       ├── plan.md              # 計画概要
│       ├── tasks/               # タスクファイル
│       │   ├── 001_<name>.md
│       │   └── ...
│       └── reports/             # 検証レポート
├── screen-specs/                 # 画面仕様
│   └── <screen-id>.md
└── webtests/                     # Webテスト
    ├── plans/                   # テスト計画
    │   └── <plan-name>.md
    └── errors/                  # 検出されたエラー
```

## 注意事項

1. **前提条件**: ほとんどのスキルは `docs/dev/context.md` の存在を前提とします。最初に `dev-init` または `dev-context` を実行してください。
2. **Plan名**: 英数字とハイフンのみ使用可能です（日本語は自動変換されます）。
3. **品質優先**: dev-implは「品質 > 速度 > トークン消費」の優先順位で動作します。
