# superspecflow

OpenSpec + Superpowers 项目集成工具包。为任意项目结构从零集成 OpenSpec 规范驱动开发与 Superpowers AI Skill 体系，形成 propose → apply → archive 端到端开发工作流。

## 工作原理

superspecflow 将两个独立系统组合为一套完整的 AI 辅助开发工作流：

**OpenSpec**（规范驱动开发）：通过 `openspec/specs/` 下的三个规范文件（architecture.md、coding-conventions.md、business-domain.md）定义项目的架构约束、编码规范和业务域语义。所有功能变更走 `openspec new change` → `openspec archive` 流程，确保代码始终受规范约束。

**Superpowers**（AI Skill 体系）：提供 brainstorming、writing-plans、test-driven-development、systematic-debugging、verification-before-completion、code-review 等标准化子 Skill，作为工作流中的执行单元被自动编排调用。

两者集成后，形成三个项目级 Skill 组成的端到端流程：

```
propose（需求提案）→ apply（代码实施）→ archive（归档收尾）
```

每个 Skill 在执行时自动读取项目规范、调用 Superpowers 子 Skill、执行 OpenSpec CLI 命令，并在每个步骤完成后暂停等待用户确认，确保 AI 行为始终在人类监督和项目规范约束之下。

### 团队规范基准

superspecflow 维护了一套团队级规范基准文件（`shared/` 目录），作为所有项目规范的"宪法"。为项目生成规范文件时，执行"先读基准，再叠项目"的两步走策略：先读取对应基准文件获取章节骨架和团队通用条目，再叠加从项目代码分析中提取的特有约定。基准文件中的【强制】条目必须体现在项目规范中，项目特有约定不得与基准文件冲突。

## 包含的 Skill

### superspec-single

适用于 **single 项目**（单一代码库）和 **monorepo**（前后端代码共存于同一仓库）。

检测逻辑：根目录有 `.git`，且仅一种技术栈（single）或同时包含前端+后端代码（monorepo）。如果检测到 multi-repo 结构，会自动提示用户切换到 superspec-multi。

执行流程共 6 步：

```
Step 0  检测 AI 工具环境 → 确定 AI_TOOL 变量
Step 1  项目分析 → 输出变量表（技术栈、目录结构、构建命令、测试框架、Node.js 版本等）
Step 2  OpenSpec 初始化 + 规范处理（已有规范保留不覆盖，缺失的基于团队基准生成）
Step 3  创建 3 个自定义 Skill（propose / apply / archive）
Step 4  创建 AGENTS.md（+ 工具专属指令文件）
Step 5  创建 OPENSPEC_SUPERPOWERS_GUIDE.md 操作指南
Step 6  配置 .gitignore + 验证清单逐项检查
```

Step 2 中的规范文件处理采用"已有的不覆盖，缺失的才生成"策略：先检查 `openspec/specs/` 下是否已有 architecture.md、coding-conventions.md、business-domain.md，已存在的一律保留不修改，仅对缺失的文件基于团队基准 + 项目分析结果生成。这意味着重复执行 Skill 不会破坏已有的规范文件。

### superspec-multi

适用于 **multi-repo 项目**（多个独立 Git 仓库共存于一个父级目录，如 `cps-plat/` 下有 `cps-front-service/`、`cps-parent-domain/` 等各自独立的 git 仓库）。

检测逻辑：根目录无 `.git`，且多个子目录各自有独立 `.git`。如果检测到 single/monorepo 结构，会自动提示用户切换到 superspec-single。

核心设计是"父级统一管理，子项目独立规范"：父级目录负责 OpenSpec init、统一 Skill、AGENTS.md、操作指南和变更记录（`openspec/changes/`）；各子项目仅维护各自的 `openspec/specs/` 规范文件。所有 openspec CLI 命令都在父目录执行。

执行流程共 7 步：

```
Step 0  检测 AI 工具环境 → 确定 AI_TOOL 变量
Step 1  检测 multi-repo 结构 → 确认子项目列表
Step 2  逐子项目分析 → 输出各子项目变量表
Step 3  OpenSpec 初始化 + 规范处理（已有规范保留不覆盖，缺失的基于基准生成）
Step 4  创建统一 Skill（propose / apply / archive）→ 在父级目录
Step 5  创建统一 AGENTS.md（+ 工具专属指令文件）→ 在父级目录
Step 6  创建操作指南 → 在父级目录
Step 7  配置 .gitignore + 验证清单
```

Step 3 中的规范文件处理同样采用"已有的不覆盖，缺失的才生成"策略，且是逐子项目检查的：对每个子项目分别检查 `<子项目>/openspec/specs/` 下的三个规范文件，已存在的一律保留不修改，仅对缺失的文件基于团队基准 + 该子项目的分析结果生成。这在 multi-repo 场景中尤为重要——部分子项目可能已经跑过集成，重新执行时不会覆盖它们已有的规范。

### 两个 Skill 的自动互斥

两个 Skill 都内置了项目结构检测逻辑。执行任一 Skill 时，会先通过 `.git` 目录的分布情况自动判断项目结构类型。如果用户选错了 Skill，执行过程中会主动提示切换到正确的那个，无需用户自行判断。

## 使用方式

### 安装

将 `superspecflow/` 整个目录放入对应 AI 工具的 skills 目录（以 Qoder 和 CodeBuddy 为例）：

```bash
# Qoder
~/.qoder/skills/superspecflow/

# CodeBuddy
~/.codebuddy/skills-marketplace/skills/superspecflow/
```

安装后两个 Skill 自动可用，无需额外配置。

### 前置条件

执行 superspecflow Skill 之前，本地环境必须满足以下两个条件：

**1. nvm + Node.js：** 必须通过 [nvm](https://github.com/nvm-sh/nvm)（macOS/Linux）或 [nvm-windows](https://github.com/coreybutler/nvm-windows)（Windows）管理 Node.js 版本。Skill 执行 OpenSpec CLI 时会检测当前 Node.js 主版本是否 >= `config.json` 中 `nodeVersion.openspecMinMajor` 的值（默认 20），如果低于该版本会自动通过 `nvm use` 临时切换，执行完毕后再恢复。因此本地必须安装了 nvm，且通过 nvm 安装了满足要求的 Node.js 版本。如果本地安装的版本更高（如 22），直接在 `config.json` 中将 `openspecMinMajor` 改为对应值即可。

**2. Superpowers Skill：** 本地 AI 工具环境中必须已安装 superpowers Skill 包。superspecflow 在执行过程中会自动调用 superpowers 提供的子 Skill（brainstorming、writing-plans、test-driven-development、systematic-debugging、verification-before-completion、code-review 等），如果 superpowers 未安装，这些子 Skill 调用将失败。

> OpenSpec CLI 无需提前安装，Skill 执行过程中会自动执行 `npm install -g @fission-ai/openspec`。

### 触发执行

在 AI 编码工具中，以项目根目录作为工作区打开，然后执行对应命令：

| 场景 | 命令 |
|---|---|
| 单项目 / monorepo | `/superspec-single` |
| 多仓库 | `/superspec-multi` |

Skill 启动后会逐步执行各 Step，每步完成后展示结果并暂停等待确认。用户确认后继续下一步，直到所有步骤完成。

### 集成完成后的日常使用

集成完成后，项目中会生成三个项目级 Skill，对应开发流程的三个阶段：

| 阶段 | 命令（以 prefix=cps 为例） | 作用 |
|---|---|---|
| 需求提案 | `/cps-propose` | 读取项目规范 → brainstorming 设计 → 创建 OpenSpec 变更 → 生成实施计划 → 规范影响评估 |
| 代码实施 | `/cps-apply` | 加载变更上下文 → TDD 或编译驱动开发 → Bug 系统调试 → 逐任务完成 |
| 归档收尾 | `/cps-archive` | 编译+测试验证 → 代码审查 → 全局规范同步 → OpenSpec 归档 |

标准开发流程：提出需求 → `/cps-propose` → 确认设计和计划 → `/cps-apply` → 实施完成 → `/cps-archive` → 归档完成。

## 支持的 AI 工具

superspecflow 支持四种 AI 编码工具，执行时通过运行时信号（环境变量、进程、工作区目录）自动检测当前 Skill 运行在哪个工具环境中，并设置 `AI_TOOL` 变量，后续所有文件生成和路径选择都基于此变量适配。

检测优先级：CodeBuddy → Claude Code → Codex → Qoder（兜底）。各工具的检测信号：CodeBuddy 检测 `CODEBUDDY_SESSION_ID` 环境变量或 `codebuddy*` 进程或工作区 `.codebuddy/` 目录；Claude Code 检测 `CLAUDE_CODE_ENTRY`/`ANTHROPIC_API_KEY` 环境变量或 `claude` 进程；Codex 检测 `CODEX` 环境变量或 `codex` 进程；Qoder 作为兜底默认值。如需手动指定，可设置环境变量 `AI_TOOL` 覆盖自动检测结果。

各工具的适配差异：

| 配置项 | Qoder | Claude Code | Codex | CodeBuddy |
|---|---|---|---|---|
| Skill 存放路径 | `.qoder/skills/<PROJECT_PREFIX>-{phase}/SKILL.md` | `.claude/commands/<PROJECT_PREFIX>-{phase}.md` | `AGENTS.md` 内章节 | `.codebuddy/skills-marketplace/skills/<PROJECT_PREFIX>-{phase}/SKILL.md` |
| 文件格式 | YAML frontmatter + Markdown | Markdown + 可选 frontmatter | 纯 Markdown（嵌入 AGENTS.md） | YAML frontmatter + Markdown |
| 触发方式 | `/<PROJECT_PREFIX>-{phase}` | `/<PROJECT_PREFIX>-{phase}` | 自然语言引用 AGENTS.md 章节 | `/<PROJECT_PREFIX>-{phase}` 或 AI 自动选择 |
| 约束标签 | `<HARD-GATE>` | `> **MUST**` | `> **[强约束]**` | `<HARD-GATE>` |
| 子 Skill 调用 | `REQUIRED SUB-SKILL:` | 内联步骤描述 | 内联步骤描述 | `REQUIRED SUB-SKILL:` |
| OpenSpec init | `--tools qoder` | `--tools claude` | `--tools codex` | `--tools codebuddy` |
| init 产出目录 | `.qoder/` + `openspec/` | `.claude/` + `openspec/` | `AGENTS.md` + `openspec/` | `.codebuddy/` + `openspec/` |
| 项目级指令文件 | `AGENTS.md` | `AGENTS.md` + `CLAUDE.md` | `AGENTS.md` | `AGENTS.md` + `CODEBUDDY.md` |

## Git 管理

### superspecflow 自身

superspecflow 安装在各 AI 工具的全局 skills 目录下，属于个人全局配置，不纳入任何项目的 Git 管理。各工具的安装路径：

```bash
# Qoder
~/.qoder/skills/superspecflow/

# CodeBuddy
~/.codebuddy/skills-marketplace/skills/superspecflow/

# Claude Code（通过 .claude-plugin/ 识别）
~/.claude/skills/superspecflow/

# Codex（通过 AGENTS.md 识别）
~/.codex/skills/superspecflow/
```

如果需要团队共享，可以将 `superspecflow/` 目录纳入团队共享的 Skill 仓库，各成员 clone 后放入各自工具的 skills 目录即可。

### 集成产物的 Git 管理

Skill 执行完成后会在项目中生成一系列文件，其中一部分需要提交到 Git 供团队共享，另一部分是工具自动生成的、clone 后重新 init 即可恢复，不应提交。

#### superspec-single 集成产物

以 Qoder 环境为例（其他工具路径不同但原则一致）：

```
<PROJECT>/
├── .qoder/
│   ├── skills/
│   │   ├── openspec-*/              ← ❌ 不提交（自动生成，init 重新产生）
│   │   ├── <PROJECT_PREFIX>-propose/        ← ✅ 提交（自定义 Skill，团队共享）
│   │   ├── <PROJECT_PREFIX>-apply/          ← ✅ 提交
│   │   └── <PROJECT_PREFIX>-archive/        ← ✅ 提交
│   └── commands/                    ← ❌ 不提交（自动生成）
├── openspec/
│   ├── specs/                       ← ✅ 提交（项目规范，团队共享）
│   └── changes/                     ← ✅ 提交（变更历史）
├── AGENTS.md                        ← ✅ 提交（项目级 AI 规则）
└── OPENSPEC_SUPERPOWERS_GUIDE.md    ← ✅ 提交（团队操作指南）
```

各工具的提交策略汇总：

| 文件 | Qoder | Claude Code | Codex | CodeBuddy |
|---|---|---|---|---|
| 自定义 Skill（`<PROJECT_PREFIX>-*`） | ✅ `.qoder/skills/` | ✅ `.claude/commands/` | ✅ 在 AGENTS.md 内 | ✅ `.codebuddy/skills/` |
| init 生成的内置 Skill | ❌ `.qoder/skills/openspec-*/` | ❌ `.claude/commands/openspec-*.md` | — | ❌ `.codebuddy/skills-marketplace/skills/openspec-*/` |
| init 生成的 commands | ❌ `.qoder/commands/` | — | — | ❌ `.codebuddy/skills-marketplace/commands/` |
| openspec/specs/ | ✅ 提交 | ✅ 提交 | ✅ 提交 | ✅ 提交 |
| openspec/changes/ | ✅ 提交 | ✅ 提交 | ✅ 提交 | ✅ 提交 |
| AGENTS.md | ✅ 提交 | ✅ 提交 | ✅ 提交 | ✅ 提交 |
| CLAUDE.md | — | ✅ 提交 | — | — |
| CODEBUDDY.md | — | — | — | ✅ 提交 |
| OPENSPEC_SUPERPOWERS_GUIDE.md | ✅ 提交 | ✅ 提交 | ✅ 提交 | ✅ 提交 |

#### superspec-multi 集成产物

multi-repo 场景的 Git 管理更复杂，因为涉及父目录和各子项目两个层面：

```
<PARENT_DIR>/ (无 .git)
├── .qoder/                          ← 父级工具配置（不提交，或按团队约定）
├── openspec/changes/                ← 变更记录（父级管理，按团队约定）
├── AGENTS.md                        ← 统一 AI 规则
├── OPENSPEC_SUPERPOWERS_GUIDE.md    ← 操作指南
│
├── <子项目1>/ (.git — 独立仓库)
│   └── openspec/specs/              ← ✅ 提交到子项目的 git
├── <子项目2>/ (.git — 独立仓库)
│   └── openspec/specs/              ← ✅ 提交到子项目的 git
└── <子项目3>/ (.git — 独立仓库)
    └── openspec/specs/              ← ✅ 提交到子项目的 git
```

关键原则：各子项目的 `openspec/specs/` 规范文件提交到各自的 git 仓库中，确保规范跟随代码一起版本管理。父级目录本身没有 `.git`，AGENTS.md、Guide 等文件在父级目录中以本地文件形式存在，新成员 clone 各子项目后执行 Skill 即可自动重新生成。

#### .gitignore 配置

Skill 执行过程中会自动在项目 `.gitignore` 中添加对应工具的忽略规则。以下是各工具需要忽略的路径：

**Qoder：**
```gitignore
# OpenSpec auto-generated (regenerated via openspec init --tools qoder)
.qoder/commands/
.qoder/skills/openspec-*/
```

**Claude Code：**
```gitignore
# OpenSpec auto-generated (regenerated via openspec init --tools claude)
.claude/commands/openspec-*.md
```

**Codex：**
```gitignore
# OpenSpec auto-generated (regenerated via openspec init --tools codex)
# Codex 不生成独立配置文件，AGENTS.md 中的 openspec 章节为手动维护
```

**CodeBuddy：**
```gitignore
# OpenSpec auto-generated (regenerated via openspec init --tools codebuddy)
.codebuddy/skills-marketplace/commands/
.codebuddy/skills-marketplace/skills/openspec-*/
```

忽略规则的核心逻辑：`openspec init` 命令自动生成的内容（内置 Skill、斜杠命令）一律忽略，因为这些内容在 clone 后重新执行 `openspec init --tools <AI_TOOL>` 即可恢复。而自定义 Skill（`<PROJECT_PREFIX>-*`）、规范文件（`openspec/specs/`）、变更记录（`openspec/changes/`）和项目级规则文件（`AGENTS.md` 等）是团队需要共享的，必须提交。

### 新成员上手流程

新成员 clone 项目后，先确保本地已满足前置条件（nvm + Node.js、superpowers Skill，详见"前置条件"一节），然后只需三步：

1. 下载项目代码
2. 用 AI 编码工具打开项目目录
3. 根据项目类型执行对应 Skill（single/monorepo → `/superspec-single`，multi-repo → `/superspec-multi`）

Skill 会自动完成后续所有工作：环境检测、项目分析、OpenSpec 初始化、规范文件生成、自定义 Skill 创建、AGENTS.md 生成、.gitignore 配置等。每个步骤完成后会暂停展示结果，确认无误后继续下一步。

## 目录结构

```
superspecflow/
├── config.json                        # 全局配置（Node.js 版本、规范文件列表等）
├── README.md                          # 本文件
├── CLAUDE.md                          # Claude Code 入口
├── AGENTS.md                          # Codex 入口
├── .claude-plugin/                    # Claude Code 插件描述
│   ├── plugin.json
│   └── marketplace.json
├── shared/                            # 团队规范基准文件（两个 Skill 共享）
│   ├── codebook.md                    # 编码规范基准（→ coding-conventions.md）
│   ├── architecturebook.md            # 架构规范基准（→ architecture.md）
│   └── businessbook.md               # 业务域规范基准（→ business-domain.md）
└── skills/                            # 可执行 Skill 目录
    ├── superspec-single/             # single/monorepo 集成 Skill
    │   ├── SKILL.md                   # 主执行逻辑（6 步流程）
    │   ├── PLAYBOOK.md                # 各 Step 的完整模板、填充指引和决策点
    │   └── TOOL-ADAPTER.md            # 跨工具适配参考（格式示例、Shell 脚本双版本）
    └── superspec-multi/              # multi-repo 集成 Skill
        ├── SKILL.md                   # 主执行逻辑（7 步流程）
        ├── PLAYBOOK.md                # 各 Step 的完整模板、填充指引和决策点
        └── TOOL-ADAPTER.md            # 跨工具适配参考（含多项目场景适配）
```

每个 Skill 目录包含三个文件：SKILL.md 是精简执行版，定义步骤流程和决策点；PLAYBOOK.md 是详细参考，包含各步骤的完整模板和填充指引；TOOL-ADAPTER.md 是跨工具适配的速查手册。

## 配置

编辑 `config.json` 可调整以下配置：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `nodeVersion.openspecMinMajor` | 20 | OpenSpec CLI 要求的 Node.js 最低主版本号。如果本地安装的是更高版本（如 22），直接改为对应值即可 |
| `specs.files` | architecture.md, coding-conventions.md, business-domain.md | 需要生成的规范文件列表 |
| `specs.baselineDir` | shared | 团队基准文件目录（相对于 superspecflow 根目录） |
