---
name: superspec-multi
description: "为 multi-repo 项目集成 OpenSpec + Superpowers 开发工作流。在父级目录统一管理 Skill/AGENTS.md/Guide，各子项目独立维护 openspec 规范。已有规范不覆盖，仅生成缺失部分。"
---

# Multi-Repo OpenSpec + Superpowers 统一集成

为 multi-repo 项目（多个独立 Git 仓库共存于一个父级目录）提供 AI 层面的 monorepo 集成方案：
- **父级目录**：`openspec init` + 统一 Skill（propose/apply/archive）+ AGENTS.md + 操作指南 + `openspec/changes/`
- **各子项目**：仅 `openspec/specs/`（规范文件，可提交到各自 git），**无工具配置目录**
- **所有 openspec CLI 命令**均在父目录执行

**启动声明：** "我正在使用 superspec-multi Skill 执行多仓库统一集成。"

<HARD-GATE>
- 每个步骤完成后，暂停并向用户展示结果，等待确认后再进入下一步。
- 禁止跳过项目结构检测直接创建文件。
- 已有的 openspec/specs/*.md 文件严禁覆盖，必须保留原有内容。
- 禁止在未确认变量表的情况下创建 Skill / AGENTS.md / Guide。
</HARD-GATE>

## 详细参考

同目录下的 `PLAYBOOK.md` 包含各 Step 的完整模板、填充指引和决策点说明。
同目录下的 `TOOL-ADAPTER.md` 包含跨工具适配的格式示例、触发方式、子 Skill 内联模板和 Shell 脚本双版本。

**在创建 Skill 模板（Step 4）和 AGENTS.md（Step 5）时，优先读取 `PLAYBOOK.md` 中对应章节的详细模板，并根据 `AI_TOOL` 变量参考 `TOOL-ADAPTER.md` 中的格式要求，确保产出完整。**

---

## 团队规范基准文件

本 Skill 在 Step 3 为子项目生成规范文件时，必须以以下三个团队级基准文件作为基线：

| 基准文件 | 路径 | 生成目标 |
|---|---|---|
| 编码规范基准 | `codebook.md`（`../../shared/`） | → `coding-conventions.md` |
| 架构规范基准 | `architecturebook.md`（`../../shared/`） | → `architecture.md` |
| 业务域规范基准 | `businessbook.md`（`../../shared/`） | → `business-domain.md` |

**使用规则：**
- 生成新规范文件时，**先读取对应基准文件**获取章节结构和团队通用条目，**再叠加**从项目代码分析中提取的特有约定
- 已有规范文件不受影响（保留原有内容）
- 基准文件中的【强制】条目必须体现在子项目规范中
- 基准文件位于 `../../shared/` 目录（superspecflow 包级共享），路径可通过 `../../config.json` 的 `specs.baselineDir` 配置

---

## 流程总览

```
[Step 0] 检测 AI 工具环境 → 设置 AI_TOOL 变量
    ↓
[Step 1] 检测 multi-repo 结构 → 确认子项目列表
    ↓
[Step 2] 逐子项目分析 → 输出各子项目变量表
    ↓
[Step 3] OpenSpec 初始化 + 规范处理（有则保留，无则生成）
    ↓
[Step 4] 创建统一 Skill（propose / apply / archive）→ 按 AI_TOOL 选择路径和格式
    ↓
[Step 5] 创建统一 AGENTS.md（+ CLAUDE.md）→ 父级目录
    ↓
[Step 6] 创建操作指南 → 父级目录
    ↓
[Step 7] 配置 .gitignore + 验证清单
```

---

## Step 0: 检测 AI 工具环境

**在执行任何操作前，先检测当前运行在哪个 AI 工具环境中。**

> 详细的工具适配参考见同目录下的 `TOOL-ADAPTER.md`。

**检测原理：** 通过运行时信号（环境变量、进程、工作区目录）判断当前执行环境，而非检查系统中安装了哪些工具。优先级：CodeBuddy > Claude Code > Codex > Qoder（兜底）。如果自动检测结果不正确，可手动设置 `AI_TOOL` 变量覆盖。

**PowerShell 检测脚本：**

```powershell
# 检测当前 Skill 运行在哪个 AI 工具环境中（运行时信号，非安装检测）
if ($env:CODEBUDDY_SESSION_ID -or (Get-Process "codebuddy*" -ErrorAction SilentlyContinue) -or (Test-Path ".codebuddy")) {
    $env:AI_TOOL = "codebuddy"
} elseif ($env:CLAUDE_CODE_ENTRY -or $env:ANTHROPIC_API_KEY -or (Get-Process "claude" -ErrorAction SilentlyContinue)) {
    $env:AI_TOOL = "claude-code"
} elseif ($env:CODEX -or (Get-Process "codex" -ErrorAction SilentlyContinue)) {
    $env:AI_TOOL = "codex"
} else {
    # Qoder 兜底（~/.qoder/ 或 ~/.qoderwork/ 运行时始终存在）
    $env:AI_TOOL = "qoder"
}
Write-Output "AI_TOOL = $env:AI_TOOL"
Write-Output "（如检测不正确，可手动设置: $env:AI_TOOL = 'codebuddy' | 'claude-code' | 'codex' | 'qoder'）"
```

**bash 检测脚本（macOS/Linux）：**

```bash
# 检测当前 Skill 运行在哪个 AI 工具环境中（运行时信号，非安装检测）
if [ -n "$CODEBUDDY_SESSION_ID" ] || pgrep -x "codebuddy" >/dev/null 2>&1 || [ -d ".codebuddy" ]; then
    export AI_TOOL="codebuddy"
elif [ -n "$CLAUDE_CODE_ENTRY" ] || [ -n "$ANTHROPIC_API_KEY" ] || pgrep -x "claude" >/dev/null 2>&1; then
    export AI_TOOL="claude-code"
elif [ -n "$CODEX" ] || pgrep -x "codex" >/dev/null 2>&1; then
    export AI_TOOL="codex"
else
    # Qoder 兜底（~/.qoder/ 或 ~/.qoderwork/ 运行时始终存在）
    export AI_TOOL="qoder"
fi
echo "AI_TOOL = $AI_TOOL"
echo "（如检测不正确，可手动设置: export AI_TOOL=codebuddy|claude-code|codex|qoder）"
```

**检测信号说明：**

| 工具 | 环境变量 | 进程名 | 工作区目录 | 说明 |
|---|---|---|---|---|
| CodeBuddy | `CODEBUDDY_SESSION_ID` | `codebuddy*` | `.codebuddy/`（项目级） | 三者任一命中即判定 |
| Claude Code | `CLAUDE_CODE_ENTRY`、`ANTHROPIC_API_KEY` | `claude` | 无 | Claude Code CLI 运行时注入 |
| Codex | `CODEX` | `codex` | 无 | Codex 运行时注入 |
| Qoder | — | — | — | 兜底默认值 |

**输出变量：**
```
AI_TOOL = <codebuddy | claude-code | codex | qoder>
```

**展示检测结果给用户确认后，进入 Step 1。**

---

## Step 1: 检测 multi-repo 结构

**必须通过命令检测，禁止仅凭目录名称猜测：**

**PowerShell:**
```powershell
# Step A: 确认当前工作区根目录不是 git 仓库
Test-Path ".git"
# 预期结果: False（父目录本身无 .git）

# Step B: 检查子目录是否各自有独立 .git
Get-ChildItem -Directory | ForEach-Object {
    $hasGit = Test-Path "$($_.Name)/.git"
    $techHint = ""
    if (Test-Path "$($_.Name)/package.json") { $techHint = "[Node/前端]" }
    elseif (Test-Path "$($_.Name)/pom.xml") { $techHint = "[Java/Maven]" }
    elseif (Test-Path "$($_.Name)/build.gradle") { $techHint = "[Java/Gradle]" }
    elseif (Test-Path "$($_.Name)/go.mod") { $techHint = "[Go]" }
    elseif (Test-Path "$($_.Name)/requirements.txt") { $techHint = "[Python]" }
    Write-Output "$($_.Name) | git=$hasGit | $techHint"
}
```

**bash（macOS/Linux）：**
```bash
test -d .git && echo "HAS_GIT（非 multi-repo）" || echo "NO_GIT"
for d in */; do
    name="${d%/}"
    tech=""
    [ -f "$d/package.json" ] && tech="${tech}Node/前端, "
    [ -f "$d/pom.xml" ] && tech="${tech}Java/Maven, "
    [ -f "$d/build.gradle" ] && tech="${tech}Java/Gradle, "
    [ -f "$d/go.mod" ] && tech="${tech}Go, "
    [ -f "$d/requirements.txt" ] && tech="${tech}Python, "
    [ -d "$d/.git" ] && echo "$name | git=true | [${tech%, }]" || echo "$name | git=false | [${tech%, }]"
done
```

**判断规则：**
- 根目录无 `.git` + 多个子目录有独立 `.git` → 确认为 **multi-repo**，继续 Step 2
- 根目录有 `.git` → **不是 multi-repo**（这是 single 或 monorepo），停止并提示用户：

  > 当前项目不是 multi-repo 结构，请使用 `/superspec-single` 技能进行集成。

- 根目录无 `.git` + 少于 2 个子目录有 `.git` → **不是 multi-repo**，停止并提示用户：

  > 当前项目不是 multi-repo 结构（未发现多个独立 Git 仓库），请使用 `/superspec-single` 技能进行集成。

**输出：** 确认的子项目列表，例如：
```
✓ 检测到 multi-repo 结构，共 3 个子项目：
  1. cps-front-service [前端/Node]
  2. cps-mobile-service [前端/Node]
  3. cps-parent-domain [Java/Maven]
```

**展示结果给用户确认后，进入 Step 2。**

---

## Step 2: 逐子项目分析

对每个子项目进行独立分析。进入子项目目录后，判断其自身结构：
- 子项目根目录有 `.git` + 单一技术栈 → **single**
- 子项目根目录有 `.git` + 前后端代码共存 → **monorepo**（子项目本身是 monorepo）

### 2.1 分析维度（每个子项目）

1. **技术栈**：主框架版本、语言版本、构建工具、UI 组件库
2. **目录结构**：源代码根目录、模块组织方式
3. **构建与验证命令**：启动、构建、测试、Lint 命令
4. **编码规范**：命名规范、分层规范、路径别名、项目特有约定
5. **Node.js 版本**：项目构建版本（检测顺序：.nvmrc / .node-version → package.json engines.node → 依赖版本推断 → 询问用户）、NODE_BUILD_VERSION < <OPENSPEC_NODE_MAJOR> → NEED_NVM_SWITCH = true
6. **数据库**：数据库类型、ORM 框架、MCP 工具
7. **技术红线**：版本锁定、禁止引入的依赖

### 2.2 输出变量表

为每个子项目输出独立变量表：

```
# ===== 子项目: <SUB_PROJECT_NAME> =====
SUB_PROJECT_NAME    = <子项目目录名>
SUB_PROJECT_TYPE    = <single | monorepo>
TECH_TYPE           = <frontend | backend | mobile>
TECH_STACK          = <技术栈摘要>
BUILD_CMD           = <构建命令>
LINT_CMD            = <Lint 命令 / 无>
TEST_CMD            = <测试命令 / 无>
START_CMD           = <启动命令>
HAS_TEST_FRAMEWORK  = <true | false>
TEST_FRAMEWORK_NAME = <Jest / Vitest / JUnit / 无>
NEED_NVM_SWITCH     = <true（NODE_BUILD_VERSION < <OPENSPEC_NODE_MAJOR>）| false（NODE_BUILD_VERSION >= <OPENSPEC_NODE_MAJOR>）>
NODE_BUILD_VERSION  = <项目构建 Node.js 版本>
DB_TYPE             = <数据库类型 / 无>
ORM_FRAMEWORK       = <ORM 框架 / 无>
MCP_TOOLS           = <可用 MCP 工具 / 无>
TECH_CONSTRAINTS    = <技术红线列表>
HAS_EXISTING_SPECS  = <true | false（openspec/specs/ 下是否已有 .md 文件）>
```

另外输出父级目录统一变量：
```
# ===== 父级统一变量 =====
PARENT_DIR_NAME     = <父级目录名，如 cps-plat>
PROJECT_PREFIX      = <统一 Skill 命名前缀，如 cps>
SUB_PROJECT_COUNT   = <子项目数量>
SUB_PROJECT_LIST    = <子项目名称列表>
```

**前缀取值规则：**
- 格式：2-5 个小写字母，简短易记
- 来源：从父级目录名中提取缩写（如 `cps-plat` → `cps`，`my-platform` → `mp`）
- 唯一性：multi-repo 下所有子项目共用同一个前缀
- 示例：`cps`（CPS 平台）、`mp`（My Platform）

**展示所有变量表给用户确认后，进入 Step 3。**

---

## Step 3: OpenSpec 初始化 + 规范处理

> **`<OPENSPEC_NODE_MAJOR>`** 变量从 `../../config.json` 的 `nodeVersion.openspecMinMajor` 字段读取（默认 20）。

### 3.1 在父目录执行 openspec init

根据 Step 0 检测到的 AI_TOOL 变量选择对应参数：

```bash
# 在父级目录（工作区根目录）执行，不要 cd 到子项目
# Node.js 版本检查（openspec 需要 >= <OPENSPEC_NODE_MAJOR>）
# 判断依据：子项目的 NODE_BUILD_VERSION（非当前运行环境版本）
# 如果任一前端子项目 NODE_BUILD_VERSION < <OPENSPEC_NODE_MAJOR> → nvm use <OPENSPEC_NODE_MAJOR>

# 根据 AI_TOOL 选择 openspec init 参数
# AI_TOOL = qoder   → openspec init --tools qoder
# AI_TOOL = claude-code → openspec init --tools claude
# AI_TOOL = codex   → openspec init --tools codex
# AI_TOOL = codebuddy → openspec init --tools codebuddy
openspec init --tools <AI_TOOL>

# 如果之前切换了版本 → nvm use <NODE_BUILD_VERSION> 恢复
```

> 此步骤在父目录产生工具配置目录和 `openspec/`（changes 目录）。
> - **Qoder**: 生成 `.qoder/`（skills + commands）和 `openspec/`
> - **Claude Code**: 生成 `.claude/`（commands）和 `openspec/`
> - **Codex**: 生成/更新 `AGENTS.md` 和 `openspec/`
> - **CodeBuddy**: 生成 `.codebuddy/`（skills + agents）和 `openspec/`
>
> 子项目**不执行** init，不会有工具配置目录。

### 3.2 为各子项目创建 openspec/specs/ 目录

对每个子项目手动创建规范文件目录（非 init 产生）：

**PowerShell:**
```powershell
$subProjects = @(<SUB_PROJECT_LIST>)
foreach ($sp in $subProjects) {
    New-Item -ItemType Directory -Path "$sp/openspec/specs" -Force
}
```

**bash:**
```bash
for sp in <SUB_PROJECT_LIST>; do
    mkdir -p "$sp/openspec/specs"
done
```

### 3.3 规范文件处理（核心保护逻辑）

对每个子项目检查 `openspec/specs/` 目录：

**PowerShell:**
```powershell
$specsDir = "<SUB_PROJECT_NAME>/openspec/specs"
$existing = Get-ChildItem -Path $specsDir -Filter "*.md" -Recurse -ErrorAction SilentlyContinue
```

**bash:**
```bash
specs_dir="<SUB_PROJECT_NAME>/openspec/specs"
find "$specs_dir" -name "*.md" 2>/dev/null
```

**处理策略：**

| 文件 | 已存在 | 不存在 |
|---|---|---|
| architecture.md | ✅ 保留原有，不修改 | 📝 基于 `architecturebook.md` + Step 2 分析结果生成 |
| coding-conventions.md | ✅ 保留原有，不修改 | 📝 基于 `codebook.md` + Step 2 分析结果生成 |
| business-domain.md | ✅ 保留原有，不修改 | 📝 基于 `businessbook.md` + Step 2 分析结果生成 |

**生成规范时的要求（仅对不存在的文件）：**

生成流程为两步走：
1. **读取基准文件**：先读取对应的团队规范基准文件，获取章节结构和团队通用条目作为骨架
2. **叠加项目特有约定**：基于 Step 2 的代码分析结果，填充项目特有的技术栈、目录结构、命名约定、业务概念等内容

具体要求：
- **architecture.md**：以 `architecturebook.md` 的章节结构为骨架，填充项目技术栈、目录结构、模块关系、构建配置、路由架构（前端）、中间件（后端）
- **coding-conventions.md**：以 `codebook.md` 的章节结构为骨架，填充项目命名规范、分层规范、路径别名、导入规范、项目特有约定
- **business-domain.md**：以 `businessbook.md` 的章节结构为骨架，填充核心业务概念、主要功能模块、业务流程、接口映射

**展示每个子项目的规范处理结果（保留/新生成）给用户确认后，进入 Step 4。**

---

## Step 4: 创建统一 Skill（父级目录）

在父级目录下，根据 `AI_TOOL` 变量创建 3 个统一 Skill，覆盖所有子项目。

### 4.1 存放路径（按 AI_TOOL 条件分支）

<!-- AI_TOOL = qoder -->
```
<PARENT_DIR>/
└── .qoder/skills/
    ├── <PROJECT_PREFIX>-propose/SKILL.md
    ├── <PROJECT_PREFIX>-apply/SKILL.md
    └── <PROJECT_PREFIX>-archive/SKILL.md
```

<!-- AI_TOOL = claude-code -->
```
<PARENT_DIR>/
└── .claude/commands/
    ├── <PROJECT_PREFIX>-propose.md
    ├── <PROJECT_PREFIX>-apply.md
    └── <PROJECT_PREFIX>-archive.md
```

<!-- AI_TOOL = codex -->
```
三个工作流作为独立章节嵌入 <PARENT_DIR>/AGENTS.md 中：
## Propose 工作流 — ...
## Apply 工作流 — ...
## Archive 工作流 — ...
```

<!-- AI_TOOL = codebuddy -->
```
<PARENT_DIR>/
└── .codebuddy/skills/
    ├── <PROJECT_PREFIX>-propose/SKILL.md
    ├── <PROJECT_PREFIX>-apply/SKILL.md
    └── <PROJECT_PREFIX>-archive/SKILL.md
```

### 4.2 Propose Skill

<!-- AI_TOOL = qoder -->
创建 `.qoder/skills/<PROJECT_PREFIX>-propose/SKILL.md`（YAML frontmatter + Markdown 格式）

<!-- AI_TOOL = claude-code -->
创建 `.claude/commands/<PROJECT_PREFIX>-propose.md`（Markdown 格式，frontmatter 可选）

<!-- AI_TOOL = codex -->
在 `AGENTS.md` 中嵌入 `## Propose 工作流` 章节（纯 Markdown 格式）

<!-- AI_TOOL = codebuddy -->
创建 `.codebuddy/skills/<PROJECT_PREFIX>-propose/SKILL.md`（YAML frontmatter + Markdown 格式，同 Qoder）

**AI 提示词：**
```
基于 PLAYBOOK.md 中的 Propose Skill 模板，创建 Skill 文件。
决策点：
- 如果任一子项目 NEED_NVM_SWITCH = true，保留 "Node.js 版本管理" 节
- 如果有 MCP_TOOLS，技术红线含数据库操作约束
- 完成标志指向 /<PROJECT_PREFIX>-apply
```

**核心要素（必须包含）：**
- 启动时读取**所有子项目**的 `<子项目>/openspec/specs/` 规范（全局规范读取）
- 读取父目录 `AGENTS.md` 中定义的统一规则和技术红线
- 5 步流程：读规范 → brainstorming → OpenSpec 变更 → writing-plans → 规范影响评估
- OpenSpec CLI 命令在父目录执行
- 规范影响评估（specs-impact.md），标注受影响子项目的规范文件路径
- 完成标志输出格式
- 技术红线节
- 错误处理表

### 4.3 Apply Skill

<!-- AI_TOOL = qoder -->
创建 `.qoder/skills/<PROJECT_PREFIX>-apply/SKILL.md`（YAML frontmatter + Markdown 格式）

<!-- AI_TOOL = claude-code -->
创建 `.claude/commands/<PROJECT_PREFIX>-apply.md`（Markdown 格式，frontmatter 可选）

<!-- AI_TOOL = codex -->
在 `AGENTS.md` 中嵌入 `## Apply 工作流` 章节（纯 Markdown 格式）

<!-- AI_TOOL = codebuddy -->
创建 `.codebuddy/skills/<PROJECT_PREFIX>-apply/SKILL.md`（YAML frontmatter + Markdown 格式，同 Qoder）

**AI 提示词：**
```
基于 PLAYBOOK.md 中的 Apply Skill 模板，创建 Skill 文件。
决策点：
- 如果任一子项目 NEED_NVM_SWITCH = true，保留 Node.js 版本管理节
- TECH_TYPE = frontend/mobile 的子项目：TDD 检测 jest.config/vitest.config/package.json test
- TECH_TYPE = backend(Java) 的子项目：TDD 检测 pom.xml 中 spring-boot-starter-test / spring-boot-test / JUnit
- Step 3 编译验证命令使用各子项目 BUILD_CMD
- 完成报告指向 /<PROJECT_PREFIX>-archive
```

**核心要素（必须包含）：**
- 启动时读取所有子项目规范 + AGENTS.md
- Step 1: 加载上下文（4 个子步骤：读规范、加载变更、读上下文文件、展示进度）
- Step 2: 任务循环（5 个子步骤：任务评估、TDD条件化实现、Bug处理、标记完成、检查点）
  - 后端 Java 项目：Controller 层必须使用 `@SpringBootTest` 编写接口测试，验证入参、出参和请求类型
- Step 3: 完成验证与报告
- 暂停条件
- 技术红线节

### 4.4 Archive Skill

<!-- AI_TOOL = qoder -->
创建 `.qoder/skills/<PROJECT_PREFIX>-archive/SKILL.md`（YAML frontmatter + Markdown 格式）

<!-- AI_TOOL = claude-code -->
创建 `.claude/commands/<PROJECT_PREFIX>-archive.md`（Markdown 格式，frontmatter 可选）

<!-- AI_TOOL = codex -->
在 `AGENTS.md` 中嵌入 `## Archive 工作流` 章节（纯 Markdown 格式）

<!-- AI_TOOL = codebuddy -->
创建 `.codebuddy/skills/<PROJECT_PREFIX>-archive/SKILL.md`（YAML frontmatter + Markdown 格式，同 Qoder）

**AI 提示词：**
```
基于 PLAYBOOK.md 中的 Archive Skill 模板，创建 Skill 文件。
决策点：
- Step 1.1 编译验证：按子项目类型选BUILD_CMD+LINT_CMD
- Step 1.2 测试：HAS_TEST_FRAMEWORK=true → TEST_CMD
- Step 1.3 规范检查：前端写"页面模块结构、Model 命名、路由注册"；后端写"Controller 命名、分层结构、实体类注解"
- NEED_NVM_SWITCH 决定是否包含 Node.js 版本管理节
```

**核心要素（必须包含）：**
- HARD-GATE：禁止在未运行验证命令并确认输出的情况下声称工作已完成
- Step 1: 全面验证（4 个子步骤：编译+Lint、测试、规范合规、变更完整性）
- Step 2: 代码审查（3 个子步骤：准备上下文、执行审查、处理反馈）
- Step 3: 全局规范同步（3 个子步骤：读取影响、执行同步、同步输出）
- Step 4: 归档变更（使用 --skip-specs）+ 完成报告
- 暂停条件
- 技术红线节

**展示 3 个 Skill 内容给用户确认后，进入 Step 5。**

---

## Step 5: 创建统一项目级指令文件

根据 `AI_TOOL` 变量在父级目录创建对应的项目级指令文件。

### 5.0 工具适配说明

<!-- AI_TOOL = qoder -->
创建 `AGENTS.md`。Qoder 通过 `AGENTS.md` + `.qoder/skills/` 配合工作。

<!-- AI_TOOL = claude-code -->
创建两个文件：
1. `AGENTS.md` — 项目上下文和编码规则（内容同 Qoder 版）
2. `CLAUDE.md` — Claude Code 专属指令文件，包含 MUST 约束和命令引用

`CLAUDE.md` 结构：
```markdown
# CLAUDE.md — <PARENT_DIR_NAME> 项目指令

> 本文件为 Claude Code 专属指令，引用 AGENTS.md 中的完整规则。

## MUST（强制约束）

- 每步操作完成后暂停，等待用户确认
- 禁止跳过项目结构检测直接创建文件
- 已有的 openspec/specs/*.md 文件严禁覆盖
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：团队规范基准文件 > openspec/specs/ > 代码推断

## 参考文件

- 完整项目规则见 `AGENTS.md`
- 各子项目规范见 `<子项目>/openspec/specs/`
- 操作指南见 `OPENSPEC_SUPERPOWERS_GUIDE.md`
- 工具适配参考见 `TOOL-ADAPTER.md`

## 命令

- `/<PROJECT_PREFIX>-propose` — 需求提案全流程
- `/<PROJECT_PREFIX>-apply` — 代码实施全流程
- `/<PROJECT_PREFIX>-archive` — 归档收尾全流程
```

<!-- AI_TOOL = codex -->
创建 `AGENTS.md`。Codex 原生读取 `AGENTS.md`，三个工作流已在 Step 4 作为章节嵌入其中。
AGENTS.md 内容与 Qoder 版一致，区别：
- 无 `<HARD-GATE>` 标签，改用 `> **[强约束]**` 格式
- 无 `REQUIRED SUB-SKILL:` 语法，子 Skill 行为内联为逐步指令

<!-- AI_TOOL = codebuddy -->
创建 `AGENTS.md` + `CODEBUDDY.md`。CODEBUDDY.md 是 CodeBuddy 专属指令文件，引用 AGENTS.md 中的完整规则。
CodeBuddy 使用 YAML frontmatter + `<HARD-GATE>` + `REQUIRED SUB-SKILL:` 格式（同 Qoder），无需内联替换。

### 5.1 AGENTS.md 结构

```markdown
# AGENTS.md — <PARENT_DIR_NAME> 多项目统一规则

## 通用规则
- OpenSpec 工作流：propose → apply → archive
- 规范优先级：团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ 中的项目级规范 > 代码推断
- 禁止直接修改 openspec/specs/，必须通过 archive 流程更新
- 所有 openspec CLI 命令在父目录执行

## 全局规范读取
Skill 执行时必须读取所有子项目的规范文件，确保跨项目上下文完整：
- <子项目1>/openspec/specs/architecture.md
- <子项目1>/openspec/specs/coding-conventions.md
- <子项目1>/openspec/specs/business-domain.md
- <子项目2>/openspec/specs/...
- ...

## 子项目规则

### <子项目1>
- 技术栈约束
- 编码规范要点
- 技术红线

### <子项目2>
...

## 跨项目协作
- 跨项目变更需在 propose 阶段声明影响范围
- 前后端联调变更的 specs-impact.md 需标注所有受影响子项目的规范文件路径
```

**展示 AGENTS.md 内容给用户确认后，进入 Step 6。**

---

## Step 6: 创建操作指南

在父级目录创建 `OPENSPEC_SUPERPOWERS_GUIDE.md`。

### 6.1 指南内容要点

- 集成概述（简介 OpenSpec 规范驱动 + Superpowers AI Skill 体系 + 集成方式）
- 项目结构概览（multi-repo 布局）
- 各子项目技术栈速查
- 统一 Skill 使用方式（`/<PROJECT_PREFIX>-propose`、`/<PROJECT_PREFIX>-apply`、`/<PROJECT_PREFIX>-archive`）
- 各子项目 openspec 规范位置
- 新成员上手步骤（拉代码 → 用对应 AI 工具打开 → 执行 Skill 自动生成）
- FAQ（如何添加新子项目、如何处理跨项目变更）
- 当前 AI 工具环境说明（AI_TOOL 值及对应的命令触发方式）

**展示 Guide 内容给用户确认后，进入 Step 7。**

---

## Step 7: 配置与验证

### 7.1 验证清单

逐项检查（通过 ✅ 标记，未通过 ❌ 标记）：

```
[ ] Step 0: AI_TOOL 已检测并确认（codebuddy / claude-code / codex / qoder）
[ ] Step 1: multi-repo 结构确认，子项目列表正确
[ ] Step 2: 每个子项目变量表已确认
[ ] Step 3-A: 父目录 openspec init 执行成功（工具配置目录和 openspec/ 存在）
[ ] Step 3-B: 各子项目 openspec/specs/ 目录已创建
[ ] Step 3-C: 已有规范文件未被修改
[ ] Step 3-D: 缺失的规范文件已生成（基于团队基准文件 + 项目分析）
[ ] Step 3-E: 生成规范时已读取对应基准文件（codebook/architecturebook/businessbook）
[ ] Step 3-F: AGENTS.md 中规范优先级包含团队基准文件
[ ] Step 4: 3 个统一 Skill 已创建（路径匹配 AI_TOOL）
[ ] Step 5: 项目级指令文件已生成（匹配 AI_TOOL）
[ ] Step 6: 父级 OPENSPEC_SUPERPOWERS_GUIDE.md 存在
[ ] 功能验证: propose 命令可触发（方式匹配 AI_TOOL）
```

<!-- AI_TOOL = claude-code 时额外检查 -->
```
[ ] Claude Code: CLAUDE.md 已生成且引用 AGENTS.md
[ ] Claude Code: CLAUDE.md MUST 区包含强制约束
[ ] Claude Code: .claude/commands/ 下 3 个命令文件存在
```

<!-- AI_TOOL = codebuddy 时额外检查 -->
```
[ ] CodeBuddy: CODEBUDDY.md 已生成且引用 AGENTS.md
[ ] CodeBuddy: .codebuddy/skills/ 下 3 个 Skill 文件存在
```

---

## 最终目录结构

<!-- AI_TOOL = qoder -->
```
<PARENT_DIR>/ (无 .git，Qoder 工作区)
├── .qoder/
│   ├── skills/
│   │   ├── openspec-*/                ← init 生成的内置 Skill
│   │   ├── <PROJECT_PREFIX>-propose/SKILL.md  ← 统一 Propose Skill
│   │   ├── <PROJECT_PREFIX>-apply/SKILL.md    ← 统一 Apply Skill
│   │   └── <PROJECT_PREFIX>-archive/SKILL.md  ← 统一 Archive Skill
│   └── commands/                      ← init 生成
├── openspec/
│   └── changes/                       ← 变更记录（所有子项目共用）
├── AGENTS.md                          ← 统一 AI 规则
├── OPENSPEC_SUPERPOWERS_GUIDE.md      ← 操作指南
│
├── <子项目1>/ (.git)
│   └── openspec/
│       └── specs/                     ← 规范文件（可提交到 git）
├── <子项目2>/ (.git)
│   └── openspec/
│       └── specs/
└── <子项目3>/ (.git)
    └── openspec/
        └── specs/
```

<!-- AI_TOOL = claude-code -->
```
<PARENT_DIR>/ (无 .git，Claude Code 工作区)
├── .claude/
│   └── commands/
│       ├── <PROJECT_PREFIX>-propose.md        ← Propose 命令
│       ├── <PROJECT_PREFIX>-apply.md          ← Apply 命令
│       └── <PROJECT_PREFIX>-archive.md        ← Archive 命令
├── openspec/
│   └── changes/                       ← 变更记录
├── AGENTS.md                          ← 项目上下文和编码规则
├── CLAUDE.md                          ← Claude Code 专属指令
├── OPENSPEC_SUPERPOWERS_GUIDE.md      ← 操作指南
│
├── <子项目1>/ (.git)
│   └── openspec/specs/
├── <子项目2>/ (.git)
│   └── openspec/specs/
└── <子项目3>/ (.git)
    └── openspec/specs/
```

<!-- AI_TOOL = codex -->
```
<PARENT_DIR>/ (无 .git，Codex 工作区)
├── openspec/
│   └── changes/                       ← 变更记录
├── AGENTS.md                          ← 统一 AI 规则 + 3 个工作流章节
├── OPENSPEC_SUPERPOWERS_GUIDE.md      ← 操作指南
│
├── <子项目1>/ (.git)
│   └── openspec/specs/
├── <子项目2>/ (.git)
│   └── openspec/specs/
└── <子项目3>/ (.git)
    └── openspec/specs/
```

<!-- AI_TOOL = codebuddy -->
```
<PARENT_DIR>/ (无 .git，CodeBuddy 工作区)
├── .codebuddy/
│   ├── skills/
│   │   ├── <PROJECT_PREFIX>-propose/SKILL.md  ← 统一 Propose Skill
│   │   ├── <PROJECT_PREFIX>-apply/SKILL.md    ← 统一 Apply Skill
│   │   └── <PROJECT_PREFIX>-archive/SKILL.md  ← 统一 Archive Skill
│   └── agents/                        ← init 生成
├── openspec/
│   └── changes/                       ← 变更记录
├── AGENTS.md                          ← 统一 AI 规则
├── CODEBUDDY.md                       ← CodeBuddy 专属指令
├── OPENSPEC_SUPERPOWERS_GUIDE.md      ← 操作指南
│
├── <子项目1>/ (.git)
│   └── openspec/specs/
├── <子项目2>/ (.git)
│   └── openspec/specs/
└── <子项目3>/ (.git)
    └── openspec/specs/
```

> **注意：** 子项目没有工具配置目录（`.qoder/` / `.claude/` 等），也不需要 `.gitignore` 忽略它。

**完成所有验证后，输出集成完成报告。**
