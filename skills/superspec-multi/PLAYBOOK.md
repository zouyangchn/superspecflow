# Multi-Repo OpenSpec + Superpowers 统一集成操作手册

## 概述

本手册指导如何为 **multi-repo 项目**（多个独立 Git 仓库共存于一个父级目录）集成 OpenSpec + Superpowers 开发工作流。

### 核心理念

> 在 AI 工具层面将 multi-repo 当作 monorepo 管理：父级目录统一协调，各子项目独立演进。

### 与 single/monorepo 集成的区别

| 维度 | single/monorepo | multi-repo（本手册） |
|---|---|---|
| openspec init 位置 | 项目内 | **父级目录** |
| 工具配置目录位置 | 项目内 | **父级目录**（子项目无配置目录） |
| Skill 位置 | 项目内工具配置目录 | **父级目录**工具配置目录 |
| AGENTS.md | 项目根目录 | **父级目录** |
| openspec/changes/ | 项目内 | **父级目录** |
| openspec/specs/ | 项目内 | 各**子项目**内 |
| CLI 执行位置 | 项目内 | **父级目录** |
| git 管理 | 全部可提交 | Skill/AGENTS.md **不可** git 提交，子项目 specs 可提交 |
| 适用场景 | 单仓库 | 多仓库协作 |

### 前置条件

- Node.js >= <OPENSPEC_NODE_MAJOR>（OpenSpec CLI 需要）
- 已全局安装 `@fission-ai/openspec`：`npm install -g @fission-ai/openspec`
- AI 工具以**父级目录**作为工作区打开

---

## 工具配置映射

> 本节定义四个支持工具的配置差异，作为各 Step 条件分支的参考源。
> 详细格式示例和子 Skill 内联模板见同目录下的 `TOOL-ADAPTER.md`。

| 配置项 | Qoder | Claude Code | Codex | CodeBuddy |
|---|---|---|---|---|
| Skill 存放路径 | `.qoder/skills/<PROJECT_PREFIX>-{phase}/SKILL.md` | `.claude/commands/<PROJECT_PREFIX>-{phase}.md` | `AGENTS.md` 内章节 | `.codebuddy/skills/<PROJECT_PREFIX>-{phase}/SKILL.md` |
| 文件格式 | YAML frontmatter + Markdown | Markdown + 可选 frontmatter | 纯 Markdown（嵌入 AGENTS.md） | YAML frontmatter + Markdown（同 Qoder） |
| 触发方式 | `/<PROJECT_PREFIX>-{phase}` | `/<PROJECT_PREFIX>-{phase}` | 自然语言指令引用 AGENTS.md 章节 | `/<PROJECT_PREFIX>-{phase}` 或 AI 自动选择 |
| 约束标签 | `<HARD-GATE>...</HARD-GATE>` | CLAUDE.md 中的 `MUST` 指令区 | AGENTS.md 中的强约束章节 | `<HARD-GATE>...</HARD-GATE>`（同 Qoder） |
| 子 Skill 调用 | `REQUIRED SUB-SKILL:` 语法 | 内联步骤描述 | 内联步骤描述 | `REQUIRED SUB-SKILL:` 语法（同 Qoder） |
| OpenSpec init | `openspec init --tools qoder` | `openspec init --tools claude` | `openspec init --tools codex` | `openspec init --tools codebuddy` |
| 项目级指令文件 | `AGENTS.md` | `AGENTS.md` + `CLAUDE.md` | `AGENTS.md` | `AGENTS.md` + `CODEBUDDY.md` |
| init 产出目录 | `.qoder/` + `openspec/` | `.claude/` + `openspec/` | `AGENTS.md` + `openspec/` | `.codebuddy/` + `openspec/` |
| Superpowers 支持 | 完整（原生子代理编排） | 完整（原生子代理编排） | 完整（框架层支持） | 完整（原生子代理编排） |

**AI_TOOL 变量：** 由 SKILL.md Step 0 检测，值为 `codebuddy | claude-code | codex | qoder`。后续各 Step 根据此变量选择对应的路径、格式和语法。

---

## Step 1: 检测 multi-repo 结构

### 1.1 执行检测

**PowerShell:**

```powershell
# 确认父目录本身不是 git 仓库
Test-Path ".git"
# 预期: False

# 列出所有子目录及其 git 和技术栈信息
Get-ChildItem -Directory | ForEach-Object {
    $name = $_.Name
    $hasGit = Test-Path "$name/.git"
    $tech = @()
    if (Test-Path "$name/package.json") { $tech += "Node/前端" }
    if (Test-Path "$name/pom.xml") { $tech += "Java/Maven" }
    if (Test-Path "$name/build.gradle") { $tech += "Java/Gradle" }
    if (Test-Path "$name/go.mod") { $tech += "Go" }
    if (Test-Path "$name/requirements.txt") { $tech += "Python" }
    $techStr = if ($tech.Count -gt 0) { $tech -join ", " } else { "未识别" }
    Write-Output "$name | git=$hasGit | [$techStr]"
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

### 1.2 判断规则

| 检测结果 | 判定 | 下一步 |
|---|---|---|
| 根目录无 `.git` + 多个子目录有 `.git` | ✅ multi-repo | 继续 Step 2 |
| 根目录有 `.git` | ❌ 不是 multi-repo（single 或 monorepo） | 停止并提示：`请使用 /superspec-single 技能进行集成` |
| 根目录无 `.git` + 少于 2 个子目录有 `.git` | ❌ 不是 multi-repo（未发现多个独立 Git 仓库） | 停止并提示：`请使用 /superspec-single 技能进行集成` |

### 1.3 输出格式

```
✓ 检测到 multi-repo 结构
  父级目录: cps-plat
  子项目列表:
    1. cps-front-service [Node/前端] ← 独立 git 仓库
    2. cps-mobile-service [Node/前端] ← 独立 git 仓库
    3. cps-parent-domain [Java/Maven] ← 独立 git 仓库
```

---

## Step 2: 逐子项目分析

### 2.1 AI 提示词

对每个子项目，进入其目录执行分析：

```
请对子项目 <SUB_PROJECT_NAME> 进行全面分析：

1. 技术栈：主框架和版本、语言版本、构建工具、UI 组件库、HTTP 通信方案
2. 目录结构：源代码根目录、模块组织方式、路由配置位置
3. 构建与验证命令：启动、构建、测试、Lint 命令
4. 编码规范：命名规范、分层规范、路径别名、项目特有约定
5. Node.js 版本：项目构建版本（检测顺序：.nvmrc / .node-version → package.json engines.node → 依赖版本推断 → 询问用户）、NODE_BUILD_VERSION < <OPENSPEC_NODE_MAJOR> → NEED_NVM_SWITCH = true
6. 数据库：类型、ORM 框架、可用 MCP 工具
7. 技术红线：版本锁定、禁止引入的依赖

另外检查：openspec/specs/ 目录下是否已有规范文件？列出已有的文件名。
```

### 2.2 变量表模板

每个子项目输出：

```
# ===== 子项目: <SUB_PROJECT_NAME> =====
SUB_PROJECT_NAME    = <目录名>
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
PAGE_STRUCTURE      = <页面标准结构 / 不适用>
ROUTE_CONFIG_PATH   = <路由配置路径 / 不适用>
PATH_ALIAS          = <路径别名 / 无>
DB_TYPE             = <数据库类型 / 无>
ORM_FRAMEWORK       = <ORM 框架 / 无>
MCP_TOOLS           = <可用 MCP 工具 / 无>
TECH_CONSTRAINTS    = <技术红线列表>
HAS_EXISTING_SPECS  = <true | false>
EXISTING_SPEC_FILES = <已有的规范文件名列表 / 无>
```

父级统一变量：

```
# ===== 父级统一变量 =====
PARENT_DIR_NAME     = <父级目录名>
PROJECT_PREFIX      = <统一 Skill 前缀>
SUB_PROJECT_COUNT   = <子项目数量>
SUB_PROJECT_LIST    = <子项目名称列表，逗号分隔>
```

**前缀取值规则：**
- 格式：2-5 个小写字母，简短易记
- 来源：从父级目录名中提取缩写（如 `cps-plat` → `cps`，`my-platform` → `mp`）
- 唯一性：multi-repo 下所有子项目共用同一个前缀
- 示例：`cps`（CPS 平台）、`mp`（My Platform）

---

## Step 3: OpenSpec 初始化 + 规范处理

### 3.1 在父目录执行 openspec init

根据 `AI_TOOL` 变量选择对应参数：

```bash
# 在父级目录（工作区根目录）执行，不要 cd 到子项目
# Node.js 版本判断依据：子项目的 NODE_BUILD_VERSION（非当前运行环境版本）
# 如果任一前端子项目 NODE_BUILD_VERSION < 20 → nvm use <OPENSPEC_NODE_MAJOR>

# 根据 AI_TOOL 选择 openspec init 参数
# AI_TOOL = qoder   → openspec init --tools qoder
# AI_TOOL = claude-code → openspec init --tools claude
# AI_TOOL = codex   → openspec init --tools codex
# AI_TOOL = codebuddy → openspec init --tools codebuddy
openspec init --tools <AI_TOOL>

# 恢复版本（如果切换过）
# nvm use <NODE_BUILD_VERSION>
```

> 此步骤在父目录产生工具配置目录和 `openspec/`（changes 目录）：
> - **Qoder**: `.qoder/`（skills + commands）+ `openspec/`
> - **Claude Code**: `.claude/`（commands）+ `openspec/`
> - **Codex**: `AGENTS.md` + `openspec/`
> - **CodeBuddy**: `.codebuddy/`（skills）+ `openspec/`
>
> **子项目不执行 init，不会有工具配置目录。**

### 3.2 为各子项目创建 openspec/specs/ 目录

**PowerShell:**
```powershell
$subProjects = @("<SUB_PROJECT_1>", "<SUB_PROJECT_2>", "<SUB_PROJECT_3>")
foreach ($sp in $subProjects) {
    New-Item -ItemType Directory -Path "$sp/openspec/specs" -Force
}
```

**bash:**
```bash
for sp in <SUB_PROJECT_1> <SUB_PROJECT_2> <SUB_PROJECT_3>; do
    mkdir -p "$sp/openspec/specs"
done
```

### 3.3 规范文件处理（核心保护逻辑）

**对每个子项目执行以下判断：**

**PowerShell:**
```powershell
$specsPath = "<SUB_PROJECT_NAME>/openspec/specs"

# 检查三个核心规范文件
$files = @("architecture.md", "coding-conventions.md", "business-domain.md")
foreach ($file in $files) {
    $directPath = "$specsPath/$file"
    $capabilityPaths = Get-ChildItem -Path $specsPath -Directory -ErrorAction SilentlyContinue |
        ForEach-Object { "$($_.FullName)/$file" }

    $found = (Test-Path $directPath) -or ($capabilityPaths | Where-Object { Test-Path $_ })

    if ($found) {
        Write-Output "$file → ✅ 已存在，保留原有"
    } else {
        Write-Output "$file → 📝 不存在，需要生成"
    }
}
```

**bash:**
```bash
specs_path="<SUB_PROJECT_NAME>/openspec/specs"
for file in architecture.md coding-conventions.md business-domain.md; do
    if [ -f "$specs_path/$file" ] || find "$specs_path" -mindepth 2 -name "$file" -print -quit 2>/dev/null | grep -q .; then
        echo "$file → ✅ 已存在，保留原有"
    else
        echo "$file → 📝 不存在，需要生成"
    fi
done
```

### 3.3 规范生成指引（仅对不存在的文件）

> **核心原则：先读基准，再叠项目。** 生成每个规范文件时，必须先读取对应的团队规范基准文件获取章节结构和通用条目，再叠加从 Step 2 代码分析中提取的项目特有约定。

**生成流程（两步走）：**

```
Step A: 读取基准文件 → 获取章节骨架 + 团队通用条目
    ↓
Step B: 叠加项目分析结果 → 填充项目特有的技术栈/目录/命名/业务等内容
```

**各文件对应的基准文件：**

| 生成目标 | 基准文件 | 路径 |
|---|---|---|
| `architecture.md` | 架构规范基准 | `architecturebook.md`（`../../shared/`） |
| `coding-conventions.md` | 编码规范基准 | `codebook.md`（`../../shared/`） |
| `business-domain.md` | 业务域规范基准 | `businessbook.md`（`../../shared/`） |

**生成时的具体要求：**

- **architecture.md**：以 `architecturebook.md` 的章节结构为骨架，填充项目技术栈、目录结构、模块关系、构建配置、路由架构（前端）、中间件（后端）、HTTP 通信方案、环境配置
- **coding-conventions.md**：以 `codebook.md` 的章节结构为骨架，填充项目命名规范、页面/模块标准文件结构、分层规范、样式规范（前端）、路径别名、导入导出规范、注释规范、项目特有约定
- **business-domain.md**：以 `businessbook.md` 的章节结构为骨架，填充核心业务概念和术语、主要功能模块、业务流程说明、数据模型概要

**重要约束：**
- 基准文件中的【强制】条目必须体现在子项目规范中
- 项目特有约定不应与基准文件中的【强制】条目冲突
- 如有冲突，以基准文件为准

### 3.4 输出确认

对每个子项目输出处理结果：

```
父目录 openspec init: ✅ 执行成功（工具配置目录 + openspec/ 已创建）

子项目: cps-front-service
  openspec/specs/ 目录: ✅ 已创建
  architecture.md: ✅ 已存在，保留
  coding-conventions.md: ✅ 已存在，保留
  business-domain.md: 📝 新生成

子项目: cps-parent-domain
  openspec/specs/ 目录: ✅ 已创建
  architecture.md: 📝 新生成
  coding-conventions.md: 📝 新生成
  business-domain.md: 📝 新生成
```

---

## Step 4: 创建统一 Skill

根据 `AI_TOOL` 变量，在父级目录创建 3 个 Skill。路径和格式见下方条件分支。

**格式适配要点：**
- **Qoder**: YAML frontmatter（`name` + `description`）+ `<HARD-GATE>` 标签 + `REQUIRED SUB-SKILL:` 语法
- **Claude Code**: 可选 frontmatter + `> **MUST**` 约束格式 + 内联步骤描述替代子 Skill 调用
- **Codex**: 纯 Markdown 嵌入 AGENTS.md + `> **[强约束]**` 约束格式 + 内联步骤描述
- **CodeBuddy**: 与 Qoder 格式完全相同（YAML frontmatter + `<HARD-GATE>` + `REQUIRED SUB-SKILL:`），无需额外适配

> 子 Skill 内联替代模板见 `TOOL-ADAPTER.md` 第 4 节。

### 4.1 Propose Skill 模板

<!-- AI_TOOL = qoder -->
路径：`.qoder/skills/<PROJECT_PREFIX>-propose/SKILL.md`

<!-- AI_TOOL = claude-code -->
路径：`.claude/commands/<PROJECT_PREFIX>-propose.md`

<!-- AI_TOOL = codex -->
路径：嵌入 `AGENTS.md` 的 `## Propose 工作流` 章节

<!-- AI_TOOL = codebuddy -->
路径：`.codebuddy/skills/<PROJECT_PREFIX>-propose/SKILL.md`

```markdown
---
name: <PROJECT_PREFIX>-propose
description: "多项目需求提案全流程：自动串联 brainstorming → OpenSpec → writing-plans。覆盖子项目：<SUB_PROJECT_LIST>。"
---

# Propose — 多项目需求提案全流程

**启动声明：** "我正在使用 <PROJECT_PREFIX>-propose Skill 执行需求提案全流程。"

<HARD-GATE>
在用户明确批准设计之前，禁止执行任何代码编写或实现操作。
</HARD-GATE>

## 流程总览

用户需求 → [Step 1] 读取项目规范 → [Step 2] brainstorming → [Step 3] OpenSpec 变更 → [Step 4] writing-plans → [Step 5] 规范影响评估

<!-- 如果任一子项目 NEED_NVM_SWITCH = true，包含此节 -->
## Node.js 版本管理

OpenSpec CLI 依赖 Node.js >= <OPENSPEC_NODE_MAJOR>。判断依据是子项目的 NODE_BUILD_VERSION，而非当前运行环境版本。检测顺序：.nvmrc / .node-version → package.json engines.node → 依赖版本推断 → 询问用户。
1. 读取子项目 NODE_BUILD_VERSION
2. NODE_BUILD_VERSION >= <OPENSPEC_NODE_MAJOR> → 直接执行 openspec 命令
3. NODE_BUILD_VERSION < <OPENSPEC_NODE_MAJOR> → `nvm use <OPENSPEC_NODE_MAJOR>` 切换后执行 openspec 命令
4. openspec 命令完毕后 `nvm use <NODE_BUILD_VERSION>` 恢复项目构建版本

## Step 1: 读取项目规范（自动执行，全局规范读取）

读取所有子项目的规范（确保跨项目上下文完整）：
<!-- 按实际子项目列表展开 -->
- `<子项目1>/openspec/specs/architecture.md`
- `<子项目1>/openspec/specs/coding-conventions.md`
- `<子项目1>/openspec/specs/business-domain.md`
- `<子项目2>/openspec/specs/...`
- ...

同时读取父目录 `AGENTS.md` 中定义的统一规则和技术红线。

检查父目录 `openspec/changes/` 下活跃变更。

**询问用户：** 本次需求涉及哪个/哪些子项目？

## Step 2: 需求澄清与设计
**REQUIRED SUB-SKILL:** 调用 `brainstorming` Skill
- 设计文档保存到 `openspec/changes/<change-name>/design.md`（父目录下）
- 跨子项目变更需在设计中声明影响范围

## Step 3: 创建 OpenSpec 变更

在父目录执行（所有 openspec CLI 命令均在父目录）：
```bash
# [版本检查]
openspec new change "<change-name>" --json
openspec status --change "<change-name>" --json
openspec instructions propose --change "<change-name>" --json
# [版本恢复]
```

## Step 4: 生成实施计划
**REQUIRED SUB-SKILL:** 调用 `writing-plans` Skill
- 计划保存到 `openspec/changes/<change-name>/tasks.md`（父目录下）

# Step 5: 全局规范影响评估

基于已生成的 proposal.md、design.md、tasks.md，评估本次变更对各子项目规范文件的影响。

### 5.1 评估逻辑

逐子项目评估以下 3 个规范文件是否受影响：

| 规范文件 | 影响判断依据 |
|---|---|
| `architecture.md` | 是否引入新技术组件、通信模式、目录结构变更、模块依赖变更 |
| `coding-conventions.md` | 是否新增编码约定、修改命名规范、调整分层规则 |
| `business-domain.md` | 是否新增业务概念、修改业务流程、调整业务术语 |

### 5.2 生成 specs-impact.md

在 `openspec/changes/<change-name>/specs-impact.md` 中创建：

```markdown
# 全局规范影响评估

> 归档时将据此同步更新各子项目 `openspec/specs/` 下的对应文件。

## <子项目1>/openspec/specs/architecture.md
- **影响区域：** <受影响的章节名称，无影响则写"无影响">
- **变更原因：** <为什么需要更新>
- **变更类型：** ADD / UPDATE / REMOVE

## <子项目1>/openspec/specs/coding-conventions.md
- 无影响

## <子项目1>/openspec/specs/business-domain.md
- **影响区域：** <受影响的章节名称>
- **变更原因：** <为什么需要更新>
- **变更类型：** ADD / UPDATE / REMOVE

## <子项目2>/openspec/specs/architecture.md
...
```

**关键规则：**
- 无影响的规范文件写"无影响"，不要省略
- 变更类型：`ADD`（新增章节）、`UPDATE`（修改现有章节）、`REMOVE`（删除章节）
- 纯业务功能变更可能所有规范均为"无影响"，这是正常的
- 影响评估必须基于 design.md 中的实际设计决策，不要过度推断

## 完成标志

全流程完成后，输出：

```
## 提案完成

**变更名称：** <change-name>
**变更路径：** openspec/changes/<change-name>/
**涉及子项目：** <子项目列表>
**产出文件：**
  - proposal.md — 需求概述
  - design.md — 设计方案
  - tasks.md — 实施计划（N 个任务）
  - specs-impact.md — 全局规范影响评估

**规范影响：**
  - <子项目1>/architecture.md — 无影响 / ADD / UPDATE / REMOVE
  - <子项目1>/coding-conventions.md — 无影响
  - ...

**下一步：** 使用 `/<PROJECT_PREFIX>-apply` 开始实施
```

## 技术红线（贯穿全流程）

在任何步骤中，以下约束不可违反：

- 方案必须符合各子项目 `openspec/specs/architecture.md` 中声明的技术栈
- 方案必须符合各子项目 `openspec/specs/coding-conventions.md` 中的命名和分层规范
- 不得引入规范文件中未声明的新依赖
<!-- 如果有子项目配置了 MCP_TOOLS，保留此行 -->
- 数据库操作必须通过 MCP 工具（<MCP_TOOLS>）执行
- 跨子项目变更需在设计中明确标注影响范围

## 错误处理

| 情况 | 处理 |
|---|---|
| `openspec/` 目录不存在 | 提示在父目录执行 `openspec init --tools <AI_TOOL>` |
| brainstorming 阶段用户否决所有方案 | 回到 Step 2 重新探索 |
| `openspec new change` 失败 | 检查 CLI 版本和错误输出，报告给用户 |
| writing-plans 发现设计有缺陷 | 回退到 Step 2 补充设计 |
<!-- 如果任一子项目 NEED_NVM_SWITCH = true，保留此行 -->
| `nvm use <OPENSPEC_NODE_MAJOR>` 失败 | 提示用户安装 Node.js <OPENSPEC_NODE_MAJOR>：`nvm install <OPENSPEC_NODE_MAJOR>` |
| 目标子项目 specs 不存在 | 提示先使用 superspec-multi Skill 生成规范 |
```

> **跨工具适配说明：**
> - 以上模板为 Qoder 格式（含 YAML frontmatter、`<HARD-GATE>`、`REQUIRED SUB-SKILL:`）
> - **Claude Code**: 移除 YAML frontmatter（或保留为可选），`<HARD-GATE>` 改为 `> **MUST**`，`REQUIRED SUB-SKILL: 调用 brainstorming Skill` 改为内联步骤（见 TOOL-ADAPTER.md 4.1）
> - **Codex**: 嵌入 AGENTS.md 章节，标题降一级（`##` → `###`），`<HARD-GATE>` 改为 `> **[强约束]**`，所有子 Skill 调用改为内联步骤

### 4.2 Apply Skill 模板

<!-- AI_TOOL = qoder -->
路径：`.qoder/skills/<PROJECT_PREFIX>-apply/SKILL.md`

<!-- AI_TOOL = claude-code -->
路径：`.claude/commands/<PROJECT_PREFIX>-apply.md`

<!-- AI_TOOL = codex -->
路径：嵌入 `AGENTS.md` 的 `## Apply 工作流` 章节

<!-- AI_TOOL = codebuddy -->
路径：`.codebuddy/skills/<PROJECT_PREFIX>-apply/SKILL.md`

#### AI 提示词

```
基于以下模板，根据 AI_TOOL 变量创建 Apply Skill 文件（路径见上方条件分支）。
决策点：
- 如果任一子项目 NEED_NVM_SWITCH = true，保留 "Node.js 版本管理" 节
- 如果子项目 TECH_TYPE = frontend/mobile，TDD 检测条件为 jest.config/vitest.config/package.json test 脚本
- 如果子项目 TECH_TYPE = backend(Java)，TDD 检测条件为 pom.xml 中 spring-boot-starter-test / spring-boot-test / JUnit
- Step 3 编译验证命令使用各子项目的 BUILD_CMD
- 完成报告中的下一步指向 /<PROJECT_PREFIX>-archive
```

**模板：**

```markdown
---
name: <PROJECT_PREFIX>-apply
description: "多项目代码实施全流程：自动串联 OpenSpec apply → TDD → debugging → parallel-agents。在实现已批准的变更提案时使用。"
---

# <PROJECT_PREFIX> Apply — 多项目代码实施全流程

将 OpenSpec 变更提案中的任务逐个实现，全程遵循各子项目编码规范和质量要求。

**启动声明：** "我正在使用 <PROJECT_PREFIX>-apply Skill 执行多项目代码实施全流程。"

## 流程总览

```
OpenSpec 变更 → [Step 1] 加载上下文 → [Step 2] 执行任务循环 → [Step 3] 完成验证与报告
```

<!-- 如果任一子项目 NEED_NVM_SWITCH = true，包含此节 -->
## Node.js 版本管理
（同 Propose 模板的 Node.js 版本管理节）

## Step 1: 加载项目上下文（自动执行）

### 1.1 读取项目规范（全局规范读取）
读取所有子项目的规范：
- `<子项目1>/openspec/specs/architecture.md` — 架构约束
- `<子项目1>/openspec/specs/coding-conventions.md` — 编码规范
- `<子项目1>/openspec/specs/business-domain.md` — 业务规则
- `<子项目2>/openspec/specs/...`
- ...

同时读取父目录 `AGENTS.md` 中定义的统一规则和技术红线。

### 1.2 加载变更上下文
```bash
# [版本检查] 若子项目 NODE_BUILD_VERSION < 20，先执行: nvm use <OPENSPEC_NODE_MAJOR>

openspec list --json
openspec status --change "<change-name>" --json
openspec instructions apply --change "<change-name>" --json

# [版本恢复] 若之前切换过版本，执行: nvm use <NODE_BUILD_VERSION>
```

### 1.3 读取上下文文件
读取 `instructions apply` 返回的 `contextFiles` 中的所有文件。

### 1.4 展示当前进度
```
## 实施中: <change-name>
**涉及子项目:** <子项目列表>
**进度:** N/M 任务完成
**剩余任务:**
  - [ ] 任务 1: ...
  - [ ] 任务 2: ...
```

## Step 2: 执行任务循环

### 2.1 任务评估
- 判断是否需要**并行处理** → 调用 `dispatching-parallel-agents`
- 确认当前任务涉及哪个子项目

### 2.2 实现代码

**REQUIRED SUB-SKILL:** 调用 `test-driven-development` Skill（如项目已配置测试框架）

根据当前任务涉及的子项目类型，选择对应的开发模式：

<!-- TECH_TYPE = frontend/mobile -->
首先检测子项目是否已配置测试框架（检查 `jest.config.*`、`vitest.config.*` 或 `package.json` 中的 `test` 脚本）：

- **已配置测试框架** → 遵循 TDD 红绿循环：
  1. 先写失败测试
  2. 运行确认测试失败
  3. 写最小实现代码
  4. 运行确认测试通过

- **未配置测试框架** → 遵循编译驱动开发：
  1. 先写实现代码
  2. 运行 `<BUILD_CMD>` 确认编译通过
  3. 手动验证核心逻辑（浏览器/接口调用）
  4. 确保无 TypeScript 类型错误

<!-- TECH_TYPE = backend(Java) -->
首先检测子项目是否已配置单元测试（检查 `pom.xml` 中的 `spring-boot-starter-test`、`spring-boot-test` 或 JUnit 依赖）：

- **已有测试框架** → 遵循 TDD 红绿循环：
  1. 先写失败测试
  2. 运行 `mvn test` 确认测试失败
  3. 写最小实现代码
  4. 运行 `mvn test` 确认测试通过

  **Controller 层测试要求：**
  - 使用 `@SpringBootTest` 注解测试 Controller 层，验证接口入参、出参和请求类型
  - 测试类命名：`<Controller类名>Test`，如 `UserControllerTest`
  - 每个接口方法至少包含：正常场景 + 参数校验失败场景

- **无测试框架** → 遵循编译驱动开发：
  1. 先写实现代码
  2. 运行 `mvn clean compile` 确认编译通过
  3. 通过接口调用验证核心逻辑
  4. 确保无编译错误

<!-- 跨子项目变更 -->
跨子项目变更时，根据任务涉及的子项目类型分别应用对应的开发模式。

**编码过程中必须遵循对应子项目的 `openspec/specs/coding-conventions.md` 的全部约束。**

### 2.3 Bug 处理
遇到 Bug 时调用 `systematic-debugging` Skill，禁止猜测性修复。

### 2.4 标记完成
在 tasks 文件中将 `- [ ]` 改为 `- [x]`。

### 2.5 任务间检查点
每完成 3 个任务后暂停，展示进度摘要，等待用户确认后继续。

## Step 3: 完成验证与报告

所有任务完成后，对每个涉及的子项目执行编译验证：

```bash
# 前端子项目
cd <子项目>
<BUILD_CMD>
cd ..

# 后端子项目
cd <子项目>
<BUILD_CMD>
cd ..
```

**编译通过后**，输出完成报告，提示使用 `/<PROJECT_PREFIX>-archive` 进行归档。

## 暂停条件
- 任务描述不清楚
- 发现设计缺陷 → 建议回退到 `/<PROJECT_PREFIX>-propose`
- 遇到阻塞性错误
- 用户中断

## 技术红线（贯穿全流程）
- 代码必须符合对应子项目的 `openspec/specs/coding-conventions.md`
- 已配置测试框架时，禁止跳过测试步骤
- 禁止猜测性 Bug 修复
<!-- 如果有子项目 MCP_TOOLS 不为空 -->
- 数据库操作必须通过 MCP 工具执行
```

> **跨工具适配说明：**
> - **Claude Code / Codex**: `REQUIRED SUB-SKILL: 调用 test-driven-development Skill` 改为内联 TDD 步骤（见 TOOL-ADAPTER.md 4.3）
> - **Claude Code / Codex**: `REQUIRED SUB-SKILL: 调用 systematic-debugging Skill` 改为内联调试步骤（见 TOOL-ADAPTER.md 4.4）
> - 完成报告中的 `/<PROJECT_PREFIX>-archive` 触发方式按 AI_TOOL 调整

### 4.3 Archive Skill 模板

<!-- AI_TOOL = qoder -->
路径：`.qoder/skills/<PROJECT_PREFIX>-archive/SKILL.md`

<!-- AI_TOOL = claude-code -->
路径：`.claude/commands/<PROJECT_PREFIX>-archive.md`

<!-- AI_TOOL = codex -->
路径：嵌入 `AGENTS.md` 的 `## Archive 工作流` 章节

<!-- AI_TOOL = codebuddy -->
路径：`.codebuddy/skills/<PROJECT_PREFIX>-archive/SKILL.md`

#### AI 提示词

```
基于以下模板，根据 AI_TOOL 变量创建 Archive Skill 文件（路径见上方条件分支）。
决策点：
- Step 1.1 编译验证命令：按子项目类型（TECH_TYPE=frontend → BUILD_CMD + LINT_CMD；TECH_TYPE=backend → BUILD_CMD）
- Step 1.2 测试命令：HAS_TEST_FRAMEWORK = true → TEST_CMD；否则省略
- Step 1.3 规范合规检查：前端写"页面模块结构、Model 命名、路由注册"；后端写"Controller 命名、分层结构、实体类注解"
- 如果任一子项目 NEED_NVM_SWITCH = true，保留 Node.js 版本管理节
- 完成报告中的下一步指向正确的 Skill 名称
```

**模板：**

```markdown
---
name: <PROJECT_PREFIX>-archive
description: "多项目归档收尾全流程：自动串联 verification → code-review → OpenSpec archive。在所有任务实现完成后进行验证和归档时使用。"
---

# <PROJECT_PREFIX> Archive — 多项目归档收尾全流程

对已完成的 OpenSpec 变更进行验证、代码审查和归档，确保交付质量。

**启动声明：** "我正在使用 <PROJECT_PREFIX>-archive Skill 执行多项目归档收尾全流程。"

<HARD-GATE>
禁止在未运行验证命令并确认输出的情况下声称工作已完成。证据先于结论。
</HARD-GATE>

## 流程总览

```
实施完成 → [Step 1] 验证 → [Step 2] 代码审查 → [Step 3] 规范同步 → [Step 4] 归档
```

<!-- 如果任一子项目 NEED_NVM_SWITCH = true，包含此节 -->
## Node.js 版本管理
（同 Propose 模板）

## Step 1: 全面验证

**REQUIRED SUB-SKILL:** 调用 `verification-before-completion` Skill

### 1.1 编译与 Lint 验证

对每个涉及的子项目执行：

<!-- TECH_TYPE = frontend/mobile -->
```bash
cd <前端子项目>
<BUILD_CMD>
<!-- 如果 LINT_CMD 不为空 -->
<LINT_CMD>
cd ..
```
**判定标准：** 编译退出码为 0，Lint 0 error（warning 可接受）。

<!-- TECH_TYPE = backend(Java) -->
```bash
cd <后端子项目>
<BUILD_CMD>
cd ..
```
**判定标准：** 编译退出码为 0。

### 1.2 测试验证

对每个已配置测试框架的子项目执行：

<!-- HAS_TEST_FRAMEWORK = true -->
```bash
cd <子项目>
<TEST_CMD>
cd ..
```
**判定标准：** 所有测试通过，0 失败（如项目无测试脚本则跳过）。
<!-- HAS_TEST_FRAMEWORK = false，省略此子项目的测试 -->

### 1.3 规范合规检查

检查所有新增/修改的文件是否符合对应子项目的 `openspec/specs/coding-conventions.md`：

<!-- TECH_TYPE = frontend/mobile -->
- 确认页面模块结构符合规范
- 确认 Model/状态管理命名符合规范
- 确认路由注册符合规范
- 确认无违反技术红线的代码（如未声明的新依赖、非兼容语法等）

<!-- TECH_TYPE = backend -->
- 确认 Controller 命名符合规范
- 确认分层结构符合规范
- 确认实体类注解符合规范
- 确认无违反技术红线的代码（如高版本 Java 特性、未声明的新依赖等）

### 1.4 变更完整性
- 读取 `openspec/changes/<change-name>/tasks.md`
- 逐行检查每个任务是否标记为 `- [x]`
- 确认无遗漏任务

**验证失败时：** 停止流程，建议使用 `/<PROJECT_PREFIX>-apply` 继续修复。

## Step 2: 代码审查

**REQUIRED SUB-SKILL:** 调用 `requesting-code-review` Skill

### 2.1 准备审查上下文

```bash
# 在涉及的子项目中查看变更
cd <子项目>
git log --oneline --since="<变更开始时间>"
git diff <base-sha>..<head-sha> --stat
cd ..
```

### 2.2 执行审查

调用 code-reviewer 子代理进行审查，提供：

- **审查范围：** 本次变更涉及的所有子项目的提交
- **审查标准：** 各子项目 `openspec/specs/coding-conventions.md` + `architecture.md`
- **业务上下文：** 各子项目 `openspec/specs/business-domain.md`

### 2.3 处理审查反馈

- **Critical** → 立即修复，回到 Step 1 重新验证
- **Important** → 修复后继续
- **Minor** → 记录，不阻塞归档
- **审查者判断错误** → 用技术理由反驳

**REQUIRED SUB-SKILL:** 如有审查反馈需处理，调用 `receiving-code-review` Skill

## Step 3: 全局规范同步（自动执行）

### 3.1 读取影响评估
读取 `openspec/changes/<change-name>/specs-impact.md`：
- **文件不存在** → 跳过本步骤（兼容旧变更）
- **所有规范均为"无影响"** → 跳过本步骤

### 3.2 执行规范同步
对于每个标记了影响的规范文件：
1. 读取当前 `<子项目>/openspec/specs/<file>.md`
2. 读取变更的 `proposal.md` + `design.md` + `tasks.md`
3. 根据 specs-impact.md 更新对应规范文件（ADD/UPDATE/REMOVE）
4. 保持原有格式风格和章节结构

**关键约束：**
- 只更新 specs-impact.md 中标记了影响的规范文件
- 如果更新会破坏现有规范结构，暂停并提示用户确认

### 3.3 同步输出

```
## 规范同步完成
- <子项目1>/architecture.md — 无影响 / 已更新（ADD 1 节, UPDATE 0 节, REMOVE 0 节）
- <子项目1>/coding-conventions.md — 无影响
- <子项目1>/business-domain.md — 已更新（ADD 0 节, UPDATE 1 节）
- <子项目2>/... — ...
```

## Step 4: 归档变更（自动执行）

验证、审查和规范同步均通过后：

```bash
# [版本检查] 若子项目 NODE_BUILD_VERSION < 20，先执行: nvm use <OPENSPEC_NODE_MAJOR>

# 执行归档（使用 --skip-specs 跳过 OpenSpec 原生的 spec 更新，因为已在 Step 3 自行处理）
openspec archive --change "<change-name>" --skip-specs --json

# [版本恢复] 若之前切换过版本，执行: nvm use <NODE_BUILD_VERSION>
```

归档完成后输出：

```
## 归档完成

**变更名称：** <change-name>
**归档路径：** openspec/changes/archive/<timestamp>-<change-name>/
**涉及子项目：** <子项目列表>

### 规范同步
- <子项目1>/architecture.md — 无影响 / 已更新（ADD 1 节, UPDATE 0 节）
- <子项目1>/coding-conventions.md — 无影响
- <子项目1>/business-domain.md — 已更新
- ...

### 变更摘要
- **完成任务数：** N 个
- **修改文件数：** M 个
- **验证状态：** 编译通过 / 测试通过 / 规范合规
- **审查状态：** 已通过（Critical: 0, Important: 0）

### 产出物
- proposal.md — 需求概述
- design.md — 设计方案
- tasks.md — 实施记录（全部完成）
- specs-impact.md — 规范影响评估
```

## 暂停条件
- Step 1 验证失败 → 停止，报告失败详情
- Step 2 发现 Critical/Important 问题 → 停止，等待修复
- 用户中断 → 停止

## 技术红线（贯穿全流程）
- 验证必须基于实际命令输出，禁止"应该能通过"
- 代码审查必须基于子项目规范文件，而非通用最佳实践
- 归档前必须所有任务标记为完成
<!-- 如果有子项目 MCP_TOOLS 不为空 -->
- 数据库操作必须通过 MCP 工具执行
```

> **跨工具适配说明：**
> - **Claude Code / Codex**: `REQUIRED SUB-SKILL: 调用 verification-before-completion Skill` 改为内联验证步骤（见 TOOL-ADAPTER.md 4.5）
> - **Claude Code / Codex**: `REQUIRED SUB-SKILL: 调用 requesting-code-review Skill` 改为内联审查步骤（见 TOOL-ADAPTER.md 4.6）
> - **Claude Code / Codex**: `REQUIRED SUB-SKILL: 调用 receiving-code-review Skill` 改为内联步骤

---

## Step 5: 创建统一项目级指令文件

根据 `AI_TOOL` 变量创建对应的项目级指令文件。

<!-- AI_TOOL = qoder -->
创建 `AGENTS.md`。

<!-- AI_TOOL = claude-code -->
创建 `AGENTS.md` + `CLAUDE.md`。CLAUDE.md 为 Claude Code 专属指令文件，包含 MUST 约束区和命令引用。

<!-- AI_TOOL = codex -->
创建 `AGENTS.md`，三个工作流已在 Step 4 作为章节嵌入。内容无 `<HARD-GATE>` 标签，改用 `> **[强约束]**` 格式。

<!-- AI_TOOL = codebuddy -->
创建 `AGENTS.md` + `CODEBUDDY.md`。CODEBUDDY.md 为 CodeBuddy 专属指令文件，引用 AGENTS.md 中的完整规则。

路径：`<PARENT_DIR>/AGENTS.md`

### 5.1 AI 提示词

```
基于以下模板和各子项目变量表，创建父级目录的 AGENTS.md 文件。

填充指引：
- 项目上下文表：填入各子项目规范文件路径和内容描述
- 快速引用：按子项目填入技术栈摘要、构建命令、路径别名、HTTP 方案、路由方案
- propose 描述：包含 5 步（读规范 → brainstorming → OpenSpec → writing-plans → 规范影响评估）
- propose 下增加 "design.md 必须包含" 指引（按子项目类型：前端写路由设计/状态管理/API映射；后端写Controller/分层/数据库）
- apply 描述：TDD 条件化表述
- apply 下增加编码约束列表（每个子项目 6-8 条，引用 coding-conventions.md 的关键条目）
- archive 描述：包含验证 → 审查 → 规范同步 → 归档（--skip-specs）
- 日常编码规则：按子项目类型填入对应分类（前端：新增页面/修改页面/样式调整/表单开发；后端：修复 Bug/新增 API/修改数据库）
- 技术红线：从各子项目 TECH_CONSTRAINTS 填入
```

### 5.2 模板

```markdown
# AGENTS.md — <PARENT_DIR_NAME> 多项目统一规则

> 本文件是 OpenSpec + Superpowers 的连接层，定义 AI 在本工作区中编码时必须遵循的工作流和行为规范。

## 项目上下文

在编写任何代码前，必须先阅读以下规范文件获取项目知识：

### <子项目1>
| 规范 | 路径 | 内容 |
|---|---|---|
| 架构规范 | `<子项目1>/openspec/specs/architecture.md` | <架构规范内容摘要> |
| 编码规范 | `<子项目1>/openspec/specs/coding-conventions.md` | <编码规范内容摘要> |
| 业务域规范 | `<子项目1>/openspec/specs/business-domain.md` | <业务域规范内容摘要> |

### <子项目2>
| 规范 | 路径 | 内容 |
|---|---|---|
| ... | ... | ... |

### 快速参考

| 子项目 | 技术栈 | 构建命令 | 路径别名 | HTTP 方案 | 路由方案 |
|---|---|---|---|---|---|
| <子项目1> | <TECH_STACK> | <BUILD_CMD> | <PATH_ALIAS> | <HTTP方案> | <路由方案> |
| <子项目2> | <TECH_STACK> | <BUILD_CMD> | <PATH_ALIAS> | <HTTP方案> | <路由方案> |
| ... | ... | ... | ... | ... | ... |

## 通用规则

### OpenSpec 工作流
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：**团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ 中的项目级规范 > 代码推断**
- 禁止直接修改 openspec/specs/，必须通过 archive 流程更新
- 所有 openspec CLI 命令在父目录执行

### AI 行为约束
- 每步操作前声明目标子项目
- 跨子项目变更需在 propose 阶段声明影响范围
- 代码修改后必须运行对应子项目的构建/测试验证

### 全局规范读取
Skill 执行时必须读取所有子项目的规范文件，确保跨项目上下文完整：
- <子项目1>/openspec/specs/architecture.md
- <子项目1>/openspec/specs/coding-conventions.md
- <子项目1>/openspec/specs/business-domain.md
- <子项目2>/openspec/specs/...
- ...

## OpenSpec + Superpowers 集成工作流

> 以下三个阶段各对应一个统一 Skill，执行时自动串联多个 Superpowers Skill 和 OpenSpec CLI。

### 阶段一：需求提案 → 调用 `<PROJECT_PREFIX>-propose` Skill

当用户提出新功能或变更需求时，**直接调用 `<PROJECT_PREFIX>-propose` Skill**。

该 Skill 自动执行：读取所有子项目规范 → brainstorming → OpenSpec 变更创建 → writing-plans → 规范影响评估（specs-impact.md）。

**design.md 必须包含：**
<!-- 根据子项目 TECH_TYPE 填入 -->
<!-- TECH_TYPE = frontend/mobile -->
- 路由设计（菜单入口 / 功能页面的路径规划）
- 状态管理的 namespace 和 effects 定义
- 与后端接口的映射
<!-- TECH_TYPE = backend -->
- Controller 命名和 URL 路径设计
- Service/Mapper/Entity 分层规划
- 数据库表结构变更（如适用）
- API 接口入参/出参定义

### 阶段二：代码实现 → 调用 `<PROJECT_PREFIX>-apply` Skill

当开始实现已批准的变更时，**直接调用 `<PROJECT_PREFIX>-apply` Skill**。

该 Skill 自动执行：加载变更上下文 → TDD（如已配置测试框架）或编译驱动开发 → Bug 系统调试 → 并行任务分发。

代码必须符合对应子项目 `openspec/specs/coding-conventions.md` 的约束：
<!-- 根据子项目 TECH_TYPE 填入 6-8 条具体编码约束 -->
<!-- TECH_TYPE = frontend/mobile -->
- 页面模块结构: `index.tsx` + `index.less` + `model.ts` + `service.ts`
- 路由注册: 新增路由在路由配置文件中注册
- 路径别名: 使用项目约定的路径别名导入
- API 调用: 通过项目 HTTP 工具封装调用
- 样式: 使用项目预处理器和主题变量
- 新依赖: 禁止引入规范文件未声明的依赖
<!-- TECH_TYPE = backend -->
- 分层结构: controller → service → mapper → entity
- Controller 命名: 按项目命名约定
- ORM: 使用项目 ORM 框架，禁止手写 SQL（复杂查询除外）
- 实体类: 使用 Lombok 注解
- 新依赖: 禁止引入 pom.xml 未声明的依赖
- 数据库操作: 通过 MCP 工具执行

### 阶段三：归档收尾 → 调用 `<PROJECT_PREFIX>-archive` Skill

当所有任务实现完成后，**直接调用 `<PROJECT_PREFIX>-archive` Skill**。

该 Skill 自动执行：验证（编译/Lint/测试/规范检查） → 代码审查 → 全局规范同步 → OpenSpec 归档（--skip-specs）。

## 日常编码行为规则

<!-- 根据子项目 TECH_TYPE 选择对应的规则分类 -->
<!-- TECH_TYPE = frontend/mobile 的子项目 -->
### <前端子项目名> 日常规则

#### 新增页面
1. 在源代码目录下创建对应模块目录
2. 创建标准文件结构（如 index.tsx + model.ts + service.ts）
3. 在路由配置文件中注册路由
4. 如需全局数据，在全局服务中配置

#### 修改现有页面
1. 先阅读 model.ts / 状态管理理解数据流
2. 先阅读 service.ts 理解 API 调用链
3. 修改后确保路由参数兼容

#### 样式调整
1. 优先使用 UI 组件库内置样式
2. 自定义样式写在对应样式文件中
3. 全局主题色通过主题配置文件修改

#### 表单开发
1. 使用项目 UI 组件库的 Form 组件
2. 下拉数据源通过全局服务获取
3. 数据转换使用项目工具方法

<!-- TECH_TYPE = backend 的子项目 -->
### <后端子项目名> 日常规则

#### 修复 Bug
1. 调用 `systematic-debugging` Skill
2. 检查 `openspec/specs/business-domain.md` 确认业务逻辑正确性
3. 修复后调用 `verification-before-completion` Skill 验证

#### 新增 API 接口
1. 确认目标模块（多模块项目需确认子模块）
2. 按编码规范的命名约定创建 Controller
3. 创建对应的 Service + Mapper + Entity/DTO/VO
4. 编写单元测试

#### 修改数据库
1. 通过 MCP 工具操作数据库
2. 禁止绕过 MCP 使用脚本方式查询数据
3. DDL 变更需同步更新对应的 Mapper XML

<!-- 通用 -->
### 代码审查反馈
- 调用 `receiving-code-review` Skill 处理审查意见
- 对每条反馈验证技术正确性后再修改

## 子项目规则

### <子项目1>（<TECH_TYPE>）

**技术栈：** <TECH_STACK>
**技术红线：**
<TECH_CONSTRAINTS>

**编码要点：**
- <从 coding-conventions.md 提取的关键条目 1>
- <关键条目 2>
- ...

### <子项目2>（<TECH_TYPE>）
...

## 跨项目协作规则

- 跨子项目变更需在 propose 阶段声明影响范围
- 前后端联调变更的 specs-impact.md 需标注所有受影响子项目的规范文件路径
- 共享数据结构变更（如 API 接口）需同步评估所有消费方子项目

## 技术红线

<!-- 从各子项目 TECH_CONSTRAINTS 填入 -->
<版本锁定>
<禁止引入的依赖>
<禁止修改的配置>
<其他硬性约束>
```

<!-- AI_TOOL = claude-code -->
### 5.3 CLAUDE.md 模板（仅 Claude Code）

路径：`<PARENT_DIR>/CLAUDE.md`

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

<!-- AI_TOOL = codebuddy -->
### 5.4 CODEBUDDY.md 模板（仅 CodeBuddy）

路径：`<PARENT_DIR>/CODEBUDDY.md`

```markdown
# CODEBUDDY.md — <PARENT_DIR_NAME> 项目指令

> 本文件为 CodeBuddy 专属指令，引用 AGENTS.md 中的完整规则。

## 强制约束

- 每步操作完成后暂停，等待用户确认
- 禁止跳过项目结构检测直接创建文件
- 已有的 openspec/specs/*.md 文件严禁覆盖
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：团队规范基准文件 > openspec/specs/ > 代码推断

## 参考文件

- 完整项目规则见 AGENTS.md
- 各子项目规范见 <子项目>/openspec/specs/
- 操作指南见 OPENSPEC_SUPERPOWERS_GUIDE.md

## 命令

- /<PROJECT_PREFIX>-propose — 需求提案全流程
- /<PROJECT_PREFIX>-apply — 代码实施全流程
- /<PROJECT_PREFIX>-archive — 归档收尾全流程
```

---

## Step 6: 创建操作指南

路径：`<PARENT_DIR>/OPENSPEC_SUPERPOWERS_GUIDE.md`

### 6.1 模板

```markdown
# OpenSpec + Superpowers 操作指南

## 集成概述

### 什么是 OpenSpec

OpenSpec 是规范驱动开发工具，通过 `openspec/specs/` 下的规范文件（architecture.md、coding-conventions.md、business-domain.md）定义项目的架构、编码和业务约定，并通过 `openspec/changes/` 管理变更提案、设计和实施计划，让每次变更都有明确的规范依据。

### 什么是 Superpowers

Superpowers 是 AI 辅助开发 Skill 体系，包含 brainstorming（需求澄清）、writing-plans（计划生成）、test-driven-development（TDD）、systematic-debugging（系统调试）、code-review（代码审查）等可复用 Skill，让 AI 编码过程标准化、可追溯。

### 集成方式

将两者通过自定义 Skill 和 AGENTS.md 串联，形成端到端工作流：

```
需求提出 → /<PROJECT_PREFIX>-propose（brainstorming + OpenSpec 变更 + writing-plans + 规范影响评估）
      ↓
代码实现 → /<PROJECT_PREFIX>-apply（TDD / 编译驱动 + debugging + parallel-agents）
      ↓
归档收尾 → /<PROJECT_PREFIX>-archive（验证 + code-review + 规范同步 + OpenSpec 归档）
```

本工作区为 **multi-repo** 结构，集成方式为：父级目录统一管理 Skill/AGENTS.md/操作指南，各子项目独立维护 openspec/specs/ 规范文件。

## 项目概览

| 子项目 | 技术栈 | 类型 |
|---|---|---|
<!-- 按实际子项目填写 -->

## 快速开始

### 需求提案
```
/<PROJECT_PREFIX>-propose
```

### 执行实施
```
/<PROJECT_PREFIX>-apply
```

### 变更归档
```
/<PROJECT_PREFIX>-archive
```

## 各子项目规范位置

| 子项目 | 架构规范 | 编码规范 | 业务规范 |
|---|---|---|---|
| <子项目1> | openspec/specs/architecture.md | openspec/specs/coding-conventions.md | openspec/specs/business-domain.md |
| ... | ... | ... | ... |

## 新成员上手

1. 拉取所有子项目代码到同一父级目录
2. 用对应 AI 工具（Qoder / Claude Code / Codex / CodeBuddy）打开父级目录作为工作区
3. 执行 `/superspec-multi`（或自然语言指令），Skill 会自动完成：检测项目结构 → 检测 AI 工具 → 生成规范文件 → 创建统一 Skill → 生成项目级指令文件和本操作指南
4. 执行完毕后，输入 `/<PROJECT_PREFIX>-propose`（或对应触发方式）验证 Skill 可用

## 当前 AI 工具环境

| 配置项 | 值 |
|---|---|
| AI_TOOL | <codebuddy / claude-code / codex / qoder> |
| Skill 路径 | <根据 AI_TOOL 显示实际路径> |
| 触发方式 | <根据 AI_TOOL 显示触发方式> |
| OpenSpec init | `openspec init --tools <AI_TOOL>` |

## FAQ

### Q: 父级目录的文件为什么不能 git 提交？
A: 父级目录没有 .git 仓库。Skill、AGENTS.md、Guide 属于 AI 工具层配置，通过团队约定维护。

### Q: 如何添加新子项目？
A: 将新仓库 clone 到父级目录，为其创建 `openspec/specs/` 目录并生成规范文件，然后更新 AGENTS.md 和各 Skill 中的子项目列表。

### Q: 跨子项目变更怎么处理？
A: 在 propose 阶段声明涉及的所有子项目，在父目录创建一个 openspec change，specs-impact.md 中标注涉及的所有子项目规范。
```

---

## Step 7: Git 配置

各子项目的 `.gitignore` 配置需考虑：

```gitignore
# OpenSpec 规范文件（提交）
# openspec/specs/  ← 需要提交
```

**规则说明（multi-repo 场景）：**

| 路径 | 位置 | 是否提交 | 原因 |
|---|---|---|---|
| `openspec/specs/` | 子项目 | 提交 | 项目规范，团队共享 |
| 工具配置目录（`.qoder/` / `.claude/` / `.codebuddy/`） | 父目录 | 不可提交（无 .git） | AI 工具层配置 |
| `openspec/changes/` | 父目录 | 不可提交（无 .git） | 变更历史 |
| `AGENTS.md` | 父目录 | 不可提交（无 .git） | AI 规则 |
| `CLAUDE.md`（仅 Claude Code） | 父目录 | 不可提交（无 .git） | Claude Code 专属指令 |
| `CODEBUDDY.md`（仅 CodeBuddy） | 父目录 | 不可提交（无 .git） | CodeBuddy 专属指令 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 父目录 | 不可提交（无 .git） | 操作指南 |

> 注意：multi-repo 的父目录没有 `.git`，因此父目录下的文件天然不可 git 提交。子项目的 `openspec/specs/` 是唯一可提交的集成产出物。

---

## Step 8: 配置与验证

### 8.1 验证清单

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
[ ] Step 4-A: Propose Skill 已创建（路径匹配 AI_TOOL）且流程含 5 步
[ ] Step 4-B: Apply Skill 已创建（路径匹配 AI_TOOL）且 TDD 已条件化
[ ] Step 4-C: Archive Skill 已创建（路径匹配 AI_TOOL）且含 4 步（验证→审查→规范同步→归档）
[ ] Step 5-A: 项目级指令文件已生成（匹配 AI_TOOL）
[ ] Step 5-B: AGENTS.md 的 propose 描述包含"规范影响评估"
[ ] Step 5-C: AGENTS.md 的 apply 描述包含 TDD 条件化
[ ] Step 5-D: AGENTS.md 的 archive 描述包含"规范同步"和"--skip-specs"
[ ] Step 5-E: AGENTS.md 的 propose 下包含"design.md 必须包含"指引
[ ] Step 5-F: AGENTS.md 的 apply 下包含 6-8 条具体编码约束列表
[ ] Step 5-G: AGENTS.md 包含日常编码行为规则
[ ] Step 6: 父级 OPENSPEC_SUPERPOWERS_GUIDE.md 存在
[ ] Step 7: 子项目无工具配置目录（.qoder/ / .claude/ / .codebuddy/ 等）
[ ] 如果任一子项目 NEED_NVM_SWITCH=true，3 个 Skill 均包含 Node.js 版本管理节
[ ] Propose Skill 的 Step 5 紧接 Step 4，技术红线和错误处理在文件末尾
[ ] Archive Skill 的规范合规检查项与子项目类型匹配（前端无 Java 引用）
[ ] 功能验证: propose 命令可触发（方式匹配 AI_TOOL）
[ ] 功能验证: apply 命令可触发（方式匹配 AI_TOOL）
[ ] 功能验证: archive 命令可触发（方式匹配 AI_TOOL）
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

## 项目类型适配速查

### 前端项目（React/Vue）

| 配置项 | 值 |
|---|---|
| BUILD_CMD | `npm run build` |
| LINT_CMD | `npx eslint <src>/ --ext .ts,.tsx` |
| TEST_CMD | `npm test` |
| NEED_NVM_SWITCH | 通常 true（项目用 16/18，OpenSpec 用 <OPENSPEC_NODE_MAJOR>） |
| TDD 检测 | jest.config / vitest.config / package.json test 脚本 |
| 规范合规检查项 | 页面模块结构、Model 命名、路由注册、路径别名 |
| 技术红线示例 | 框架版本锁定、禁止未声明依赖、禁止修改 Webpack 配置 |

### 后端项目（Java/Spring Boot）

| 配置项 | 值 |
|---|---|
| BUILD_CMD | `mvn clean compile -pl <module> -am` |
| LINT_CMD | 无（或 checkstyle） |
| TEST_CMD | `mvn test -pl <module>` |
| NEED_NVM_SWITCH | 通常 false（后端不需要 nvm） |
| TDD 检测 | pom.xml 中 spring-boot-starter-test / spring-boot-test / JUnit 依赖 |
| 规范合规检查项 | Controller 命名、分层结构、实体类注解 |
| 技术红线示例 | Java 版本锁定、禁止未声明依赖、禁止修改 parent POM |

### 移动端项目（React Native/H5）

| 配置项 | 值 |
|---|---|
| BUILD_CMD | `npm run build` |
| LINT_CMD | `npx eslint <src>/ --ext .ts,.tsx` |
| TEST_CMD | `npm test` |
| NEED_NVM_SWITCH | 通常 true |
| TDD 检测 | 同前端 |
| 规范合规检查项 | 页面结构、路由（根路径如 /mobile）、移动端交互规则 |
| 技术红线示例 | 框架版本锁定、禁止 hover 效果、禁止绝对路径跳转 |

### Monorepo 子项目（前后端同仓库）

| 配置项 | 值 |
|---|---|
| SUB_PROJECT_TYPE | `monorepo` |
| 需要的额外变量 | `FRONTEND_DIR`、`BACKEND_DIR`、`FE_BUILD_CMD`、`BE_BUILD_CMD` 等 |
| 构建验证 | 前端 `<FE_BUILD_CMD>` + 后端 `<BE_BUILD_CMD>` 分别执行 |
| 测试验证 | 前端 `<FE_TEST_CMD>` + 后端 `<BE_TEST_CMD>` 分别执行 |
| TDD 检测 | 根据任务涉及目录判断前端/后端，分别检测对应测试框架 |
| 规范文件 | architecture.md 和 coding-conventions.md 分前端/后端两区域 |
| 技术红线示例 | 前端框架版本锁定 + Java 版本锁定 + 禁止跨层引用 |

---

## 执行顺序

```
Step 0: 检测 AI 工具环境 → 设置 AI_TOOL 变量
    ↓
Step 1: 检测 multi-repo 结构 → 确认子项目列表
    ↓
Step 2: 逐子项目分析 → 输出各子项目变量表
    ↓
Step 3: OpenSpec 初始化（父目录，按 AI_TOOL 选参数）+ 规范处理（子项目）
    ↓
Step 4: 创建 3 个统一 Skill（按 AI_TOOL 选择路径和格式）
    ↓
Step 5: 创建项目级指令文件（AGENTS.md，Claude Code 额外创建 CLAUDE.md，CodeBuddy 额外创建 CODEBUDDY.md）
    ↓
Step 6: 创建操作指南（含 AI_TOOL 环境说明）
    ↓
Step 7: Git 配置
    ↓
Step 8: 验证清单逐项检查（含工具特定检查项）
```

> **提示：** Step 4-6 的模板填充可以并行执行，但建议先完成 Step 4（Skill 创建），因为 AGENTS.md 和 Guide 中的描述需要与 Skill 实际步骤一致。
