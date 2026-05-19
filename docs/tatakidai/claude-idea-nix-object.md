# Tumigi向け Nix Expression 仕様定義案

> `docs/tatakidai/chatgpt-idea-ts-object.md` のTypeScript object版を、
> **Nix expression（NixOS module-style）** に置き換えた版。
>
> 目的は同じ。Markdownでは検索効率が悪いので、要件・ワークフロー・タスク・検証を
> **構造化データ**として持ち、`ast-grep` / `cocoindex-code (ccc)` で
> AST/意味レベルの検索を可能にする。

---

## 0. なぜNixか（TypeScript版との比較）

| 観点 | TypeScript object | Nix expression |
|---|---|---|
| 値の表現 | `as const` + interface | attribute set + `lib.types` |
| スキーマ検査 | `tsc` / Biome | `lib.evalModules` (NixOS module system) |
| 必要ツールチェイン | Node + TS + tsx | `nix` だけ |
| 機械可読出力 | TSCで`.d.ts`+JSON生成 | `nix eval --json` 一発 |
| バージョン固定 | package.json | flake.lock（搬送する側もろとも固定） |
| 検索ツール配布 | 各自で入れる | `nix develop` で全員に降ってくる |
| ast-grep対応 | `--lang ts` 標準サポート | `customLanguages.nix` 登録が必要（後述） |
| LLMの慣れ | 高い | TSより低いが、attrsetの構文は単純で読みやすい |
| 不純な副作用 | 書こうと思えば書ける | 純粋関数。仕様ファイルが副作用を持てない |

**Nixを採用する主な根拠**は次の3つ。

1. **toolchain と仕様の一体配布**：
   `flake.nix` に `ast-grep` / `cocoindex-code` / `ripgrep` / `jq` を `devShell`
   として書いておけば、Coding Agent が `nix develop -c sg run ...` するだけで
   検索環境が成立する。バージョンずれが起きない。
2. **スキーマがそのまま型検査になる**：
   NixOS module system (`lib.evalModules`) を使えば、`requirement` モジュールに
   `lib.types.enum`, `lib.types.listOf`, `lib.types.submodule` でフィールド定義が
   できる。書いた時点で構造違反が `nix flake check` で落ちる。
3. **副作用のない正本**：
   仕様ファイルは式（値）であって、IOを持てない。
   仕様の評価結果は決定的で、git diff で意味的な変更だけが見える。

逆に注意すべきは次の2点。

- ast-grep の **Nix 用 tree-sitter ライブラリは標準同梱されていない**ので、
  `tree-sitter-nix.so` を flake で用意して `sgconfig.yml` に登録する手間がある。
- LLM は Nix より TS のほうが慣れていることが多い。**attrset と let-in と
  module system だけ**に限定し、関数定義は最小限に抑えると LLM 可読性が上がる。

---

## 1. ディレクトリ構成

```txt
specs/
  flake.nix                     # devShell + 評価ターゲット
  flake.lock
  sgconfig.yml                  # ast-grep customLanguages.nix を定義
  default.nix                   # specs全体を集約してattrsetで返す

  _schema/
    types.nix                   # lib.types ベースのモジュール定義
    ids.nix                     # ID用の正規表現/型
    define.nix                  # 各仕様ノードを返すヘルパー

  product/
    tumigi-product.nix

  commands/
    kairo.nix
    tdd.nix
    dev-skills.nix
    dcs.nix
    utility.nix
    reverse.nix

  workflows/
    kairo-workflow.nix
    dev-workflow.nix
    dcs-workflow.nix

  artifacts/
    tumigidoc.nix
    dev-docs.nix
    dcs-output.nix

  compatibility/
    claude-code-plugin.nix
    rulesync.nix

  quality/
    command-quality.nix
    documentation-quality.nix
```

ポイントは TS 版と同じ。**`implementedBy` が指すのは src/**ではなく、
`commands/*.md` / `skills/*` / `docs/**` / `.claude-plugin` などの
**コマンド・Skill・出力契約**。

---

## 2. ID 設計

ID は単なる文字列だが、誤った文字列が混ざらないよう
`_schema/ids.nix` で正規表現ベースの型を作る。

```nix
# specs/_schema/ids.nix
{ lib }:
let
  mkIdType = name: regex: lib.types.strMatching regex // {
    description = "${name} ID (${regex})";
  };
in
{
  tumigiProductRequirementId =
    mkIdType "TumigiProductRequirement" "^REQ-TUMIGI-[0-9]+$";

  commandRequirementId =
    mkIdType "CommandRequirement" "^REQ-CMD-[A-Z][A-Z0-9-]*-[0-9]+$";

  skillRequirementId =
    mkIdType "SkillRequirement" "^REQ-SKILL-[A-Z][A-Z0-9-]*-[0-9]+$";

  workflowRequirementId =
    mkIdType "WorkflowRequirement" "^REQ-WF-[A-Z][A-Z0-9-]*-[0-9]+$";

  artifactRequirementId =
    mkIdType "ArtifactRequirement" "^REQ-ARTIFACT-[A-Z][A-Z0-9-]*-[0-9]+$";

  compatibilityRequirementId =
    mkIdType "CompatibilityRequirement" "^REQ-COMPAT-[A-Z][A-Z0-9-]*-[0-9]+$";

  qualityRequirementId =
    mkIdType "QualityRequirement" "^REQ-QUALITY-[A-Z][A-Z0-9-]*-[0-9]+$";

  # 統合
  tumigiRequirementId =
    mkIdType "TumigiRequirement"
      "^REQ-(TUMIGI|CMD|SKILL|WF|ARTIFACT|COMPAT|QUALITY)-[A-Z0-9-]*[0-9]+$";

  tumigiPlanId = mkIdType "Plan" "^PLAN-[A-Z][A-Z0-9-]*-[0-9]+$";
  tumigiTaskId = mkIdType "Task" "^TASK-[A-Z][A-Z0-9-]*-[0-9]+$";
  tumigiDecisionId = mkIdType "Decision" "^DEC-[A-Z][A-Z0-9-]*-[0-9]+$";
  tumigiVerificationId = mkIdType "Verification" "^VERIFY-[A-Z][A-Z0-9-]*-[0-9]+$";
}
```

TS 版の template literal type と同じ役割を `strMatching` で実現する。
`evalModules` 時に正規表現に合わない ID は弾かれる。

---

## 3. 型定義（NixOS module system）

```nix
# specs/_schema/types.nix
{ lib }:
let
  ids = import ./ids.nix { inherit lib; };
  inherit (lib) types mkOption;

  area = types.enum [
    "product" "kairo" "tdd" "dev-skills" "dcs" "utility"
    "reverse-engineering" "claude-plugin" "rulesync"
    "documentation" "quality"
  ];

  requirementStatus = types.enum [
    "draft" "accepted" "implemented" "verified" "deprecated"
  ];

  priority = types.enum [ "must" "should" "could" "wont" ];

  artifactKind = types.enum [
    "command" "skill" "doc" "plugin-config"
    "output-template" "workflow" "test" "script"
  ];

  artifactRef = types.submodule {
    options = {
      path = mkOption { type = types.str; description = "リポジトリルートからの相対パス"; };
      kind = mkOption { type = artifactKind; };
    };
  };

  commandRef = types.submodule {
    options = {
      name = mkOption { type = types.str; };
      path = mkOption {
        type = types.strMatching "^(commands/|\\.claude/commands/).*\\.(md|yml)$";
      };
      prefix = mkOption {
        type = types.nullOr (types.enum [ "/tumigi:" ]);
        default = null;
      };
    };
  };

  skillRef = types.submodule {
    options = {
      name = mkOption { type = types.str; };
      path = mkOption { type = types.strMatching "^skills/.+$"; };
    };
  };

  outputContract = types.submodule {
    options = {
      pathPattern = mkOption { type = types.str; };
      description = mkOption { type = types.str; };
      requiredFiles = mkOption {
        type = types.listOf types.str;
        default = [ ];
      };
    };
  };
in
rec {
  inherit ids area requirementStatus priority
          artifactKind artifactRef commandRef skillRef outputContract;

  # Requirement 本体
  requirement = types.submodule {
    options = {
      kind = mkOption { type = types.enum [ "requirement" ]; default = "requirement"; };
      id = mkOption { type = ids.tumigiRequirementId; };
      area = mkOption { type = area; };
      title = mkOption { type = types.str; };
      status = mkOption { type = requirementStatus; };
      priority = mkOption { type = priority; };

      description = mkOption { type = types.str; };
      rationale = mkOption { type = types.nullOr types.str; default = null; };
      ears = mkOption { type = types.listOf types.str; default = [ ]; };

      dependsOn = mkOption {
        type = types.listOf ids.tumigiRequirementId;
        default = [ ];
      };
      relatedCommands = mkOption { type = types.listOf commandRef; default = [ ]; };
      relatedSkills = mkOption { type = types.listOf skillRef; default = [ ]; };
      outputContracts = mkOption { type = types.listOf outputContract; default = [ ]; };

      implementedBy = mkOption { type = types.listOf artifactRef; default = [ ]; };
      verifiedBy = mkOption {
        type = types.listOf ids.tumigiVerificationId;
        default = [ ];
      };

      tags = mkOption { type = types.listOf types.str; default = [ ]; };
    };
  };

  workflow = types.submodule {
    options = {
      kind = mkOption { type = types.enum [ "workflow" ]; default = "workflow"; };
      id = mkOption { type = ids.tumigiPlanId; };
      area = mkOption { type = area; };
      title = mkOption { type = types.str; };
      status = mkOption { type = requirementStatus; };

      description = mkOption { type = types.str; };
      implements = mkOption { type = types.listOf ids.tumigiRequirementId; };

      steps = mkOption {
        type = types.listOf (types.submodule {
          options = {
            order = mkOption { type = types.int; };
            command = mkOption { type = types.str; };
            input = mkOption { type = types.nullOr types.str; default = null; };
            output = mkOption { type = types.nullOr types.str; default = null; };
            required = mkOption { type = types.bool; default = true; };
          };
        });
      };

      artifacts = mkOption { type = types.listOf artifactRef; default = [ ]; };
      tags = mkOption { type = types.listOf types.str; default = [ ]; };
    };
  };

  task = types.submodule {
    options = {
      kind = mkOption { type = types.enum [ "task" ]; default = "task"; };
      id = mkOption { type = ids.tumigiTaskId; };
      title = mkOption { type = types.str; };
      status = mkOption { type = requirementStatus; };
      area = mkOption { type = area; };

      description = mkOption { type = types.str; };
      implements = mkOption { type = types.listOf ids.tumigiRequirementId; };
      touches = mkOption { type = types.listOf artifactRef; };
      verification = mkOption {
        type = types.listOf ids.tumigiVerificationId;
        default = [ ];
      };
      tags = mkOption { type = types.listOf types.str; default = [ ]; };
    };
  };

  decision = types.submodule {
    options = {
      kind = mkOption { type = types.enum [ "decision" ]; default = "decision"; };
      id = mkOption { type = ids.tumigiDecisionId; };
      title = mkOption { type = types.str; };
      status = mkOption {
        type = types.enum [ "proposed" "accepted" "superseded" "rejected" ];
      };
      context = mkOption { type = types.str; };
      decision = mkOption { type = types.str; };
      consequences = mkOption { type = types.str; };
      affects = mkOption { type = types.listOf ids.tumigiRequirementId; };
      tags = mkOption { type = types.listOf types.str; default = [ ]; };
    };
  };

  verification = types.submodule {
    options = {
      kind = mkOption { type = types.enum [ "verification" ]; default = "verification"; };
      id = mkOption { type = ids.tumigiVerificationId; };
      title = mkOption { type = types.str; };
      status = mkOption { type = requirementStatus; };

      verifies = mkOption { type = types.listOf ids.tumigiRequirementId; };
      method = mkOption {
        type = types.enum [
          "manual" "snapshot" "secretlint"
          "command-structure-check" "docs-output-check"
          "agent-simulation" "rulesync-generation"
        ];
      };
      command = mkOption { type = types.nullOr types.str; default = null; };
      expectedArtifacts = mkOption { type = types.listOf artifactRef; default = [ ]; };
      tags = mkOption { type = types.listOf types.str; default = [ ]; };
    };
  };
}
```

`types.submodule` を使うことで「未知のフィールド」はエラーになり、
TS の `as const` よりも厳しい検査が無料でついてくる。

---

## 4. define ヘルパー

TS版の `defineRequirement<T>(v: T) => v` 相当。
ここでは「型を付けたまま値を返す」必要はないので、単純なパススルー＋
モジュール評価を1ファイル内で完結させるヘルパーだけ用意する。

```nix
# specs/_schema/define.nix
{ lib }:
let
  schemaTypes = import ./types.nix { inherit lib; };

  evalAs = optionType: value:
    let
      result = lib.evalModules {
        modules = [
          ({ ... }: {
            options.value = lib.mkOption { type = optionType; };
            config.value = value;
          })
        ];
      };
    in result.config.value;
in
{
  defineRequirement = v: evalAs schemaTypes.requirement v;
  defineWorkflow    = v: evalAs schemaTypes.workflow v;
  defineTask        = v: evalAs schemaTypes.task v;
  defineDecision    = v: evalAs schemaTypes.decision v;
  defineVerification = v: evalAs schemaTypes.verification v;
}
```

これで個別 `.nix` ファイルが評価された瞬間に型違反が落ちる。
（ファイルを `nix eval` で評価せずに `import` するだけでもエラーになる）

---

## 5. 製品全体の要件

```nix
# specs/product/tumigi-product.nix
{ lib }:
let define = import ../_schema/define.nix { inherit lib; };
in {
  reqTumigiProduct001 = define.defineRequirement {
    id = "REQ-TUMIGI-001";
    area = "product";
    title = "TumigiはAI駆動開発の一連の工程をコマンドとして支援できる";
    status = "accepted";
    priority = "must";

    description = ''
      Tumigiは、要件定義、設計、タスク分割、実装、検証、デバッグ、リバースエンジニアリングを、
      Claude Codeを中心としたAIコーディング環境で実行できるようにする。
    '';

    ears = [
      "When ユーザーが新規機能開発を開始する, the system shall KairoまたはDev Skillsの開始コマンドを提示できる。"
      "When ユーザーが既存コードを調査したい, the system shall DCSまたはリバースエンジニアリング系コマンドを提示できる。"
      "When ユーザーがClaude Code PluginとしてTumigiを導入する, the system shall /tumigi: プレフィックス付きコマンドを利用できる。"
    ];

    implementedBy = [
      { kind = "doc"; path = "README.md"; }
      { kind = "plugin-config"; path = ".claude-plugin"; }
      { kind = "command"; path = "commands/help.md"; }
    ];

    tags = [ "tumigi" "product" "ai-driven-development" ];
  };
}
```

`''...''` のヒアドキュメント、`[ ... ]` のリスト、`{ ... }` の attrset、
セミコロン区切り。要素は **改行で並べる**だけでコンマ不要なのが Nix 流儀。

---

## 6. Kairo 要件

```nix
# specs/commands/kairo.nix
{ lib }:
let define = import ../_schema/define.nix { inherit lib; };
in {
  reqCmdKairo001 = define.defineRequirement {
    id = "REQ-CMD-KAIRO-001";
    area = "kairo";
    title = "Kairoは要件定義から実装までの包括的な開発フローを提供する";
    status = "accepted";
    priority = "must";

    description = ''
      Kairoは、技術スタック特定、要件定義、設計文書生成、タスク分割、実装実行、
      自動連続実装を一連の開発フローとして提供する。
    '';

    ears = [
      "When ユーザーがKairo開発を開始する, the system shall init-tech-stack, kairo-requirements, kairo-design, kairo-tasks, kairo-loop の順序を提示できる。"
      "When ユーザーが要件定義を実行する, the system shall EARS記法を用いて受け入れ基準を含む要件定義書を作成する。"
      "When ユーザーが実装を開始する, the system shall kairo-implement または kairo-loop によって実装に進める。"
    ];

    relatedCommands = [
      { name = "init-tech-stack";    path = "commands/init-tech-stack.md";    prefix = "/tumigi:"; }
      { name = "kairo-requirements"; path = "commands/kairo-requirements.md"; prefix = "/tumigi:"; }
      { name = "kairo-design";       path = "commands/kairo-design.md";       prefix = "/tumigi:"; }
      { name = "kairo-tasks";        path = "commands/kairo-tasks.md";        prefix = "/tumigi:"; }
      { name = "kairo-loop";         path = "commands/kairo-loop.md";         prefix = "/tumigi:"; }
    ];

    outputContracts = [
      {
        pathPattern = "docs/tumigidoc/{requirementName}/spec/";
        description = "Kairo要件定義・設計・タスク分割の成果物を格納する。";
        requiredFiles = [ "requirements.md" "interview-record.md" ];
      }
    ];

    implementedBy = [
      { kind = "command"; path = "commands/kairo-requirements.md"; }
      { kind = "command"; path = "commands/kairo-design.md"; }
      { kind = "command"; path = "commands/kairo-tasks.md"; }
      { kind = "command"; path = "commands/kairo-loop.md"; }
    ];

    tags = [ "kairo" "requirements" "design" "tasks" "implementation" ];
  };
}
```

---

## 7. Dev Skills / DCS / 互換要件

骨子は TS 版（`chatgpt-idea-ts-object.md` §7-9）と同じなので、
Nix 構文への置き換えだけ抜粋。

```nix
# specs/commands/dev-skills.nix
{ lib }:
let define = import ../_schema/define.nix { inherit lib; };
in {
  reqSkillDev001 = define.defineRequirement {
    id = "REQ-SKILL-DEV-001";
    area = "dev-skills";
    title = "Dev Skillsはコンテキスト分析から計画・実装・検証・デバッグまでを統合的に支援する";
    status = "accepted";
    priority = "must";

    description = ''
      Dev Skillsは、新規または既存プロジェクトのコンテキストを分析し、
      実装計画、テストファースト実装、検証、デバッグ、画面仕様生成、Webテストまでを扱う。
    '';

    ears = [
      "When 既存プロジェクトでDev Skillsを使う, the system shall dev-context により docs/dev/context.md を生成する。"
      "When ユーザーが機能名と要件を指定する, the system shall dev-plan により docs/dev/plans/{planName}/ を生成する。"
      "When ユーザーがPlan内のタスクを実装する, the system shall dev-impl または dev-run によりテストファースト実装を行う。"
      "When 実装完了後, the system shall dev-verify によりテスト・ビルド・Lintを検証する。"
    ];

    relatedSkills = [
      { name = "dev-context"; path = "skills/dev-context"; }
      { name = "dev-plan";    path = "skills/dev-plan"; }
      { name = "dev-impl";    path = "skills/dev-impl"; }
      { name = "dev-run";     path = "skills/dev-run"; }
      { name = "dev-verify";  path = "skills/dev-verify"; }
      { name = "dev-debug";   path = "skills/dev-debug"; }
    ];

    outputContracts = [
      {
        pathPattern = "docs/dev/context.md";
        description = "プロジェクトコンテキスト。Dev Skillsの多くが前提とする。";
      }
      {
        pathPattern = "docs/dev/plans/{planName}/";
        description = "実装計画、タスク、検証レポートを格納する。";
        requiredFiles = [ "plan.md" ];
      }
    ];

    tags = [ "dev-skills" "context" "plan" "implementation" "verification" ];
  };
}
```

DCS 系・compatibility 系は TS 版を1対1で置き換えるだけなので省略。

---

## 8. Workflow 定義

```nix
# specs/workflows/kairo-workflow.nix
{ lib }:
let define = import ../_schema/define.nix { inherit lib; };
in {
  workflowKairo001 = define.defineWorkflow {
    id = "PLAN-KAIRO-001";
    area = "kairo";
    title = "Kairo包括的開発フロー";
    status = "accepted";

    description = ''
      Kairoは、技術スタック初期化、要件定義、設計、タスク分割、実装を順に進める。
    '';

    implements = [ "REQ-CMD-KAIRO-001" ];

    steps = [
      { order = 1; command = "/tumigi:init-tech-stack";    output = "技術スタック情報"; }
      { order = 2; command = "/tumigi:kairo-requirements"; output = "docs/tumigidoc/{requirementName}/spec/requirements.md"; }
      { order = 3; command = "/tumigi:kairo-design";       output = "設計文書"; }
      { order = 4; command = "/tumigi:kairo-tasks";        output = "タスク分割"; }
      { order = 5; command = "/tumigi:kairo-loop";         output = "実装結果"; }
    ];

    artifacts = [
      { kind = "command"; path = "commands/init-tech-stack.md"; }
      { kind = "command"; path = "commands/kairo-requirements.md"; }
      { kind = "command"; path = "commands/kairo-design.md"; }
      { kind = "command"; path = "commands/kairo-tasks.md"; }
      { kind = "command"; path = "commands/kairo-loop.md"; }
    ];

    tags = [ "kairo" "workflow" ];
  };
}
```

---

## 9. 集約：`specs/default.nix`

仕様ノードを全部 import して、1つの attrset として返す。
ここを `nix eval --json` するだけで全仕様が JSON 化できる。

```nix
# specs/default.nix
{ pkgs ? import <nixpkgs> { }, lib ? pkgs.lib }:
let
  product       = import ./product/tumigi-product.nix       { inherit lib; };
  kairo         = import ./commands/kairo.nix               { inherit lib; };
  devSkills     = import ./commands/dev-skills.nix          { inherit lib; };
  dcs           = import ./commands/dcs.nix                 { inherit lib; };
  pluginCompat  = import ./compatibility/claude-code-plugin.nix { inherit lib; };
  rulesyncCompat = import ./compatibility/rulesync.nix      { inherit lib; };
  kairoWf       = import ./workflows/kairo-workflow.nix     { inherit lib; };
  devWf         = import ./workflows/dev-workflow.nix       { inherit lib; };
  cmdQuality    = import ./quality/command-quality.nix      { inherit lib; };
  docsQuality   = import ./quality/documentation-quality.nix { inherit lib; };
in
{
  requirements =
    product // kairo // devSkills // dcs // pluginCompat // rulesyncCompat;

  workflows = kairoWf // devWf;
  verifications = cmdQuality // docsQuality;

  # フラット化したリスト形式（ast-grep ではなく cocoindex から使う用）
  allRequirementsList = lib.attrValues (product // kairo // devSkills // dcs);
  allWorkflowsList    = lib.attrValues (kairoWf // devWf);
}
```

---

## 10. flake.nix（toolchain 同梱）

```nix
# specs/flake.nix
{
  description = "Tumigi spec store with AST-search toolchain";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    flake-utils.url = "github:numtide/flake-utils";
  };

  outputs = { self, nixpkgs, flake-utils }:
    flake-utils.lib.eachDefaultSystem (system:
      let
        pkgs = import nixpkgs { inherit system; };

        # tree-sitter-nix の .so を ast-grep の sgconfig.yml から参照する。
        tsNix = pkgs.tree-sitter-grammars.tree-sitter-nix;

        # nix-on-nix な仕様評価ターゲット
        specs = import ./default.nix { inherit pkgs; };

        # 仕様を JSON で書き出す
        specsJson = pkgs.runCommand "tumigi-specs.json" { } ''
          ${pkgs.nix}/bin/nix eval --json --impure --expr \
            '(import ${./.}/default.nix { pkgs = import ${nixpkgs} {}; })' > $out
        '';
      in
      {
        packages = {
          inherit specsJson;
          default = specsJson;
        };

        # nix develop で全部入り
        devShells.default = pkgs.mkShell {
          packages = with pkgs; [
            ast-grep
            ripgrep
            jq
            nix
            # cocoindex-code は nixpkgs にまだ無いので
            # uv で venv を作るラッパースクリプトを別途用意するか、
            # poetry2nix / uv2nix を使う想定。
          ];
          shellHook = ''
            export TUMIGI_SPECS_DIR=$PWD/specs
            export TUMIGI_TREE_SITTER_NIX=${tsNix}/parser
            echo "tumigi specs shell ready. Try: sg run -p 'area = \"kairo\"' specs"
          '';
        };

        # nix flake check で仕様の型検査
        checks.specs-eval = pkgs.runCommand "specs-eval-check" { } ''
          ${pkgs.nix}/bin/nix eval --json --impure --expr \
            '(import ${./.}/default.nix { pkgs = import ${nixpkgs} {}; })' > /dev/null
          touch $out
        '';
      });
}
```

cocoindex-code（`ccc`）が現時点で nixpkgs に無い場合の選択肢：

- `pkgs.python3.withPackages` + `pip install cocoindex-code` をラップ
- `uv2nix` で `pyproject.toml` を取り込んで純Nixビルド
- 一旦 flake の `devShells` に `pkgs.uv` だけ入れて、shellHook で
  `uv tool install cocoindex-code` を idempotent に呼ぶ

ここは README に「最初の `nix develop` で `uv tool install` も走らせる」と
明記する運用が現実的。

---

## 11. sgconfig.yml（ast-grep に Nix を教える）

```yaml
# specs/sgconfig.yml
customLanguages:
  nix:
    libraryPath: ${TUMIGI_TREE_SITTER_NIX}.so
    extensions: [ nix ]
    expandoChar: _
```

shellHook で `TUMIGI_TREE_SITTER_NIX` を流し込んでいるので、
`nix develop` 内に入れば `sg run --lang nix -p ...` がそのまま動く。

---

## 12. ast-grep 検索例

```bash
# Kairo 関連要件を AST レベルで探す
sg run --lang nix -p 'area = "kairo"' specs

# 特定コマンドに紐づく要件を探す
sg run --lang nix -p 'name = "kairo-requirements"' specs

# docs/tumigidoc 出力契約を持つ仕様を探す
sg run --lang nix -p 'pathPattern = "docs/tumigidoc/$_/spec/"' specs

# Dev Skills 関連要件
sg run --lang nix -p 'area = "dev-skills"' specs

# DCS 関連要件
sg run --lang nix -p 'area = "dcs"' specs

# 「mustだけど未実装」みたいな複合検索は YAML rule で書く
sg scan -c specs/sgrules/must-but-draft.yml specs
```

`specs/sgrules/must-but-draft.yml` の例：

```yaml
id: must-but-draft
language: nix
severity: warning
rule:
  all:
    - has:
        pattern: 'priority = "must"'
    - has:
        pattern: 'status = "draft"'
message: must な要件が draft のまま残っている
```

---

## 13. cocoindex-code (ccc) との連携

ccc は AST より上位の **意味検索**。
Nix の attrset と heredoc を読ませる方法は2つある。

**A. `.nix` ファイルを直接インデックス**
ccc に Nix 言語のチャンカが無くてもファイル単位で索引できるので、
`description` / `ears` などの自然言語フィールドは普通に検索ヒットする。

```bash
ccc index specs/
ccc search "要件名のディレクトリを作るのはどのコマンドか"
```

**B. JSON にして自然言語フィールドだけ抜き出す**
`nix eval --json` で書き出した JSON を ccc に食わせる前処理を入れる。
`jq` でフラットなドキュメント風に直す。

```bash
nix eval --json --impure --expr \
  '(import ./specs).requirements' \
  | jq -r 'to_entries[] | "# \(.value.id) \(.value.title)\n\(.value.description)\n\nEARS:\n\(.value.ears | join("\n"))"' \
  > .cache/specs-flat.md

ccc index .cache/specs-flat.md
```

つまり、

- **AST-grep**：構造クエリ（field 名・enum 値）に強い → 機械的な参照解決に使う
- **ccc**：自然言語クエリに強い → 「あの要件どこだっけ」に使う

の二段構えにする。

---

## 14. Coding Agent 用エントリポイント

LLM/Agent から呼びやすいよう、頻出クエリを `Justfile` か `flake apps`
として封じ込める。

```nix
# flake.nix の outputs に追加
apps.specs-find = {
  type = "app";
  program = "${pkgs.writeShellScript "specs-find" ''
    set -eu
    case "$1" in
      area)         shift; ${pkgs.ast-grep}/bin/sg run --lang nix -p "area = \"$1\"" specs ;;
      command)      shift; ${pkgs.ast-grep}/bin/sg run --lang nix -p "name = \"$1\"" specs ;;
      requirement)  shift; ${pkgs.ast-grep}/bin/sg run --lang nix -p "id = \"$1\"" specs ;;
      *) echo "usage: specs-find {area|command|requirement} <value>"; exit 2 ;;
    esac
  ''}";
};
```

これで Agent は

```bash
nix run .#specs-find -- area kairo
nix run .#specs-find -- command kairo-requirements
nix run .#specs-find -- requirement REQ-CMD-KAIRO-001
```

の3つだけ覚えれば、ほぼ全ての参照系クエリを賄える。

---

## 15. CLAUDE.md / AGENTS.md に追記する運用

```md
# Tumigi Specification Policy (Nix版)

Tumigiの正本仕様は `specs/**/*.nix` に Nix expression として置く。
編集・参照する前に必ず次を守る。

## 仕様の探索

1. **構造クエリ**は `sg run --lang nix` または `nix run .#specs-find` を使う。
2. **自然言語クエリ**は `ccc search` を使う。
3. **生 grep は使わない**。フィールド名と値のペアは AST のほうが正確。

## 仕様の更新

1. `specs/**/*.nix` を編集したら必ず `nix flake check` を通す。
2. ID は `_schema/ids.nix` の正規表現に従う。
3. `implementedBy` には `commands/*.md` / `skills/*` / `docs/**` を書く。
   実コードのシンボルは書かない（Tumigi は Markdown 中心のリポジトリ）。
4. `outputContracts.pathPattern` を変えたら、対応する commands/*.md の
   出力先記述も同じ commit で直す。

## 出力契約の表

| Area | pathPattern |
|---|---|
| Kairo | `docs/tumigidoc/{要件名}/spec/` |
| Dev Skills | `docs/dev/context.md`, `docs/dev/plans/{planName}/` |
| DCS | `.dcs/{timestamp}_{targetName}/` |

## 禁止事項

- `specs/**/*.nix` を読まずに README/commands/*.md を編集する
- `nix flake check` をスキップして commit する
- 出力ディレクトリを変えたのに `outputContracts` を更新しない
```

---

## 16. Markdown 併存と移行

いきなり Markdown を捨てるとレビューしづらいので、**Nix が正本、Markdown は
ビューア**という構成を推奨。

```txt
specs/**/*.nix              ← 正本
↓ nix run .#specs-render
docs/tumigidoc/_generated/  ← Nix から生成された Markdown（読み物用）
```

ジェネレータは Nix から JSON を吐いて、軽い Markdown テンプレートを当てるだけ
（`pkgs.writeShellApplication` + `jq` で十分）。

CI / pre-commit では次の3つを回せばよい。

1. `nix flake check` … 型検査
2. `nix run .#specs-render` … 生成 Markdown と git tree の差分を `git diff --exit-code`
3. `pnpm secretlint` … 既存のシークレット検査

---

## 17. TS 版との比較まとめ

| 項目 | TS版 (chatgpt-idea) | Nix版 (this) |
|---|---|---|
| 正本ファイル | `specs/**/*.ts` | `specs/**/*.nix` |
| 型検査 | `tsc --noEmit` | `nix flake check`（モジュール評価） |
| JSON出力 | `tsx` で関数を呼ぶ | `nix eval --json` 一発 |
| ast-grep | 標準対応 (`--lang ts`) | custom language 登録が必要 |
| 同梱配布 | npm/pnpm | flake devShell |
| toolchain pin | package.json + lock | flake.lock |
| LLM親和性 | 高 | 中（attrset構文だけなら高） |
| 副作用混入リスク | あり（関数を書ける） | ほぼゼロ（純粋関数言語） |

**Nix を選ぶべきユーザー像**：

- すでに `nix` や `flake` を業務で使っている
- toolchain (ast-grep / cocoindex / jq) のバージョン固定を最重要視
- 仕様ファイルに副作用を絶対書かせたくない
- CI ホストに Node を入れたくない（Nix だけにしたい）

**TS を選ぶべきユーザー像**：

- ast-grep の Nix custom language セットアップを避けたい
- LLM/レビュワーが TS のほうが圧倒的に慣れている
- monorepo の他パッケージと型定義を共有したい

---

## 18. 最小スケルトン（着手用）

ここまで読んで「やってみる」となった時の最短コース。

```bash
mkdir -p specs/{_schema,product,commands,workflows,quality}

# 1. types/ids/define を置く
$EDITOR specs/_schema/ids.nix
$EDITOR specs/_schema/types.nix
$EDITOR specs/_schema/define.nix

# 2. 最初の1要件だけ書く
$EDITOR specs/product/tumigi-product.nix

# 3. 集約
$EDITOR specs/default.nix

# 4. flake と sgconfig
$EDITOR specs/flake.nix
$EDITOR specs/sgconfig.yml

# 5. 検査
cd specs
nix flake check

# 6. 検索
nix develop
sg run --lang nix -p 'area = "product"' .
ccc index .
ccc search "/tumigi: プレフィックス"
```

---

## 19. まとめ

> **Tumigi の仕様を Nix expression にする利点は、
> 「toolchain と仕様が flake.lock で同じ粒度で固定される」ことに尽きる。**

- 正本：`specs/**/*.nix`（attrset、heredoc、`lib.types`で型検査）
- 構造検索：`ast-grep`（tree-sitter-nix を flake から供給）
- 意味検索：`cocoindex-code`（自然言語フィールドを索引）
- 配布：`flake devShell` で Agent と開発者に検索 CLI ごと配る
- 検証：`nix flake check` がそのままスキーマ検査になる

TS版が「アプリのコードと同じ言語で仕様も書く」発想なら、
Nix版は「**仕様と検索ツールチェインを同じ言語で書く**」発想。
Tumigi のようにコード本体より commands/skills/docs が主役のリポジトリでは、
後者のほうが噛み合いが良い。
