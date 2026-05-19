もちろん。
`tumigi` 向けなら、前回の汎用 `Requirement / Plan / Task / Decision / Test` では少し抽象的すぎる。

`tumigi` は README 上では **Claude Code向けのAI駆動開発支援フレームワーク**で、Kairo / TDD / Dev Skills / DCS / ユーティリティ / リバースエンジニアリング系のコマンド群を持つ構成になっている。特に Kairo は「要件定義→設計→タスク分割→実装」の流れ、Dev Skills は `docs/dev/context.md` や `docs/dev/plans/` を中心にした開発フロー、DCS は `.dcs/` 配下に分析結果を出す構成なので、仕様モデルも **アプリ機能の要件**ではなく、**コマンド・Skill・出力成果物・ワークフローの要件**として定義したほうがよい。([GitHub][1])

以下、`tumigi` 用に修正した版。

---

# Tumigi向け TypeScript Object 要件定義案

## 1. ディレクトリ構成

```txt
specs/
  _schema/
    ids.ts
    tumigi-types.ts
    define.ts

  product/
    tumigi-product.ts

  commands/
    kairo.ts
    tdd.ts
    dev-skills.ts
    dcs.ts
    utility.ts
    reverse.ts

  workflows/
    kairo-workflow.ts
    dev-workflow.ts
    dcs-workflow.ts

  artifacts/
    tumigidoc.ts
    dev-docs.ts
    dcs-output.ts

  compatibility/
    claude-code-plugin.ts
    rulesync.ts

  quality/
    command-quality.ts
    documentation-quality.ts
```

`tumigi` はコード本体よりも `commands/*.md`、`skills/*`、`docs/*`、`.claude-plugin` などが中心のリポジトリなので、`implementedBy` は普通の `src/**/*.ts` よりも **コマンドファイル・Skillディレクトリ・出力ディレクトリ**を指す設計にするのがよい。リポジトリには `commands`、`skills`、`docs`、`.claude-plugin` などが存在し、コマンド一覧も `commands/*.md` として管理されている。([GitHub][1])

---

## 2. ID設計

```ts
// specs/_schema/ids.ts

export type TumigiProductRequirementId = `REQ-TUMIGI-${number}`;
export type CommandRequirementId = `REQ-CMD-${Uppercase<string>}-${number}`;
export type SkillRequirementId = `REQ-SKILL-${Uppercase<string>}-${number}`;
export type WorkflowRequirementId = `REQ-WF-${Uppercase<string>}-${number}`;
export type ArtifactRequirementId = `REQ-ARTIFACT-${Uppercase<string>}-${number}`;
export type CompatibilityRequirementId = `REQ-COMPAT-${Uppercase<string>}-${number}`;
export type QualityRequirementId = `REQ-QUALITY-${Uppercase<string>}-${number}`;

export type TumigiRequirementId =
  | TumigiProductRequirementId
  | CommandRequirementId
  | SkillRequirementId
  | WorkflowRequirementId
  | ArtifactRequirementId
  | CompatibilityRequirementId
  | QualityRequirementId;

export type TumigiPlanId = `PLAN-${Uppercase<string>}-${number}`;
export type TumigiTaskId = `TASK-${Uppercase<string>}-${number}`;
export type TumigiDecisionId = `DEC-${Uppercase<string>}-${number}`;
export type TumigiVerificationId = `VERIFY-${Uppercase<string>}-${number}`;
```

普通のアプリなら `REQ-AUTH-001` でよいが、`tumigi` は「コマンド仕様」が中心なので、`REQ-CMD-KAIRO-001` や `REQ-SKILL-DEVPLAN-001` のように分けたほうが検索しやすい。

---

## 3. Tumigi用の型

```ts
// specs/_schema/tumigi-types.ts

import type {
  TumigiRequirementId,
  TumigiPlanId,
  TumigiTaskId,
  TumigiDecisionId,
  TumigiVerificationId,
} from "./ids";

export type RequirementStatus =
  | "draft"
  | "accepted"
  | "implemented"
  | "verified"
  | "deprecated";

export type Priority = "must" | "should" | "could" | "wont";

export type TumigiArea =
  | "product"
  | "kairo"
  | "tdd"
  | "dev-skills"
  | "dcs"
  | "utility"
  | "reverse-engineering"
  | "claude-plugin"
  | "rulesync"
  | "documentation"
  | "quality";

export type TumigiArtifactRef = {
  path: string;
  kind:
    | "command"
    | "skill"
    | "doc"
    | "plugin-config"
    | "output-template"
    | "workflow"
    | "test"
    | "script";
};

export type TumigiCommandRef = {
  name: string;
  path: `commands/${string}.md` | `.claude/commands/${string}.yml`;
  prefix?: "/tumigi:";
};

export type TumigiSkillRef = {
  name: string;
  path: `skills/${string}`;
};

export type OutputContract = {
  pathPattern: string;
  description: string;
  requiredFiles?: string[];
};

export type TumigiRequirement = {
  kind: "requirement";
  id: TumigiRequirementId;
  area: TumigiArea;
  title: string;
  status: RequirementStatus;
  priority: Priority;

  description: string;

  rationale?: string;
  ears?: string[];

  dependsOn?: TumigiRequirementId[];
  relatedCommands?: TumigiCommandRef[];
  relatedSkills?: TumigiSkillRef[];
  outputContracts?: OutputContract[];

  implementedBy?: TumigiArtifactRef[];
  verifiedBy?: TumigiVerificationId[];

  tags?: string[];
};

export type TumigiWorkflow = {
  kind: "workflow";
  id: TumigiPlanId;
  area: TumigiArea;
  title: string;
  status: RequirementStatus;

  description: string;
  steps: {
    order: number;
    command: string;
    input?: string;
    output?: string;
    required?: boolean;
  }[];

  implements: TumigiRequirementId[];
  artifacts?: TumigiArtifactRef[];

  tags?: string[];
};

export type TumigiTask = {
  kind: "task";
  id: TumigiTaskId;
  title: string;
  status: RequirementStatus;
  area: TumigiArea;

  description: string;
  implements: TumigiRequirementId[];
  touches: TumigiArtifactRef[];

  verification?: TumigiVerificationId[];

  tags?: string[];
};

export type TumigiDecision = {
  kind: "decision";
  id: TumigiDecisionId;
  title: string;
  status: "proposed" | "accepted" | "superseded" | "rejected";

  context: string;
  decision: string;
  consequences: string;

  affects: TumigiRequirementId[];

  tags?: string[];
};

export type TumigiVerification = {
  kind: "verification";
  id: TumigiVerificationId;
  title: string;
  status: RequirementStatus;

  verifies: TumigiRequirementId[];
  method:
    | "manual"
    | "snapshot"
    | "secretlint"
    | "command-structure-check"
    | "docs-output-check"
    | "agent-simulation"
    | "rulesync-generation";

  command?: string;
  expectedArtifacts?: TumigiArtifactRef[];

  tags?: string[];
};

export type TumigiSpecNode =
  | TumigiRequirement
  | TumigiWorkflow
  | TumigiTask
  | TumigiDecision
  | TumigiVerification;
```

ここで重要なのは、`implementedBy` を `CodeSymbolRef` ではなく `TumigiArtifactRef` にしているところ。
`tumigi` の主対象は通常のアプリコードではなく、Claude Codeコマンド、Skill、Markdown仕様、出力ディレクトリ、plugin設定だから。

---

## 4. define関数

```ts
// specs/_schema/define.ts

import type {
  TumigiRequirement,
  TumigiWorkflow,
  TumigiTask,
  TumigiDecision,
  TumigiVerification,
} from "./tumigi-types";

export const defineRequirement = <T extends TumigiRequirement>(value: T) => value;
export const defineWorkflow = <T extends TumigiWorkflow>(value: T) => value;
export const defineTask = <T extends TumigiTask>(value: T) => value;
export const defineDecision = <T extends TumigiDecision>(value: T) => value;
export const defineVerification = <T extends TumigiVerification>(value: T) => value;
```

---

# 5. 製品全体の要件

```ts
// specs/product/tumigi-product.ts

import { defineRequirement } from "../_schema/define";

export const reqTumigiProduct001 = defineRequirement({
  kind: "requirement",
  id: "REQ-TUMIGI-001",
  area: "product",
  title: "TumigiはAI駆動開発の一連の工程をコマンドとして支援できる",
  status: "accepted",
  priority: "must",

  description: `
Tumigiは、要件定義、設計、タスク分割、実装、検証、デバッグ、リバースエンジニアリングを、
Claude Codeを中心としたAIコーディング環境で実行できるようにする。
  `,

  ears: [
    "When ユーザーが新規機能開発を開始する, the system shall KairoまたはDev Skillsの開始コマンドを提示できる。",
    "When ユーザーが既存コードを調査したい, the system shall DCSまたはリバースエンジニアリング系コマンドを提示できる。",
    "When ユーザーがClaude Code PluginとしてTumigiを導入する, the system shall /tumigi: プレフィックス付きコマンドを利用できる。",
  ],

  implementedBy: [
    { kind: "doc", path: "README.md" },
    { kind: "plugin-config", path: ".claude-plugin" },
    { kind: "command", path: "commands/help.md" },
  ],

  tags: ["tumigi", "product", "ai-driven-development"],
});
```

READMEでは Tumigi が「要件定義から実装まで、AIを活用した効率的な開発プロセスを提供する」と説明され、Claude Code Pluginとして `/tumigi:` プレフィックス付きコマンドを使う構成が示されているので、この要件はかなり中核になる。([GitHub][1])

---

# 6. Kairo要件

```ts
// specs/commands/kairo.ts

import { defineRequirement } from "../_schema/define";

export const reqCmdKairo001 = defineRequirement({
  kind: "requirement",
  id: "REQ-CMD-KAIRO-001",
  area: "kairo",
  title: "Kairoは要件定義から実装までの包括的な開発フローを提供する",
  status: "accepted",
  priority: "must",

  description: `
Kairoは、技術スタック特定、要件定義、設計文書生成、タスク分割、実装実行、自動連続実装を
一連の開発フローとして提供する。
  `,

  ears: [
    "When ユーザーがKairo開発を開始する, the system shall init-tech-stack, kairo-requirements, kairo-design, kairo-tasks, kairo-loop の順序を提示できる。",
    "When ユーザーが要件定義を実行する, the system shall EARS記法を用いて受け入れ基準を含む要件定義書を作成する。",
    "When ユーザーが実装を開始する, the system shall kairo-implement または kairo-loop によってTDD/DIRECTを用いた実装に進める。",
  ],

  relatedCommands: [
    { name: "init-tech-stack", path: "commands/init-tech-stack.md", prefix: "/tumigi:" },
    { name: "kairo-requirements", path: "commands/kairo-requirements.md", prefix: "/tumigi:" },
    { name: "kairo-design", path: "commands/kairo-design.md", prefix: "/tumigi:" },
    { name: "kairo-tasks", path: "commands/kairo-tasks.md", prefix: "/tumigi:" },
    { name: "kairo-loop", path: "commands/kairo-loop.md", prefix: "/tumigi:" },
  ],

  outputContracts: [
    {
      pathPattern: "docs/tumigidoc/{requirementName}/spec/",
      description: "Kairo要件定義・設計・タスク分割の成果物を格納する。",
      requiredFiles: ["requirements.md", "interview-record.md"],
    },
  ],

  implementedBy: [
    { kind: "command", path: "commands/kairo-requirements.md" },
    { kind: "command", path: "commands/kairo-design.md" },
    { kind: "command", path: "commands/kairo-tasks.md" },
    { kind: "command", path: "commands/kairo-loop.md" },
  ],

  tags: ["kairo", "requirements", "design", "tasks", "implementation"],
});
```

Kairoコマンド群は README で「包括的開発フロー」として説明され、`kairo-requirements` は EARS記法による要件定義、`kairo-design` は設計文書生成、`kairo-tasks` はタスク分割、`kairo-loop` は自動連続実装として列挙されている。([GitHub][1])
また `commands/kairo-requirements.md` では、出力ディレクトリが `docs/tumigidoc/{要件名}/spec` とされ、要件定義書やヒアリング記録を出力する流れになっている。([GitHub][2])

---

# 7. Dev Skills要件

```ts
// specs/commands/dev-skills.ts

import { defineRequirement } from "../_schema/define";

export const reqSkillDev001 = defineRequirement({
  kind: "requirement",
  id: "REQ-SKILL-DEV-001",
  area: "dev-skills",
  title: "Dev Skillsはコンテキスト分析から計画・実装・検証・デバッグまでを統合的に支援する",
  status: "accepted",
  priority: "must",

  description: `
Dev Skillsは、新規または既存プロジェクトのコンテキストを分析し、
実装計画、テストファースト実装、検証、デバッグ、画面仕様生成、Webテストまでを扱う。
  `,

  ears: [
    "When 既存プロジェクトでDev Skillsを使う, the system shall dev-context により docs/dev/context.md を生成する。",
    "When ユーザーが機能名と要件を指定する, the system shall dev-plan により docs/dev/plans/{planName}/ を生成する。",
    "When ユーザーがPlan内のタスクを実装する, the system shall dev-impl または dev-run によりテストファースト実装を行う。",
    "When 実装完了後, the system shall dev-verify によりテスト・ビルド・Lintを検証する。",
  ],

  relatedSkills: [
    { name: "dev-context", path: "skills/dev-context" },
    { name: "dev-plan", path: "skills/dev-plan" },
    { name: "dev-impl", path: "skills/dev-impl" },
    { name: "dev-run", path: "skills/dev-run" },
    { name: "dev-verify", path: "skills/dev-verify" },
    { name: "dev-debug", path: "skills/dev-debug" },
  ],

  outputContracts: [
    {
      pathPattern: "docs/dev/context.md",
      description: "プロジェクトコンテキスト。Dev Skillsの多くが前提とする。",
    },
    {
      pathPattern: "docs/dev/plans/{planName}/",
      description: "実装計画、タスク、検証レポートを格納する。",
      requiredFiles: ["plan.md"],
    },
  ],

  implementedBy: [
    { kind: "skill", path: "skills/dev-context" },
    { kind: "skill", path: "skills/dev-plan" },
    { kind: "skill", path: "skills/dev-impl" },
    { kind: "skill", path: "skills/dev-run" },
    { kind: "skill", path: "skills/dev-verify" },
    { kind: "doc", path: "DEV_README.md" },
  ],

  tags: ["dev-skills", "context", "plan", "implementation", "verification"],
});
```

Dev Skillsは `dev-init`、`dev-context`、`dev-plan`、`dev-impl`、`dev-run`、`dev-verify`、`dev-debug` などから構成され、`docs/dev/context.md` を共通基盤にして後続スキルが動く設計になっている。([GitHub][3])

---

# 8. DCS要件

```ts
// specs/commands/dcs.ts

import { defineRequirement } from "../_schema/define";

export const reqCmdDcs001 = defineRequirement({
  kind: "requirement",
  id: "REQ-CMD-DCS-001",
  area: "dcs",
  title: "DCSはコードベース分析・設計調査・影響範囲分析を構造化成果物として出力する",
  status: "accepted",
  priority: "must",

  description: `
DCSは、機能アイデア整理、シーケンス図作成、状態遷移分析、影響範囲分析、
増分開発計画、バグ分析、性能分析、コード質問、エッジケース分析を支援する。
  `,

  ears: [
    "When ユーザーが曖昧な機能アイデアを整理したい, the system shall feature-rubber-duck によりPRDを作成する。",
    "When ユーザーが変更前に影響範囲を把握したい, the system shall impact-analysis により層別の影響レポートを作成する。",
    "When ユーザーが状態遷移を理解したい, the system shall state-transition-analysis により状態遷移図・遷移表・リスク評価を作成する。",
    "When ユーザーが分析コマンドを実行する, the system shall .dcs/{timestamp}_{targetName}/ 配下に構造化された成果物を保存する。",
  ],

  relatedCommands: [
    { name: "dcs:feature-rubber-duck", path: "commands/dcs/feature-rubber-duck.md", prefix: "/tumigi:" },
    { name: "dcs:impact-analysis", path: "commands/dcs/impact-analysis.md", prefix: "/tumigi:" },
    { name: "dcs:incremental-dev", path: "commands/dcs/incremental-dev.md", prefix: "/tumigi:" },
    { name: "dcs:bug-analysis", path: "commands/dcs/bug-analysis.md", prefix: "/tumigi:" },
    { name: "dcs:code-question", path: "commands/dcs/code-question.md", prefix: "/tumigi:" },
  ],

  outputContracts: [
    {
      pathPattern: ".dcs/{timestamp}_{targetName}/",
      description: "DCS各コマンドの分析セッション成果物を格納する。",
      requiredFiles: ["index.md"],
    },
  ],

  implementedBy: [
    { kind: "doc", path: "DCS_README.md" },
    { kind: "command", path: "commands/dcs" },
  ],

  tags: ["dcs", "analysis", "impact-analysis", "prd", "reverse-analysis"],
});
```

DCSは `.dcs/` 配下にタイムスタンプ付きディレクトリを作り、`index.md` や分析種別ごとの成果物を出力する設計になっている。また、PRD作成、シーケンス図、状態遷移、影響範囲、増分開発、バグ分析、性能分析、コード質問、エッジケース分析などの用途が定義されている。([GitHub][4])

---

# 9. Claude Code Plugin / rulesync互換要件

```ts
// specs/compatibility/claude-code-plugin.ts

import { defineRequirement } from "../_schema/define";

export const reqCompatClaude001 = defineRequirement({
  kind: "requirement",
  id: "REQ-COMPAT-CLAUDE-001",
  area: "claude-plugin",
  title: "TumigiはClaude Code Pluginとして導入できる",
  status: "accepted",
  priority: "must",

  description: `
ユーザーはClaude Code Pluginの marketplace add / plugin install によりTumigiを導入できる。
導入後、/tumigi: プレフィックス付きスラッシュコマンドを利用できる。
  `,

  ears: [
    "When ユーザーがTumigiをClaude Code Pluginとしてインストールする, the system shall Tumigiのスラッシュコマンドとエージェントを利用可能にする。",
    "When ユーザーがコマンドを実行する, the system shall /tumigi: プレフィックスを要求する。",
  ],

  implementedBy: [
    { kind: "plugin-config", path: ".claude-plugin" },
    { kind: "doc", path: "README.md" },
  ],

  tags: ["claude-code", "plugin", "install"],
});
```

```ts
// specs/compatibility/rulesync.ts

import { defineRequirement } from "../_schema/define";

export const reqCompatRulesync001 = defineRequirement({
  kind: "requirement",
  id: "REQ-COMPAT-RULESYNC-001",
  area: "rulesync",
  title: "Tumigiはrulesync経由でClaude Code以外のAIコーディングツールにも展開できる",
  status: "accepted",
  priority: "should",

  description: `
Tumigiはrulesyncを組み合わせることで、Gemini CLI、Cursor、Codex CLI、Rooなど、
Claude Code以外のツール向けにもコマンドを出力できるようにする。
  `,

  ears: [
    "When ユーザーがClaude Code以外のツールでTumigiを使いたい, the system shall rulesync の import/generate フローを提示する。",
    "When ターゲットツールがカスタムスラッシュコマンドを直接サポートしない, the system shall experimental simulate commands の利用可能性を説明する。",
  ],

  implementedBy: [
    { kind: "doc", path: "README.md" },
  ],

  tags: ["rulesync", "cursor", "gemini-cli", "codex-cli", "compatibility"],
});
```

READMEでは、Claude Code Pluginの導入コマンドと `/tumigi:` プレフィックスの注意が示されており、さらに rulesync によって Gemini CLI、Cursor、Copilot、Codex CLI、Roo などへの生成例も説明されている。([GitHub][1])

---

# 10. Workflow定義

```ts
// specs/workflows/kairo-workflow.ts

import { defineWorkflow } from "../_schema/define";

export const workflowKairo001 = defineWorkflow({
  kind: "workflow",
  id: "PLAN-KAIRO-001",
  area: "kairo",
  title: "Kairo包括的開発フロー",
  status: "accepted",

  description: `
Kairoは、技術スタック初期化、要件定義、設計、タスク分割、実装を順に進める。
  `,

  implements: ["REQ-CMD-KAIRO-001"],

  steps: [
    {
      order: 1,
      command: "/tumigi:init-tech-stack",
      output: "技術スタック情報",
      required: true,
    },
    {
      order: 2,
      command: "/tumigi:kairo-requirements",
      output: "docs/tumigidoc/{requirementName}/spec/requirements.md",
      required: true,
    },
    {
      order: 3,
      command: "/tumigi:kairo-design",
      output: "設計文書",
      required: true,
    },
    {
      order: 4,
      command: "/tumigi:kairo-tasks",
      output: "タスク分割",
      required: true,
    },
    {
      order: 5,
      command: "/tumigi:kairo-loop",
      output: "実装結果",
      required: true,
    },
  ],

  artifacts: [
    { kind: "command", path: "commands/init-tech-stack.md" },
    { kind: "command", path: "commands/kairo-requirements.md" },
    { kind: "command", path: "commands/kairo-design.md" },
    { kind: "command", path: "commands/kairo-tasks.md" },
    { kind: "command", path: "commands/kairo-loop.md" },
  ],

  tags: ["kairo", "workflow"],
});
```

```ts
// specs/workflows/dev-workflow.ts

import { defineWorkflow } from "../_schema/define";

export const workflowDev001 = defineWorkflow({
  kind: "workflow",
  id: "PLAN-DEV-001",
  area: "dev-skills",
  title: "Dev Skills既存プロジェクト開発フロー",
  status: "accepted",

  description: `
既存プロジェクトでは、まずdev-contextでコンテキストを作成し、
dev-planで実装計画を作り、dev-implまたはdev-runで実装し、dev-verifyで検証する。
  `,

  implements: ["REQ-SKILL-DEV-001"],

  steps: [
    {
      order: 1,
      command: "/tumigi:dev-context",
      output: "docs/dev/context.md",
      required: true,
    },
    {
      order: 2,
      command: '/tumigi:dev-plan auth "ユーザー認証機能を実装"',
      output: "docs/dev/plans/{planName}/",
      required: true,
    },
    {
      order: 3,
      command: "/tumigi:dev-run auth 001 005",
      output: "実装結果",
      required: false,
    },
    {
      order: 4,
      command: "/tumigi:dev-verify auth",
      output: "docs/dev/plans/{planName}/reports/",
      required: true,
    },
  ],

  artifacts: [
    { kind: "skill", path: "skills/dev-context" },
    { kind: "skill", path: "skills/dev-plan" },
    { kind: "skill", path: "skills/dev-run" },
    { kind: "skill", path: "skills/dev-verify" },
  ],

  tags: ["dev-skills", "workflow"],
});
```

READMEのクイックスタートでも、Kairoは `init-tech-stack → kairo-requirements → kairo-design → kairo-tasks → kairo-loop`、Dev Skillsは `dev-context → dev-plan → dev-run → dev-verify` の流れで示されている。([GitHub][1])

---

# 11. Verification定義

```ts
// specs/quality/command-quality.ts

import { defineVerification } from "../_schema/define";

export const verifyCommandStructure001 = defineVerification({
  kind: "verification",
  id: "VERIFY-CMD-001",
  title: "全コマンドMarkdownに必要なfrontmatterが存在する",
  status: "draft",

  verifies: [
    "REQ-CMD-KAIRO-001",
    "REQ-CMD-DCS-001",
  ],

  method: "command-structure-check",

  command: "pnpm spec:check:commands",

  expectedArtifacts: [
    { kind: "command", path: "commands/kairo-requirements.md" },
    { kind: "command", path: "commands/kairo-design.md" },
    { kind: "command", path: "commands/kairo-tasks.md" },
    { kind: "command", path: "commands/dcs" },
  ],

  tags: ["command", "frontmatter", "quality"],
});
```

```ts
// specs/quality/documentation-quality.ts

import { defineVerification } from "../_schema/define";

export const verifyDocs001 = defineVerification({
  kind: "verification",
  id: "VERIFY-DOCS-001",
  title: "READMEと詳細ドキュメントのコマンド一覧が矛盾していない",
  status: "draft",

  verifies: [
    "REQ-TUMIGI-001",
    "REQ-CMD-KAIRO-001",
    "REQ-SKILL-DEV-001",
    "REQ-CMD-DCS-001",
  ],

  method: "docs-output-check",

  command: "pnpm spec:check:docs",

  expectedArtifacts: [
    { kind: "doc", path: "README.md" },
    { kind: "doc", path: "DEV_README.md" },
    { kind: "doc", path: "DCS_README.md" },
    { kind: "doc", path: "MANUAL.md" },
  ],

  tags: ["documentation", "consistency"],
});
```

`tumigi` の `package.json` には現状 `secretlint` と `biome` 周辺のスクリプト・devDependencies があり、`pnpm secretlint` を pre-commit に使う構成が見える。仕様検査を入れるなら、ここに `spec:check` 系を追加するのが自然。([GitHub][5])

---

# 12. Tumigi向けに追加すべき検査

前回の一般版では「コードsymbolが存在するか」を見ていたけど、`tumigi` では次を検査したほうがよい。

```txt
コマンド検査
- specs上の relatedCommands.path が実在する
- commands/*.md に description / allowed-tools / argument-hint がある
- READMEに載っているコマンドと commands/*.md がズレていない
- /tumigi: プレフィックスの説明がREADMEと各ドキュメントで矛盾しない

Skill検査
- relatedSkills.path が skills/* に存在する
- DEV_README.md のスキル一覧と skills/* がズレていない
- dev-* workflow の前提条件が docs/dev/context.md に統一されている

出力契約検査
- Kairo系は docs/tumigidoc/{要件名}/spec/ を出力契約に持つ
- Dev Skills系は docs/dev/context.md / docs/dev/plans/{planName}/ を出力契約に持つ
- DCS系は .dcs/{timestamp}_{targetName}/ を出力契約に持つ

ドキュメント整合性検査
- README / MANUAL / DEV_README / DCS_README のコマンド名が一致する
- examplesのコマンドが実在する
- deprecated扱いのコマンドがREADMEに残っていない

プラグイン互換検査
- .claude-plugin に含まれるコマンドと commands/*.md が一致する
- rulesyncで生成対象にするコマンドが仕様上マークされている
```

---

# 13. ast-grep検索例

`tumigi` では TSコードではなく、仕様TSと Markdown コマンドファイルを検索する運用になる。

```bash
# Kairo要件を探す
ast-grep run --lang ts -p 'area: "kairo"' specs

# 特定コマンドに紐づく要件を探す
ast-grep run --lang ts -p 'name: "kairo-requirements"' specs

# docs/tumigidoc 出力契約を持つ仕様を探す
ast-grep run --lang ts -p 'pathPattern: "docs/tumigidoc/{requirementName}/spec/"' specs

# Dev Skills関連要件を探す
ast-grep run --lang ts -p 'area: "dev-skills"' specs

# DCS関連要件を探す
ast-grep run --lang ts -p 'area: "dcs"' specs
```

---

# 14. TumigiのAGENTS.md / CLAUDE.md に書くなら

```md
# Tumigi Specification Policy

Tumigiでは、コマンド・Skill・ワークフロー・出力成果物の仕様を `specs/**/*.ts` に TypeScript object として定義する。

## 仕様確認手順

実装・修正前に必ず以下を確認する。

1. `specs/**/*.ts` から関連する requirement / workflow / task / verification を探す
2. 対象がコマンドの場合は `relatedCommands.path` を確認する
3. 対象がSkillの場合は `relatedSkills.path` を確認する
4. 対象が出力仕様の場合は `outputContracts` を確認する
5. README / DEV_README / DCS_README / MANUAL との整合性を確認する

## 重要な出力契約

- Kairo: `docs/tumigidoc/{要件名}/spec/`
- Dev Skills: `docs/dev/context.md`, `docs/dev/plans/{planName}/`
- DCS: `.dcs/{timestamp}_{targetName}/`

## 禁止事項

- コマンド名を変更したのにREADMEを更新しない
- 出力ディレクトリを変更したのに仕様を更新しない
- `commands/*.md` と `skills/*` の対応を曖昧にする
- Claude Code Plugin用の仕様とrulesync用の説明を矛盾させる
```

---

# まとめ

`tumigi` 向けにするなら、前回の汎用モデルをこう変えるのがよい。

```txt
一般アプリ向け:
Requirement -> Plan -> Task -> CodeSymbol -> Test

Tumigi向け:
Requirement -> Workflow -> Command / Skill -> OutputContract -> Verification
```

つまり、`tumigi` の正本仕様では、`implementedBy` をコードsymbolではなく、

```txt
commands/*.md
skills/*
README.md
DEV_README.md
DCS_README.md
.claude-plugin
docs/tumigidoc/{要件名}/spec/
docs/dev/**
.dcs/**
```

に結びつける。

一言でいうと、

> **Tumigiの仕様は「アプリ機能の仕様」ではなく、「AI開発コマンド群の仕様」としてTypeScript object化するのがよい。**

この形なら、Coding Agent が「このコマンドは何を保証しているのか」「どの出力成果物を作るべきか」「READMEと実装コマンドがズレていないか」を追いやすくなる。

[1]: https://github.com/seiei-sogen/tumigi "GitHub - seiei-sogen/tumigi: classmethod/tsumiki 改造版 · GitHub"
[2]: https://raw.githubusercontent.com/seiei-sogen/tumigi/main/commands/kairo-requirements.md "raw.githubusercontent.com"
[3]: https://raw.githubusercontent.com/seiei-sogen/tumigi/main/DEV_README.md "raw.githubusercontent.com"
[4]: https://raw.githubusercontent.com/seiei-sogen/tumigi/main/DCS_README.md "raw.githubusercontent.com"
[5]: https://raw.githubusercontent.com/seiei-sogen/tumigi/main/package.json "raw.githubusercontent.com"
