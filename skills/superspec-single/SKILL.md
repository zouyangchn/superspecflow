---
name: superspec-single
description: "为 single 或 monorepo 项目集成 OpenSpec + Superpowers 开发工作流。按 6 步执行：项目分析 → OpenSpec 初始化 → Skill 创建 → AGENTS.md → 操作指南 → Git 配置。multi-repo 项目请使用 superspec-multi 技能。"
---

# OpenSpec + Superpowers 项目集成

为任意项目从零集成 OpenSpec（规范驱动开发）+ Superpowers（AI 辅助 Skill 体系），形成 propose → apply → archive 端到端工作流。

**启动声明：** "我正在使用 superspec-single Skill 执行项目集成。"

<HARD-GATE>
- 每个步骤完成后，暂停并向用户展示结果，等待确认后再进入下一步。
- 禁止跳过项目分析直接创建文件。模板填充必须基于实际项目代码分析。
- 已有的 openspec/specs/*.md 文件严禁覆盖，必须保留原有内容。
- 禁止在未确认变量表的情况下创建 Skill / AGENTS.md / Guide。
</HARD-GATE>

## 详细参考

本 Skill 是精简执行版。同目录下的 `PLAYBOOK.md` 包含各 Phase 的完整模板、填充指引、AI 提示词和决策点说明。
同目录下的 `TOOL-ADAPTER.md` 包含跨工具适配的格式示例、触发方式、子 Skill 内联模板和 Shell 脚本双版本。

**在创建 Skill 模板（Step 3）和 AGENTS.md（Step 4）时，优先读取 `PLAYBOOK.md` 中对应 Phase 的详细模板，并根据 `AI_TOOL` 变量参考 `TOOL-ADAPTER.md` 中的格式要求，确保产出完整。**

---

## 团队规范基准文件

本 Skill 在 Step 2 生成规范文件时，必须以以下三个团队级基准文件作为基线：

| 基准文件 | 路径 | 生成目标 |
|---|---|---|
| 编码规范基准 | `codebook.md`（`../../shared/`） | → `coding-conventions.md` |
| 架构规范基准 | `architecturebook.md`（`../../shared/`） | → `architecture.md` |
| 业务域规范基准 | `businessbook.md`（`../../shared/`） | → `business-domain.md` |

**使用规则：**
- 生成新规范文件时，**先读取对应基准文件**获取章节结构和团队通用条目，**再叠加**从项目代码分析中提取的特有约定
- 基准文件中的【强制】条目必须体现在项目规范中
- 基准文件位于 `../../shared/` 目录（superspecflow 包级共享），路径可通过 `../../config.json` 的 `specs.baselineDir` 配置

---

## 流程总览

```
[Step 0] 检测 AI 工具环境 → 确定 AI_TOOL 变量
    ↓
[Step 1] 项目分析 → 输出变量表
    ↓
[Step 2] OpenSpec 初始化 + 创建 3 个规范文件
    ↓
[Step 3] 创建 3 个自定义 Skill（propose / apply / archive）→ 路径和格式匹配 AI_TOOL
    ↓
[Step 4] 创建 AGENTS.md（+ 工具专属指令文件）
    ↓
[Step 5] 创建 OPENSPEC_SUPERPOWERS_GUIDE.md
    ↓
[Step 6] 配置 .gitignore + 验证清单逐项检查
```

---

## Step 0: 检测 AI 工具环境

检测当前 Skill 运行在哪个 AI 工具环境中，设置 `AI_TOOL` 变量，后续步骤根据此变量选择对应的 Skill 路径、格式和语法。

**检测原理：** 通过运行时信号（环境变量、进程、工作区目录）判断当前执行环境，而非检查系统中安装了哪些工具。优先级：CodeBuddy > Claude Code > Codex > Qoder（兜底）。如果自动检测不正确，可手动设置 `AI_TOOL` 环境变量覆盖。

**PowerShell 检测脚本（Windows）：**

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

## Step 1: 项目分析

分析项目技术栈、目录结构、编码规范、测试框架、Node.js 版本需求，输出变量表。

### 1.1 判断项目结构类型

**必须通过以下命令检测，禁止仅凭目录名称猜测：**

```bash
# Step A: 检查当前工作区根目录是否是 git 仓库
ls .git 2>$null || echo "NO_ROOT_GIT"

# Step B: 检查子目录是否各自有独立 .git
Get-ChildItem -Directory | ForEach-Object { if (Test-Path "$($_.Name)/.git") { Write-Output "$($_.Name) → 独立仓库" } }
```

**判断规则：**

| 检测结果 | 判定 | 处理方式 |
|---|---|---|
| 根目录有 `.git`，且仅一种技术栈 | **single** | 直接执行 Step 1.2-1.4 |
| 根目录有 `.git`，且同时包含前端+后端代码（如 package.json + pom.xml 共存） | **monorepo** | 执行 Step 1.2-1.4，变量表需填前端+后端两组 |
| 根目录**无** `.git`，子目录各自有独立 `.git` | **multi-repo** | ❌ 停止并提示用户：`当前项目是 multi-repo 结构，请使用 /superspec-multi 技能进行集成` |

> **易混淆场景：** 当工作区是一个包含多个子项目的容器目录（如 `cps-plat/` 下有 `cps-front-service/`、`cps-parent-domain/`），即使看起来"有前端+后端"，只要子目录各自有 `.git`，就是 **multi-repo**，不是 monorepo。

### 1.2 分析维度

对项目进行全面分析，覆盖以下 7 个维度：

1. **技术栈**：主框架版本、语言版本、构建工具、UI 组件库、路由方案、HTTP 通信、状态管理
2. **目录结构**：源代码根目录、模块组织方式、标准文件结构、路由配置位置、样式方案
3. **构建与验证命令**：启动命令、构建命令、测试命令、Lint 命令、是否配置测试框架
4. **编码规范**：命名规范、分层规范、路径别名、导入规范、样式规范、项目特有约定
5. **Node.js 版本**：项目构建需要的版本、是否与 OpenSpec CLI >= <OPENSPEC_NODE_MAJOR> 冲突、是否需要 nvm
6. **数据库**：数据库类型、ORM 框架、是否有 MCP 工具
7. **技术红线**：版本锁定、禁止引入的依赖、禁止修改的配置

### 1.3 输出变量表

将分析结果整理为变量表，后续步骤引用：

```
# === 基础变量 ===
PROJECT_NAME        = <项目名称>
PROJECT_STRUCTURE   = <single | monorepo>
PROJECT_TYPE        = <frontend | backend | mobile（monorepo 填主类型，如 frontend）>
IS_MULTI_MODULE     = <true | false>
MODULE_STRUCTURE    = <模块名和职责；单模块填"无">
PROJECT_PREFIX      = <Skill 命名前缀，如 myapp>
TECH_STACK          = <技术栈摘要>
BUILD_CMD           = <构建命令>
LINT_CMD            = <Lint 命令 / 无>
TEST_CMD            = <测试命令 / 无>
START_CMD           = <启动命令>
HAS_TEST_FRAMEWORK  = <true | false>
TEST_FRAMEWORK_NAME = <Jest / Vitest / JUnit / 无>
NEED_NVM_SWITCH     = <true | false>
NODE_BUILD_VERSION  = <项目构建 Node.js 版本>
PAGE_STRUCTURE      = <页面标准文件结构>
ROUTE_CONFIG_PATH   = <路由配置路径>
ROUTE_BASE          = <路由根路径>
PATH_ALIAS          = <路径别名>
DB_TYPE             = <数据库类型 / 无>
ORM_FRAMEWORK       = <ORM 框架 / 无>
MCP_TOOLS           = <可用 MCP 工具 / 无>
TECH_CONSTRAINTS    = <技术红线列表>

# === monorepo 专用变量（PROJECT_STRUCTURE = monorepo 时填写）===
FRONTEND_DIR        = <前端源码目录>
BACKEND_DIR         = <后端源码目录>
FE_BUILD_CMD        = <前端构建命令>
FE_LINT_CMD         = <前端 Lint 命令>
FE_TEST_CMD         = <前端测试命令>
FE_HAS_TEST         = <true | false>
FE_NEED_NVM         = <true | false>
BE_BUILD_CMD        = <后端构建命令>
BE_TEST_CMD         = <后端测试命令>
BE_HAS_TEST         = <true | false>
BE_NEED_NVM         = <false（后端通常不需要 nvm）>
```

**前缀取值规则：**
- 格式：2-5 个小写字母，简短易记
- 来源：从项目名称中提取缩写（如 `cps-front-service` → `cps-fs`，`myapp-web` → `mw`）
- 唯一性：同一团队内不同项目的前缀不应重复
- 示例：`cps-fs`（CPS 前端服务）、`cps-pd`（CPS 父域）、`mw`（MyApp Web）

**展示变量表给用户确认后，进入 Step 2。**

---

## Step 2: OpenSpec 初始化

### 2.1 安装与初始化

```bash
# 确认已全局安装
npm install -g @fission-ai/openspec

# 根据 Step 0 检测到的 AI_TOOL 选择对应的 init 参数
# AI_TOOL = qoder       → openspec init --tools qoder
# AI_TOOL = claude-code → openspec init --tools claude
# AI_TOOL = codex       → openspec init --tools codex
# AI_TOOL = codebuddy   → openspec init --tools codebuddy
openspec init --tools <AI_TOOL>

# 确认目录结构
# openspec/specs/ 已创建
<!-- AI_TOOL = qoder -->
# .qoder/skills/openspec-*/ 和 .qoder/commands/ 已生成
<!-- AI_TOOL = claude-code -->
# .claude/commands/ 已生成
<!-- AI_TOOL = codex -->
# AGENTS.md 已创建或更新
<!-- AI_TOOL = codebuddy -->
# .codebuddy/skills/openspec-*/ 和 .codebuddy/commands/ 已生成
```

### 2.2 规范文件处理（核心保护逻辑）

先检查 `openspec/specs/` 目录是否已有规范文件：

```bash
ls openspec/specs/*.md 2>/dev/null
```

**处理策略：**

| 文件 | 已存在 | 不存在 |
|---|---|---|
| architecture.md | 保留原有，不修改 | 基于 `architecturebook.md` + Step 1 分析结果生成 |
| coding-conventions.md | 保留原有，不修改 | 基于 `codebook.md` + Step 1 分析结果生成 |
| business-domain.md | 保留原有，不修改 | 基于 `businessbook.md` + Step 1 分析结果生成 |

**生成流程（两步走，仅对不存在的文件）：**

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

**重要约束：**
- 基准文件中的【强制】条目必须体现在项目规范中
- 项目特有约定不应与基准文件中的【强制】条目冲突
- 如有冲突，以基准文件为准

> monorepo 项目：architecture.md 和 coding-conventions.md 分"前端架构"和"后端架构"两大区域，business-domain.md 共享。

**展示每个规范文件的处理结果（保留/新生成）给用户确认后，进入 Step 3。**

---

## Step 3: 创建 3 个自定义 Skill

### 3.1 命名规范

根据 Step 0 检测到的 `AI_TOOL` 选择对应路径：

<!-- AI_TOOL = qoder -->
| Skill | 路径 |
|---|---|
| Propose | `.qoder/skills/<PROJECT_PREFIX>-propose/SKILL.md` |
| Apply | `.qoder/skills/<PROJECT_PREFIX>-apply/SKILL.md` |
| Archive | `.qoder/skills/<PROJECT_PREFIX>-archive/SKILL.md` |

格式：YAML frontmatter + Markdown，支持 `<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法。

<!-- AI_TOOL = claude-code -->
| Skill | 路径 |
|---|---|
| Propose | `.claude/commands/<PROJECT_PREFIX>-propose.md` |
| Apply | `.claude/commands/<PROJECT_PREFIX>-apply.md` |
| Archive | `.claude/commands/<PROJECT_PREFIX>-archive.md` |

格式：纯 Markdown（无 YAML frontmatter），约束用 `> **MUST**` 格式，子 Skill 调用改为内联步骤描述。

<!-- AI_TOOL = codex -->
| Skill | 路径 |
|---|---|
| Propose / Apply / Archive | 全部嵌入 `AGENTS.md` 的独立章节 |

格式：纯 Markdown，约束用 `> **[强约束]**` 格式，子 Skill 调用改为内联步骤描述。通过自然语言指令触发。

<!-- AI_TOOL = codebuddy -->
| Skill | 路径 |
|---|---|
| Propose | `.codebuddy/skills/<PROJECT_PREFIX>-propose/SKILL.md` |
| Apply | `.codebuddy/skills/<PROJECT_PREFIX>-apply/SKILL.md` |
| Archive | `.codebuddy/skills/<PROJECT_PREFIX>-archive/SKILL.md` |

格式：YAML frontmatter + Markdown（同 Qoder），支持 `<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法。

### 3.2 Propose Skill 模板

根据 `AI_TOOL` 在对应路径创建 Propose Skill，结构如下：

<!-- AI_TOOL = qoder -->
创建 `.qoder/skills/<PROJECT_PREFIX>-propose/SKILL.md`
<!-- AI_TOOL = claude-code -->
创建 `.claude/commands/<PROJECT_PREFIX>-propose.md`
<!-- AI_TOOL = codex -->
在 `AGENTS.md` 中添加 `## Propose 工作流 — 需求提案全流程` 章节
<!-- AI_TOOL = codebuddy -->
创建 `.codebuddy/skills/<PROJECT_PREFIX>-propose/SKILL.md`

**决策点：**
- `NEED_NVM_SWITCH = true` → 保留 Node.js 版本管理节
- `NEED_NVM_SWITCH = false` → 删除该节
- `MCP_TOOLS` 不为空 → 技术红线含"数据库操作必须通过 MCP 工具"

**模板：**

> **格式适配说明：**
> <!-- AI_TOOL = qoder / codebuddy --> Qoder 和 CodeBuddy 使用以下模板原样（含 YAML frontmatter、`<HARD-GATE>` 标签、`REQUIRED SUB-SKILL:` 语法）。
> <!-- AI_TOOL = claude-code --> Claude Code：去掉 YAML frontmatter，`<HARD-GATE>` 改为 `> **MUST**` 格式，`REQUIRED SUB-SKILL` 改为内联步骤描述（见下方替代块）。
> <!-- AI_TOOL = codex --> Codex：去掉 YAML frontmatter，作为 AGENTS.md 的 `##` 章节嵌入，`<HARD-GATE>` 改为 `> **[强约束]**` 格式，`REQUIRED SUB-SKILL` 改为内联步骤描述。

```markdown
---
name: <PROJECT_PREFIX>-propose
description: "项目需求提案全流程：自动串联 brainstorming → OpenSpec → writing-plans。"
---

# Propose — 需求提案全流程

**启动声明：** "我正在使用 <PROJECT_PREFIX>-propose Skill 执行需求提案全流程。"

<HARD-GATE>
在用户明确批准设计之前，禁止执行任何代码编写或实现操作。
</HARD-GATE>

## 流程总览

用户需求 → [Step 1] 读取项目规范 → [Step 2] brainstorming → [Step 3] OpenSpec 变更 → [Step 4] writing-plans → [Step 5] 规范影响评估

<!-- 如果 NEED_NVM_SWITCH = true，包含此节；否则删除 -->
## Node.js 版本管理

> **`<OPENSPEC_NODE_MAJOR>`** 变量从 `../../config.json` 的 `nodeVersion.openspecMinMajor` 字段读取（默认 20）。

OpenSpec CLI 依赖 Node.js >= <OPENSPEC_NODE_MAJOR>，而项目业务代码可能使用较低版本（如 <NODE_BUILD_VERSION>）。执行 openspec 命令前：
1. `node --version` 获取当前版本
2. 主版本 >= <OPENSPEC_NODE_MAJOR> → 直接执行
3. 主版本 < <OPENSPEC_NODE_MAJOR> → 记录当前版本，`nvm use <OPENSPEC_NODE_MAJOR>` 切换后执行
4. openspec 命令完毕后，`nvm use <原版本>` 恢复

## Step 1: 读取项目规范（自动执行）
读取 openspec/specs/architecture.md、coding-conventions.md、business-domain.md，检查 openspec/changes/ 下活跃变更。

## Step 2: 需求澄清与设计
**REQUIRED SUB-SKILL:** 调用 `brainstorming` Skill
- 设计文档保存到 `openspec/changes/<change-name>/design.md`
- 设计文档标注引用了哪些规范条目

## Step 3: 创建 OpenSpec 变更
（注意先按 Node.js 版本管理节检查版本）
\`\`\`bash
# [版本检查] 若 node 主版本 < <OPENSPEC_NODE_MAJOR>，先执行: nvm use <OPENSPEC_NODE_MAJOR>
openspec new change "<change-name>" --json
openspec status --change "<change-name>" --json
openspec instructions propose --change "<change-name>" --json
# [版本恢复] 若之前切换过版本，执行: nvm use <原版本>
\`\`\`
将设计写入 proposal.md、specs/、design.md。等待用户确认后进入 Step 4。

## Step 4: 生成实施计划
**REQUIRED SUB-SKILL:** 调用 `writing-plans` Skill
- 计划保存到 `openspec/changes/<change-name>/tasks.md`

## Step 5: 全局规范影响评估
评估对 architecture.md / coding-conventions.md / business-domain.md 的影响，生成 `openspec/changes/<change-name>/specs-impact.md`。
每个规范标注：无影响 / ADD / UPDATE / REMOVE。

## 完成标志
输出变更名称、路径、产出文件列表、规范影响摘要。下一步：`/<PROJECT_PREFIX>-apply`

## 技术红线（贯穿全流程）
- 方案必须符合 architecture.md 的技术栈
- 方案必须符合 coding-conventions.md 的命名和分层规范
- 不得引入规范文件中未声明的新依赖
<!-- 如果 MCP_TOOLS 不为空 -->
- 数据库操作必须通过 MCP 工具（<MCP_TOOLS>）执行

## 错误处理
| 情况 | 处理 |
|---|---|
| openspec/ 目录不存在 | 提示运行 `openspec init --tools <AI_TOOL>` |
| brainstorming 阶段用户否决 | 回到 Step 2 重新探索 |
| openspec new change 失败 | 检查 Node.js 版本和 CLI 错误输出 |
<!-- 如果 NEED_NVM_SWITCH = true -->
| nvm use <OPENSPEC_NODE_MAJOR> 失败 | 提示安装 Node.js <OPENSPEC_NODE_MAJOR>：`nvm install <OPENSPEC_NODE_MAJOR>` |
| writing-plans 发现设计缺陷 | 回退到 Step 2 |
```

> **Claude Code / Codex 内联替代（替换 REQUIRED SUB-SKILL 调用）：**
>
> **brainstorming（Step 2）替代：**
> ```
> 按以下步骤执行需求澄清与设计：
> 1. 读取 openspec/specs/ 下 3 个规范文件
> 2. 基于需求和规范，列出 3-5 个可行方案
> 3. 对每个方案评估：技术可行性、对现有规范的影响、实施复杂度
> 4. 推荐最优方案并说明理由
> 5. 等待用户确认方案后继续
> 6. 将确认的方案写入 openspec/changes/<change-name>/design.md
> ```
>
> **writing-plans（Step 4）替代：**
> ```
> 按以下步骤生成实施计划：
> 1. 读取 design.md 中的确认方案
> 2. 将方案分解为具体任务，每个任务包含：描述、预估复杂度
> 3. 按依赖关系排序任务
> 4. 将计划写入 openspec/changes/<change-name>/tasks.md
> ```

### 3.3 Apply Skill 模板

根据 `AI_TOOL` 在对应路径创建 Apply Skill，结构如下：

<!-- AI_TOOL = qoder -->
创建 `.qoder/skills/<PROJECT_PREFIX>-apply/SKILL.md`
<!-- AI_TOOL = claude-code -->
创建 `.claude/commands/<PROJECT_PREFIX>-apply.md`
<!-- AI_TOOL = codex -->
在 `AGENTS.md` 中添加 `## Apply 工作流 — 代码实施全流程` 章节
<!-- AI_TOOL = codebuddy -->
创建 `.codebuddy/skills/<PROJECT_PREFIX>-apply/SKILL.md`

**决策点：**
- `NEED_NVM_SWITCH = true` → 保留 Node.js 版本管理节
- `PROJECT_TYPE = frontend/mobile` → TDD 检测 jest/vitest/package.json
- `PROJECT_TYPE = backend(Java)` → TDD 检测 pom.xml 中 spring-boot-starter-test / spring-boot-test / JUnit
- `PROJECT_STRUCTURE = monorepo` → 按任务目录判断前端/后端
- `PROJECT_STRUCTURE = single` → 使用单栈验证

**模板：**

> **格式适配说明：**
> <!-- AI_TOOL = qoder / codebuddy --> Qoder 和 CodeBuddy 使用以下模板原样。
> <!-- AI_TOOL = claude-code --> Claude Code：去掉 YAML frontmatter，`REQUIRED SUB-SKILL` 改为内联步骤描述。
> <!-- AI_TOOL = codex --> Codex：去掉 YAML frontmatter，作为 AGENTS.md 章节嵌入，`REQUIRED SUB-SKILL` 改为内联步骤描述。

```markdown
---
name: <PROJECT_PREFIX>-apply
description: "项目代码实施全流程：自动串联 OpenSpec apply → TDD → debugging。"
---

# Apply — 代码实施全流程

**启动声明：** "我正在使用 <PROJECT_PREFIX>-apply Skill 执行代码实施全流程。"

## 流程总览
OpenSpec 变更 → [Step 1] 加载上下文 → [Step 2] 执行任务循环 → [Step 3] 完成验证与报告

<!-- 如果 NEED_NVM_SWITCH = true，包含 Node.js 版本管理节（同 Propose） -->

## Step 1: 加载项目上下文（自动执行）
### 1.1 读取 openspec/specs/ 下 3 个规范文件
### 1.2 加载变更上下文
\`\`\`bash
# [版本检查] 若 node 主版本 < <OPENSPEC_NODE_MAJOR>，先执行: nvm use <OPENSPEC_NODE_MAJOR>
openspec list --json
openspec status --change "<change-name>" --json
openspec instructions apply --change "<change-name>" --json
# [版本恢复] 若之前切换过版本，执行: nvm use <原版本>
\`\`\`
### 1.3 读取 contextFiles
### 1.4 展示当前进度

## Step 2: 执行任务循环
### 2.1 任务评估（判断是否需要并行 → dispatching-parallel-agents）
### 2.2 实现代码
**REQUIRED SUB-SKILL:** 调用 `test-driven-development` Skill（如已配置测试框架）

<!-- PROJECT_STRUCTURE = single, PROJECT_TYPE = frontend/mobile -->
检测测试框架（jest.config / vitest.config / package.json test 脚本）：
- 已配置 → TDD 红绿循环：1.先写失败测试 2.运行确认失败 3.写最小实现 4.运行确认通过
- 未配置 → 编译驱动：1.先写实现 2.<BUILD_CMD> 确认编译 3.手动验证 4.TS 类型检查

<!-- PROJECT_STRUCTURE = single, PROJECT_TYPE = backend(Java) -->
检测测试框架（pom.xml 中 spring-boot-starter-test / spring-boot-test / JUnit）：
- 已配置 → TDD 红绿循环：1.先写失败测试 2.mvn test 确认失败 3.写最小实现 4.mvn test 确认通过；Controller 层使用 `@SpringBootTest` 编写接口测试
- 未配置 → 编译驱动：1.先写实现 2.mvn clean compile 3.接口验证 4.无编译错误

<!-- PROJECT_STRUCTURE = monorepo -->
根据任务涉及目录判断前端/后端：
- 涉及 <FRONTEND_DIR>/ → 检测前端测试框架 → TDD 或编译驱动（<FE_BUILD_CMD>）
- 涉及 <BACKEND_DIR>/ → 检测后端测试框架 → TDD 或编译驱动（<BE_BUILD_CMD>）

### 2.3 Bug 处理
**REQUIRED SUB-SKILL:** 调用 `systematic-debugging` Skill（禁止猜测性修复）
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->
### 2.4 标记完成（- [ ] → - [x]）
### 2.5 每完成 3 个任务暂停，展示进度，等待确认

## Step 3: 完成验证与报告
<!-- PROJECT_STRUCTURE = single -->
\`\`\`bash
<BUILD_CMD>
\`\`\`
<!-- PROJECT_STRUCTURE = monorepo -->
\`\`\`bash
<FE_BUILD_CMD>
<BE_BUILD_CMD>
\`\`\`
编译通过后输出完成报告。下一步：`/<PROJECT_PREFIX>-archive`

## 暂停条件
- 任务描述不清楚 / 发现设计缺陷 → 回退到 `/<PROJECT_PREFIX>-propose`
- 遇到阻塞性错误 / 用户中断

## 技术红线（贯穿全流程）
- 代码必须符合 coding-conventions.md
- 已配置测试框架时，禁止跳过测试步骤
- 禁止猜测性 Bug 修复
<!-- 如果 MCP_TOOLS 不为空 -->
- 数据库操作必须通过 MCP 工具执行
```

> **Claude Code / Codex 内联替代（替换 REQUIRED SUB-SKILL 调用）：**
>
> **dispatching-parallel-agents（Step 2.1）替代：**
> ```
> 判断任务是否可并行执行：
> 1. 检查任务列表中是否存在相互独立的任务（无依赖关系）
> 2. 如存在独立任务，按模块分组，分别执行
> 3. 各组独立完成后，汇总结果并检查冲突
> 4. 如任务存在依赖，按顺序逐个执行
> ```
>
> **test-driven-development（Step 2.2）替代：**
> ```
> 按 TDD 红绿循环执行：
> 1. 阅读当前任务的实现要求
> 2. 先写失败测试（覆盖核心逻辑）
> 3. 运行测试确认失败
> 4. 写最小实现代码
> 5. 运行测试确认通过
> 6. 如有需要，重构代码并保持测试通过
>
> 未配置测试框架时，按编译驱动开发执行：
> 1. 先写实现代码
> 2. 运行 BUILD_CMD 确认编译通过
> 3. 手动验证核心逻辑
> ```
>
> **systematic-debugging（Step 2.3）替代：**
> ```
> 遇到 Bug 时按以下步骤系统调试：
> 1. 复现问题：记录错误信息和复现步骤
> 2. 定位范围：确定问题所在的模块
> 3. 收集证据：查看日志、断点调试、检查数据流
> 4. 形成假设：基于证据提出可能原因
> 5. 验证假设：逐一验证，禁止猜测性修复
> 6. 修复并验证：修复后运行测试确认
> ```

### 3.4 Archive Skill 模板

根据 `AI_TOOL` 在对应路径创建 Archive Skill，结构如下：

<!-- AI_TOOL = qoder -->
创建 `.qoder/skills/<PROJECT_PREFIX>-archive/SKILL.md`
<!-- AI_TOOL = claude-code -->
创建 `.claude/commands/<PROJECT_PREFIX>-archive.md`
<!-- AI_TOOL = codex -->
在 `AGENTS.md` 中添加 `## Archive 工作流 — 归档收尾全流程` 章节
<!-- AI_TOOL = codebuddy -->
创建 `.codebuddy/skills/<PROJECT_PREFIX>-archive/SKILL.md`

**决策点：**
- `PROJECT_TYPE = frontend/mobile` → 规范合规检查写"页面模块结构、Model 命名、路由注册"
- `PROJECT_TYPE = backend` → 规范合规检查写"Controller 命名、分层结构、实体类注解"
- `PROJECT_STRUCTURE = monorepo` → Step 1.1/1.2 双验证
- `HAS_TEST_FRAMEWORK = false` → 省略 Step 1.2 测试验证
- `NEED_NVM_SWITCH = true` → 保留 Node.js 版本管理节

**模板：**

> **格式适配说明：**
> <!-- AI_TOOL = qoder / codebuddy --> Qoder 和 CodeBuddy 使用以下模板原样。`<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法均可直接使用。
> <!-- AI_TOOL = claude-code --> Claude Code：去掉 YAML frontmatter，`<HARD-GATE>` 改为 `> **MUST**`，`REQUIRED SUB-SKILL` 改为内联步骤描述。
> <!-- AI_TOOL = codex --> Codex：去掉 YAML frontmatter，作为 AGENTS.md 章节嵌入，`<HARD-GATE>` 改为 `> **[强约束]**`，`REQUIRED SUB-SKILL` 改为内联步骤描述。

```markdown
---
name: <PROJECT_PREFIX>-archive
description: "项目归档收尾全流程：自动串联 verification → code-review → OpenSpec archive。"
---

# Archive — 归档收尾全流程

**启动声明：** "我正在使用 <PROJECT_PREFIX>-archive Skill 执行归档收尾全流程。"

<HARD-GATE>
禁止在未运行验证命令并确认输出的情况下声称工作已完成。证据先于结论。
</HARD-GATE>

## 流程总览
实施完成 → [Step 1] 验证 → [Step 2] 代码审查 → [Step 3] 规范同步 → [Step 4] 归档

<!-- 如果 NEED_NVM_SWITCH = true，包含 Node.js 版本管理节（同 Propose） -->

## Step 1: 全面验证
**REQUIRED SUB-SKILL:** 调用 `verification-before-completion` Skill

### 1.1 编译与 Lint 验证
<!-- PROJECT_STRUCTURE = single -->
\`\`\`bash
<BUILD_CMD>
<!-- 如果 LINT_CMD 不为空 -->
<LINT_CMD>
\`\`\`
判定标准：编译退出码为 0，Lint 0 error。

<!-- PROJECT_STRUCTURE = monorepo -->
\`\`\`bash
<FE_BUILD_CMD>
<FE_LINT_CMD>
<BE_BUILD_CMD>
\`\`\`
判定标准：前后端编译均退出码为 0，前端 Lint 0 error。

### 1.2 测试验证
<!-- PROJECT_STRUCTURE = single, HAS_TEST_FRAMEWORK = true -->
\`\`\`bash
<TEST_CMD>
\`\`\`
判定标准：所有测试通过，0 失败。
<!-- HAS_TEST_FRAMEWORK = false → 省略此节 -->

<!-- PROJECT_STRUCTURE = monorepo -->
\`\`\`bash
<FE_TEST_CMD>
<BE_TEST_CMD>
\`\`\`
判定标准：已配置的测试全部通过。

### 1.3 规范合规检查
- 检查所有新增/修改文件是否符合 coding-conventions.md
<!-- PROJECT_TYPE = frontend/mobile -->
- 确认页面模块结构、Model 命名、路由注册符合规范
- 确认无违反技术红线的代码（如未声明新依赖、非兼容语法）
<!-- PROJECT_TYPE = backend -->
- 确认 Controller 命名、分层结构、实体类注解符合规范
- 确认无违反技术红线的代码（如 Java 11+ 特性、未声明新依赖）

### 1.4 变更完整性
逐行检查 tasks.md 中每个任务标记为 - [x]。

## Step 2: 代码审查
**REQUIRED SUB-SKILL:** 调用 `requesting-code-review` Skill

### 2.1 准备审查上下文
\`\`\`bash
git log --oneline --since="<变更开始时间>"
git diff <base-sha>..<head-sha> --stat
\`\`\`
### 2.2 执行审查
调用 code-reviewer 子代理，提供审查标准（coding-conventions.md + architecture.md）和业务上下文（business-domain.md）。
### 2.3 处理审查反馈
- Critical → 立即修复，回到 Step 1
- Important → 修复后继续
- Minor → 记录，不阻塞
- 审查者判断错误 → 用技术理由反驳
**REQUIRED SUB-SKILL:** 如有审查反馈，调用 `receiving-code-review` Skill

## Step 3: 全局规范同步
### 3.1 读取 specs-impact.md（不存在则跳过；全"无影响"则跳过）
### 3.2 对标记了影响的规范文件执行 ADD/UPDATE/REMOVE
### 3.3 输出同步结果

## Step 4: 归档变更
\`\`\`bash
# [版本检查] 若 node 主版本 < <OPENSPEC_NODE_MAJOR>，先执行: nvm use <OPENSPEC_NODE_MAJOR>
openspec archive --change "<change-name>" --skip-specs --json
# [版本恢复] 若之前切换过版本，执行: nvm use <原版本>
\`\`\`
输出归档完成报告（变更名称、归档路径、规范同步状态、变更摘要、产出物）。

## 暂停条件
- Step 1 验证失败 → 停止，报告失败详情
- Step 2 发现 Critical/Important → 停止，等待修复
- 用户中断

## 技术红线
- 验证必须基于实际命令输出，禁止"应该能通过"
- 代码审查必须基于项目规范文件
- 归档前必须所有任务标记为完成
```

> **Claude Code / Codex 内联替代（替换 REQUIRED SUB-SKILL 调用）：**
>
> **verification-before-completion（Step 1）替代：**
> ```
> 按以下步骤执行全面验证：
> 1. 运行 BUILD_CMD，确认编译通过
> 2. 如已配置测试框架，运行 TEST_CMD，确认测试通过
> 3. 检查所有新增/修改文件是否符合 coding-conventions.md
> 4. 逐行检查 tasks.md 确认所有任务标记为 [x]
> 5. 汇总验证结果，任何失败项停止流程
> ```
>
> **requesting-code-review（Step 2）替代：**
> ```
> 按以下步骤执行代码审查：
> 1. 收集本次变更的 git diff
> 2. 审查标准：coding-conventions.md + architecture.md
> 3. 逐文件检查：命名规范、分层结构、技术红线、测试覆盖
> 4. 问题分级：Critical（阻塞）、Important（需修复）、Minor（记录）
> 5. 输出审查报告
> ```
>
> **receiving-code-review（Step 2.3）替代：**
> ```
> 按以下步骤处理审查反馈：
> 1. 逐条阅读审查意见，理解反馈意图
> 2. 对每条反馈验证技术正确性（对照规范文件判断）
> 3. Critical 反馈：立即修复，回到 Step 1 重新确认
> 4. Important 反馈：修复后继续
> 5. Minor 反馈：记录，不阻塞归档
> 6. 审查者判断错误 → 用技术理由反驳并记录
> ```

**3 个 Skill 创建完成后，展示文件列表给用户确认，进入 Step 4。**

---

## Step 4: 创建 AGENTS.md

在项目根目录创建 `AGENTS.md`，按以下结构填充：

```markdown
# <PROJECT_NAME> — AI 编码集成规则

> 本文件是 OpenSpec + Superpowers 的连接层。

## 项目上下文
- 规范优先级：团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ 中的项目级规范 > 代码推断

| 规范 | 路径 | 内容 |
|---|---|---|
| 架构规范 | openspec/specs/architecture.md | <摘要> |
| 编码规范 | openspec/specs/coding-conventions.md | <摘要> |
| 业务域规范 | openspec/specs/business-domain.md | <摘要> |

### 快速参考
- 技术栈 / 构建 / 路径别名 / HTTP / 路由
<!-- monorepo：分前端/后端两区域 -->

## 通用规则

### OpenSpec 工作流
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：**团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ 中的项目级规范 > 代码推断**
- 禁止直接修改 openspec/specs/，必须通过 archive 流程更新

### 全局规范读取
Skill 执行时必须读取项目规范文件，确保上下文完整：
- openspec/specs/architecture.md
- openspec/specs/coding-conventions.md
- openspec/specs/business-domain.md
<!-- monorepo：分前端/后端两路径列出 -->

## OpenSpec + Superpowers 集成工作流
> 以下三个阶段各对应一个项目级 Skill。

### 阶段一：需求提案 → 调用 `<PROJECT_PREFIX>-propose` Skill
该 Skill 自动执行：读取项目规范 → brainstorming → OpenSpec 变更创建 → writing-plans → 规范影响评估。

**design.md 必须包含**：
<!-- PROJECT_TYPE = frontend/mobile -->
- 路由设计、状态管理 namespace、与后端接口映射
<!-- PROJECT_TYPE = backend -->
- Controller 命名和 URL 路径、分层规划、API 入参/出参定义

### 阶段二：代码实现 → 调用 `<PROJECT_PREFIX>-apply` Skill
该 Skill 自动执行：加载变更上下文 → TDD（如已配置测试框架）或编译驱动开发 → Bug 调试。

代码必须符合 coding-conventions.md：
<!-- PROJECT_TYPE = frontend/mobile -->
- 页面模块结构、路由注册、路径别名、API 调用、样式规范、禁止未声明依赖
<!-- PROJECT_TYPE = backend -->
- 分层结构、Controller 命名、ORM 使用、Lombok 注解、禁止未声明依赖、MCP 数据库操作

### 阶段三：归档收尾 → 调用 `<PROJECT_PREFIX>-archive` Skill
该 Skill 自动执行：验证 → 代码审查 → 全局规范同步 → OpenSpec 归档（--skip-specs）。

## 日常编码行为规则
<!-- PROJECT_TYPE = frontend/mobile -->
### 新增页面 / 修改现有页面 / 样式调整 / 表单开发
<!-- PROJECT_TYPE = backend -->
### 修复 Bug / 新增 API 接口 / 修改数据库
<!-- 通用 -->
### 代码审查反馈 → 调用 receiving-code-review Skill

## 技术红线
<!-- 从 TECH_CONSTRAINTS 填入 -->
```

### 4.2 工具专属指令文件

根据 `AI_TOOL` 决定是否需要额外生成工具专属指令文件：

<!-- AI_TOOL = qoder -->
只需生成 `AGENTS.md`。Qoder 通过 `AGENTS.md` + `.qoder/skills/` 配合工作。

<!-- AI_TOOL = claude-code -->
需要额外生成 `CLAUDE.md`（Claude Code 在每次会话开始时读取）：

```markdown
# CLAUDE.md — <PROJECT_NAME> 项目指令

> 本文件为 Claude Code 专属指令，引用 AGENTS.md 中的完整规则。

## MUST（强制约束）

- 每步操作完成后暂停，等待用户确认
- 禁止跳过项目分析直接创建文件
- 已有的 openspec/specs/*.md 文件严禁覆盖
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ > 代码推断

## 参考文件

- 完整项目规则见 `AGENTS.md`
- 各规范见 `openspec/specs/`
- 操作指南见 `OPENSPEC_SUPERPOWERS_GUIDE.md`

## 命令

- `/<PROJECT_PREFIX>-propose` — 需求提案全流程
- `/<PROJECT_PREFIX>-apply` — 代码实施全流程
- `/<PROJECT_PREFIX>-archive` — 归档收尾全流程
```

<!-- AI_TOOL = codex -->
只需生成 `AGENTS.md`。Codex 原生读取 `AGENTS.md`，三个工作流已嵌入为独立章节。

<!-- AI_TOOL = codebuddy -->
需要额外生成 `CODEBUDDY.md`（CodeBuddy 在每次会话开始时读取）：

```markdown
# CODEBUDDY.md — <PROJECT_NAME> 项目指令

> 本文件为 CodeBuddy 专属指令，引用 AGENTS.md 中的完整规则。
> CodeBuddy 在每次会话开始时自动读取本文件。

## 强制约束

- 每步操作完成后暂停，等待用户确认
- 禁止跳过项目分析直接创建文件
- 已有的 openspec/specs/*.md 文件严禁覆盖
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ > 代码推断

## 参考文件

- 完整项目规则见 `AGENTS.md`
- 各规范见 `openspec/specs/`
- 操作指南见 `OPENSPEC_SUPERPOWERS_GUIDE.md`

## 命令

- `/<PROJECT_PREFIX>-propose` — 需求提案全流程
- `/<PROJECT_PREFIX>-apply` — 代码实施全流程
- `/<PROJECT_PREFIX>-archive` — 归档收尾全流程
```

---

## Step 5: 创建操作指南

在项目根目录创建 `OPENSPEC_SUPERPOWERS_GUIDE.md`，骨架如下：

```markdown
# OpenSpec + Superpowers 使用指南

## 1. 集成概述（OpenSpec / Superpowers / 集成方式）
## 2. 初始化配置
### 2.1 环境要求
| Node.js | >= <OPENSPEC_NODE_MAJOR> | <根据 NEED_NVM_SWITCH 填写：OpenSpec CLI 依赖 <OPENSPEC_NODE_MAJOR>+；系统环境默认配置前端构建 Node.js 版本，Skill 执行 openspec 时临时切换到 <OPENSPEC_NODE_MAJOR>>
<!-- backend：增加 Java/Maven 版本 -->
<!-- monorepo：同时包含 Node.js 和 Java/Maven -->
### 2.2 安装 OpenSpec CLI
### 2.3 克隆项目后的初始化步骤
1. cd <PROJECT_NAME>
2. <npm install（前端）/ 无（后端）/ 分别 install（monorepo）>
3. openspec init --tools <AI_TOOL>
4. 确认工具配置目录下包含自定义 Skill（路径根据 AI_TOOL 而定）
5. 确认 openspec/specs/ 下存在规范文件
6. 在 AI 编码工具中以项目为工作区打开
7. 加载/刷新 Skill（方式根据 AI 工具而定）
### 2.4 Superpowers 安装（全局一次性）
## 3. 使用规范
### 3.1 自定义 Skill 列表
### 3.2 Skill 详细说明（propose 5 步 / apply TDD 条件化 / archive 4 步）
## 4. 工作流程
### 4.1 标准开发流程
### 4.2 各阶段操作示例（apply 写"TDD（如已配置测试框架）或编译驱动方式"）
### 4.3 AGENTS.md 配置说明
## 5. 团队协作
### 5.1 新成员上手清单（包含 openspec init 步骤）
### 5.2 提交规范
### 5.3 常见问题（包含 openspec-* Skill 不存在 FAQ）
### 5.4 获取帮助
```

---

## Step 6: Git 配置 + 验证

### 6.1 配置 .gitignore

根据 `AI_TOOL` 在项目根目录 .gitignore 添加对应规则：

<!-- AI_TOOL = qoder -->
```
# OpenSpec auto-generated (regenerated via openspec init --tools qoder)
.qoder/commands/
.qoder/skills/openspec-*/
```

<!-- AI_TOOL = claude-code -->
```
# OpenSpec auto-generated (regenerated via openspec init --tools claude)
.claude/commands/openspec-*.md
```

<!-- AI_TOOL = codex -->
```
# OpenSpec auto-generated (regenerated via openspec init --tools codex)
# Codex 不生成独立配置文件，AGENTS.md 中的 openspec 章节为手动维护
```

<!-- AI_TOOL = codebuddy -->
```
# OpenSpec auto-generated (regenerated via openspec init --tools codebuddy)
.codebuddy/commands/
.codebuddy/skills/openspec-*/
```

**提交策略：**

<!-- AI_TOOL = qoder -->
| 路径 | 提交 | 原因 |
|---|---|---|
| `.qoder/skills/<PROJECT_PREFIX>-*/` | 是 | 自定义 Skill，团队共享 |
| `.qoder/skills/openspec-*/` | 否 | 自动生成，clone 后 openspec init 重新生成 |
| `.qoder/commands/` | 否 | 自动生成的斜杠命令 |
| `openspec/specs/` | 是 | 项目规范 |
| `openspec/changes/` | 是 | 变更历史 |
| `AGENTS.md` | 是 | 项目级 AI 规则 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 是 | 团队指南 |

<!-- AI_TOOL = claude-code -->
| 路径 | 提交 | 原因 |
|---|---|---|
| `.claude/commands/<PROJECT_PREFIX>-*.md` | 是 | 自定义命令，团队共享 |
| `.claude/commands/openspec-*.md` | 否 | 自动生成 |
| `openspec/specs/` | 是 | 项目规范 |
| `openspec/changes/` | 是 | 变更历史 |
| `AGENTS.md` | 是 | 项目级 AI 规则 |
| `CLAUDE.md` | 是 | Claude Code 专属指令 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 是 | 团队指南 |

<!-- AI_TOOL = codex -->
| 路径 | 提交 | 原因 |
|---|---|---|
| `AGENTS.md` | 是 | 项目级 AI 规则（含三个工作流章节） |
| `openspec/specs/` | 是 | 项目规范 |
| `openspec/changes/` | 是 | 变更历史 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 是 | 团队指南 |

<!-- AI_TOOL = codebuddy -->
| 路径 | 提交 | 原因 |
|---|---|---|
| `.codebuddy/skills/<PROJECT_PREFIX>-*/` | 是 | 自定义 Skill，团队共享 |
| `.codebuddy/skills/openspec-*/` | 否 | 自动生成 |
| `.codebuddy/commands/` | 否 | 自动生成的斜杠命令 |
| `openspec/specs/` | 是 | 项目规范 |
| `openspec/changes/` | 是 | 变更历史 |
| `AGENTS.md` | 是 | 项目级 AI 规则 |
| `CODEBUDDY.md` | 是 | CodeBuddy 专属指令 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 是 | 团队指南 |

### 6.2 验证清单

逐项检查（通过 ✅ 标记，未通过 ❌ 标记）：

```
[ ] Step 0: AI_TOOL 已检测并确认（codebuddy / claude-code / codex / qoder）
[ ] openspec/specs/ 下有 architecture.md、coding-conventions.md、business-domain.md
[ ] 规范文件已基于团队基准文件 + 项目分析生成（两步走）
[ ] 生成规范时已读取对应基准文件（codebook/architecturebook/businessbook）
[ ] AGENTS.md 中规范优先级包含团队基准文件
[ ] propose Skill 流程图含 5 步
[ ] apply Skill TDD 已条件化
[ ] archive Skill 含 4 步（验证→审查→规范同步→归档）
[ ] AGENTS.md propose 描述包含"规范影响评估"
[ ] AGENTS.md apply 描述包含 TDD 条件化
[ ] AGENTS.md archive 描述包含"规范同步"和"--skip-specs"
[ ] AGENTS.md propose 下包含"design.md 必须包含"指引
[ ] AGENTS.md apply 下包含 6-8 条编码约束列表
[ ] AGENTS.md 包含日常编码行为规则
[ ] Guide 2.3 包含 openspec init --tools <AI_TOOL> 步骤
[ ] Guide 5.1 上手清单包含 openspec init 步骤
[ ] Guide 4.2 apply 示例反映 TDD 条件化
[ ] Guide 5.3 FAQ 包含 openspec-* Skill 排错条目
[ ] .gitignore 包含对应工具的自动生成忽略规则
[ ] NEED_NVM_SWITCH=true 时，3 个 Skill 均含 Node.js 版本管理节
[ ] NEED_NVM_SWITCH=true 时，Guide 2.1 Node.js 说明为"临时切换"表述
[ ] propose Skill Step 5 紧接 Step 4，技术红线和错误处理在文件末尾
[ ] archive Skill 规范合规检查项与项目类型匹配
[ ] 3 个 Skill 路径和格式匹配 AI_TOOL
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

> **monorepo 专项**（仅当 PROJECT_STRUCTURE = monorepo 时显示，single 项目跳过此段）：
> ```
> [ ] 变量表含 FRONTEND_DIR / BACKEND_DIR / FE_BUILD_CMD / BE_BUILD_CMD 等
> [ ] apply Skill Step 2.2 含按目录判断前端/后端的 TDD 双模式
> [ ] archive Skill Step 1.1 含前端+后端双编译验证
> [ ] architecture.md 和 coding-conventions.md 分前端/后端两区域
> [ ] AGENTS.md 日常编码规则含前端+后端两套分类
> [ ] Guide 2.1 环境要求同时包含 Node.js 和 Java/Maven
> ```

---

## 项目类型适配速查

| 配置项 | 前端(React/Vue) | 后端(Java) | 移动端(H5) | Monorepo |
|---|---|---|---|---|
| BUILD_CMD | npm run build | mvn clean compile -pl \<module\> -am | npm run build | FE + BE 分别 |
| LINT_CMD | npx eslint | 无/checkstyle | npx eslint | FE 有, BE 无 |
| TEST_CMD | npm test | mvn test | npm test | FE + BE 分别 |
| NEED_NVM_SWITCH | 通常 true | 通常 false | 通常 true | FE 可能 true |
| TDD 检测 | jest/vitest/package.json | pom.xml JUnit | 同前端 | 按目录判断 |
| 规范合规检查 | 页面结构/Model/路由 | Controller/分层/注解 | 页面/路由/交互 | 两套 |
| 技术红线 | 框架版本锁定 | Java 版本锁定 | 框架+移动端规则 | 前后端红线 |
