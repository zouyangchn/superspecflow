# OpenSpec + Superpowers 项目集成操作手册

> 本手册指导如何为任意项目集成 OpenSpec + Superpowers 开发工作流。按 Phase 1 → 6 顺序执行，每个 Phase 包含 AI 提示词和模板。

**AI_TOOL 变量：** 由 SKILL.md Step 0 检测，值为 `codebuddy | claude-code | codex | qoder`。后续各 Phase 根据此变量选择对应的路径、格式和语法。

---

## 1. 概述

### 1.1 集成目标

将 OpenSpec（规范驱动开发）与 Superpowers（AI 辅助开发 Skill 体系）通过自定义 Skill 和 AGENTS.md 串联，形成端到端工作流：

```
propose（规范约束下的设计） → apply（规范驱动的实现） → archive（验证+规范同步+归档）
```

### 1.2 产出物清单

| 产出物 | 路径 | 说明 |
|---|---|---|
| 规范文件 | `openspec/specs/*.md` | 架构、编码、业务域规范 |
| Propose Skill | 见下方条件路径 | 需求提案全流程 |
| Apply Skill | 见下方条件路径 | 代码实施全流程 |
| Archive Skill | 见下方条件路径 | 归档收尾全流程 |
| AGENTS.md | `AGENTS.md` | 项目级 AI 行为规则 |
| 操作指南 | `OPENSPEC_SUPERPOWERS_GUIDE.md` | 面向团队成员的使用指南 |
| Git 配置 | `.gitignore` | 忽略 OpenSpec 自动生成的文件 |

**Skill 路径（根据 AI_TOOL）：**
<!-- AI_TOOL = qoder -->
| Skill | 路径 |
|---|---|
| Propose | `.qoder/skills/<prefix>-propose/SKILL.md` |
| Apply | `.qoder/skills/<prefix>-apply/SKILL.md` |
| Archive | `.qoder/skills/<prefix>-archive/SKILL.md` |

<!-- AI_TOOL = claude-code -->
| Skill | 路径 |
|---|---|
| Propose | `.claude/commands/<prefix>-propose.md` |
| Apply | `.claude/commands/<prefix>-apply.md` |
| Archive | `.claude/commands/<prefix>-archive.md` |

<!-- AI_TOOL = codex -->
| Skill | 路径 |
|---|---|
| Propose / Apply / Archive | 全部嵌入 `AGENTS.md` 的独立章节 |

<!-- AI_TOOL = codebuddy -->
| Skill | 路径 |
|---|---|
| Propose | `.codebuddy/skills/<prefix>-propose/SKILL.md` |
| Apply | `.codebuddy/skills/<prefix>-apply/SKILL.md` |
| Archive | `.codebuddy/skills/<prefix>-archive/SKILL.md` |

### 1.3 前置条件

- Node.js >= <OPENSPEC_NODE_MAJOR>（运行 OpenSpec CLI）
- AI 编码工具（CodeBuddy / Claude Code / Codex / Qoder，已安装 Superpowers 或等效 Skill 体系）
- 项目已有可运行的构建命令

---

## 工具配置映射

> 本节定义四个支持工具的配置差异，作为各 Phase 条件分支的参考源。
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

**AI_TOOL 变量：** 由 SKILL.md Step 0 检测，值为 `codebuddy | claude-code | codex | qoder`。后续各 Phase 根据此变量选择对应的路径、格式和语法。

---

## 2. Phase 1: 项目分析

> **目标**：分析项目技术栈、目录结构、编码规范、测试框架、Node.js 版本需求，为后续模板填充提供依据。

### AI 提示词

```
请对当前项目进行全面分析，输出以下信息：

**首先判断项目结构类型（必须通过 .git 检测，禁止仅凭目录名称猜测）：**

执行以下检测命令：
```bash
# Step A: 检查当前工作区根目录是否是 git 仓库
ls .git 2>$null || echo "NO_ROOT_GIT"

# Step B: 检查子目录是否各自有独立 .git
Get-ChildItem -Directory | ForEach-Object { if (Test-Path "$($_.Name)/.git") { Write-Output "$($_.Name) → 独立仓库" } }
```

根据检测结果判断：
- **single** — 根目录有 `.git`，且仅一种技术栈（纯前端 / 纯后端 / 纯移动端）
- **monorepo** — 根目录有 `.git`，且同时包含前端+后端代码（如 package.json + pom.xml 共存于同一仓库）
- **multi-repo** — 根目录**无** `.git`，子目录各自有独立 `.git`（如 cps-plat 的 3 个子项目）→ **停止并提示用户**：

  > 当前项目是 multi-repo 结构，请使用 `/superspec-multi` 技能进行集成。

> **易混淆场景：** 当工作区是一个包含多个子项目的容器目录（如 `cps-plat/`），即使看起来“有前端+后端”，只要子目录各自有 `.git`，就是 **multi-repo**，不是 monorepo。

以下分析维度适用于 single 和 monorepo 项目结构。

## 1. 技术栈
- 主框架和版本（如 React 17 / Vue 3 / Spring Boot 2.x）
- 语言和版本（如 TypeScript 4.3 / Java 1.8）
- 构建工具（如 Webpack 5 / Vite / Maven）
- UI 组件库（如 Ant Design / Element Plus）
- 路由方案（如 DVA / React Router / Spring MVC）
- HTTP 通信方案（如 axios / fetch / HttpUtils）
- 状态管理方案（如 DVA Model / Redux / Pinia）

## 2. 目录结构
- 源代码根目录（如 app/ / src/ / src/main/java/）
- 页面/模块组织方式（如按功能分目录）
- 标准文件结构（如 index.tsx + model.ts + service.ts）
- 路由配置位置（如 app/index.tsx）
- 样式方案（如 Less / Sass / CSS Modules）

## 3. 构建与验证命令
- 启动命令（如 npm run start / mvn spring-boot:run）
- 构建命令（如 npm run build / mvn clean compile）
- 测试命令（如 npm test / mvn test）
- Lint 命令（如 npx eslint / checkstyle）
- 是否已配置测试框架？如配置，是什么？（Jest / Vitest / JUnit）

## 4. 编码规范
- 命名规范（组件、文件、变量、接口）
- 分层规范（如 controller → service → mapper → entity）
- 路径别名（如 current-app → app/）
- 导入规范
- 样式规范

## 5. Node.js 版本需求
- 项目构建需要的 Node.js 版本（检查 .nvmrc / engines 字段）
- 是否与 OpenSpec CLI 要求的 >= <OPENSPEC_NODE_MAJOR> 存在冲突？
- 是否需要 nvm 版本管理逻辑？

## 6. 数据库
- 数据库类型（MySQL / OceanBase / PostgreSQL）
- ORM 框架（MyBatis-Plus / TypeORM / Prisma）
- 是否有 MCP 工具可用？

## 7. 技术红线
- 版本锁定（如禁止升级 React 18）
- 禁止引入的依赖
- 禁止修改的配置文件
- 其他硬性约束
```

### 分析结果格式

将分析结果整理为以下变量表，后续 Phase 填充模板时引用：

```
PROJECT_NAME        = <项目名称，如 cps-front-service>
PROJECT_STRUCTURE  = <single | monorepo>
PROJECT_TYPE        = <frontend | backend | mobile（monorepo 填主类型，如 frontend）>
IS_MULTI_MODULE     = <true | false，是否多模块项目（如 Maven 多模块）>
MODULE_STRUCTURE    = <模块名和职责，如 cps-common-domain(公共) / cps-admin-domain(管理) / cps-core-domain(核心)；单模块项目填"无">
PROJECT_PREFIX      = <Skill 命名前缀，如 cps-fs / cps-pd / cps-ms>
TECH_STACK          = <技术栈摘要，如 React 17 + TypeScript 4.3 + DVA 2.4>
BUILD_CMD           = <构建命令，如 npm run build / mvn clean compile>
LINT_CMD            = <Lint 命令，如 npx eslint app/ --ext .ts,.tsx / 无>
TEST_CMD            = <测试命令，如 npm test / mvn test / 无>
START_CMD           = <启动命令，如 npm run start>
HAS_TEST_FRAMEWORK  = <true | false>
TEST_FRAMEWORK_NAME = <Jest / Vitest / JUnit / 无>
NEED_NVM_SWITCH     = <true | false，项目构建 Node.js 版本是否 < <OPENSPEC_NODE_MAJOR>>
NODE_BUILD_VERSION  = <项目构建 Node.js 版本，如 16 / 18 / 20>
PAGE_STRUCTURE      = <页面标准文件结构，如 index.tsx + model.ts + service.ts>
ROUTE_CONFIG_PATH   = <路由配置路径，如 app/index.tsx>
ROUTE_BASE          = <路由根路径，如 / / /mobile>
PATH_ALIAS          = <路径别名，如 current-app → app/>
DB_TYPE             = <数据库类型，如 OceanBase>
ORM_FRAMEWORK       = <ORM 框架，如 MyBatis-Plus>
MCP_TOOLS           = <可用 MCP 工具，如 oceanbase-cps, oceanbase-lake>
TECH_CONSTRAINTS    = <技术红线列表>

# === 以下变量仅 PROJECT_STRUCTURE = monorepo 时填写 ===
FRONTEND_DIR        = <前端源码目录，如 frontend/ 或 web/>
BACKEND_DIR         = <后端源码目录，如 backend/ 或 server/>
FE_BUILD_CMD        = <前端构建命令，如 cd frontend && npm run build>
FE_LINT_CMD         = <前端 Lint 命令，如 cd frontend && npx eslint src/ --ext .ts,.tsx>
FE_TEST_CMD         = <前端测试命令，如 cd frontend && npm test>
FE_HAS_TEST         = <true | false，前端是否配置测试框架>
FE_NEED_NVM         = <true | false，前端构建 Node.js 版本是否 < <OPENSPEC_NODE_MAJOR>>
BE_BUILD_CMD        = <后端构建命令，如 cd backend && mvn clean compile>
BE_TEST_CMD         = <后端测试命令，如 cd backend && mvn test>
BE_HAS_TEST         = <true | false，后端是否配置单元测试>
BE_NEED_NVM         = <false（后端通常不需要 nvm）>
```

**前缀取值规则：**
- 格式：2-5 个小写字母，简短易记
- 来源：从项目名称中提取缩写（如 `cps-front-service` → `cps-fs`，`myapp-web` → `mw`）
- 唯一性：同一团队内不同项目的前缀不应重复
- 示例：`cps-fs`（CPS 前端服务）、`cps-pd`（CPS 父域）、`mw`（MyApp Web）

---

## 3. Phase 2: OpenSpec 初始化

### AI 提示词

```
基于 Phase 1 的项目分析结果，执行以下操作：

1. 确认已全局安装 OpenSpec CLI（npm install -g @fission-ai/openspec）
2. 在项目根目录执行：
   # 根据 Step 0 检测到的 AI_TOOL 选择对应的 init 参数
   # AI_TOOL = qoder       → openspec init --tools qoder
   # AI_TOOL = claude-code → openspec init --tools claude
   # AI_TOOL = codex       → openspec init --tools codex
   # AI_TOOL = codebuddy   → openspec init --tools codebuddy
   openspec init --tools <AI_TOOL>
3. 确认 openspec/specs/ 目录已创建
4. 确认工具配置文件已生成：
   <!-- AI_TOOL = qoder -->
   # .qoder/skills/openspec-*/ 和 .qoder/commands/ 已生成
   <!-- AI_TOOL = claude-code -->
   # .claude/commands/ 已生成
   <!-- AI_TOOL = codex -->
   # AGENTS.md 已创建或更新
   <!-- AI_TOOL = codebuddy -->
   # .codebuddy/skills/openspec-*/ 和 .codebuddy/commands/ 已生成
```

### 规范文件处理（核心保护逻辑）

> **核心原则：已有的不覆盖，缺失的才生成。** 先检查 `openspec/specs/` 目录是否已有规范文件，已存在的保留原有内容不做任何修改。

#### AI 提示词

```
先检查 openspec/specs/ 目录下是否已有规范文件：
  ls openspec/specs/*.md 2>/dev/null

处理策略：
- architecture.md：已存在 → 保留原有，不修改；不存在 → 基于基准文件 + 项目分析生成
- coding-conventions.md：已存在 → 保留原有，不修改；不存在 → 基于基准文件 + 项目分析生成
- business-domain.md：已存在 → 保留原有，不修改；不存在 → 基于基准文件 + 项目分析生成

**仅对不存在的文件，执行以下生成流程（两步走）：**
Step A: 读取基准文件 → 获取章节骨架 + 团队通用条目
Step B: 叠加项目分析结果 → 填充项目特有的技术栈/目录/命名/业务等内容

各文件对应的基准文件：
- architecture.md ← architecturebook.md（`../../shared/`）
- coding-conventions.md ← codebook.md（`../../shared/`）
- business-domain.md ← businessbook.md（`../../shared/`）

重要约束：
- 基准文件中的【强制】条目必须体现在项目规范中
- 项目特有约定不应与基准文件中的【强制】条目冲突
- 如有冲突，以基准文件为准
- 每个文件的内容必须基于实际项目代码分析，不要泛泛而谈

### architecture.md
以 architecturebook.md 的章节结构为骨架，填充：
- 项目技术栈（框架、语言、版本）
- 目录结构说明（源代码根目录、模块组织方式）
- 模块结构与依赖关系（多模块项目）
- 构建配置、路由架构、HTTP 通信方案、中间件配置、环境配置

> 如果 PROJECT_STRUCTURE = monorepo，分”前端架构”和”后端架构”两大区域。

### coding-conventions.md
以 codebook.md 的章节结构为骨架，填充：
- 项目命名规范、页面/模块标准文件结构、分层规范
- 样式规范（前端）、路径别名、导入导出规范
- **项目特定规范**：分析代码中存在的特有约定

### business-domain.md
以 businessbook.md 的章节结构为骨架，填充：
- 核心业务概念和术语、主要功能模块
- 业务流程说明、接口映射关系、数据模型概要
```

---

## 4. Phase 3: 创建自定义 Skill

### 4.1 命名规范

Skill 命名格式：`<PROJECT_PREFIX>-<phase>`

| Skill | 名称格式 | 示例 |
|---|---|---|
| Propose | `<PROJECT_PREFIX>-propose` | `cps-fs-propose` |
| Apply | `<PROJECT_PREFIX>-apply` | `cps-fs-apply` |
| Archive | `<PROJECT_PREFIX>-archive` | `cps-fs-archive` |

存放路径（根据 AI_TOOL 选择）：

<!-- AI_TOOL = qoder -->
`.qoder/skills/<PROJECT_PREFIX>-<phase>/SKILL.md`
格式：YAML frontmatter + Markdown，支持 `<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法。

<!-- AI_TOOL = claude-code -->
`.claude/commands/<PROJECT_PREFIX>-<phase>.md`
格式：纯 Markdown（无 YAML frontmatter），约束用 `> **MUST**` 格式，子 Skill 调用改为内联步骤描述。

<!-- AI_TOOL = codex -->
嵌入 `AGENTS.md` 的独立 `##` 章节
格式：纯 Markdown，约束用 `> **[强约束]**` 格式，子 Skill 调用改为内联步骤描述。通过自然语言指令触发。

<!-- AI_TOOL = codebuddy -->
`.codebuddy/skills/<PROJECT_PREFIX>-<phase>/SKILL.md`
格式：YAML frontmatter + Markdown（同 Qoder），支持 `<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法。

### 4.2 Propose Skill 模板

> 以下模板中 `<PLACEHOLDER>` 需根据 Phase 1 分析结果替换。

#### AI 提示词

```
基于以下模板，根据 AI_TOOL 在对应路径创建 Propose Skill 文件：
# AI_TOOL = qoder       → .qoder/skills/<PROJECT_PREFIX>-propose/SKILL.md
# AI_TOOL = claude-code → .claude/commands/<PROJECT_PREFIX>-propose.md
# AI_TOOL = codex       → 嵌入 AGENTS.md 的 ## Propose 工作流 章节
# AI_TOOL = codebuddy   → .codebuddy/skills/<PROJECT_PREFIX>-propose/SKILL.md
将所有 <PLACEHOLDER> 替换为 Phase 1 分析结果中的实际值。

决策点：
- 如果 NEED_NVM_SWITCH = true，保留 "Node.js 版本管理" 节
- 如果 NEED_NVM_SWITCH = false，删除 "Node.js 版本管理" 节
- 完成标志中的 /<PROJECT_PREFIX>-apply 指向正确的下一阶段 Skill 名称
```

> **格式适配说明：**
> <!-- AI_TOOL = qoder / codebuddy --> Qoder 和 CodeBuddy 使用以下模板原样（含 YAML frontmatter、`<HARD-GATE>` 标签、`REQUIRED SUB-SKILL:` 语法）。
> <!-- AI_TOOL = claude-code --> Claude Code：去掉 YAML frontmatter，`<HARD-GATE>` 改为 `> **MUST**` 格式，`REQUIRED SUB-SKILL` 改为内联步骤描述（见模板后替代块）。
> <!-- AI_TOOL = codex --> Codex：去掉 YAML frontmatter，作为 AGENTS.md 的 `##` 章节嵌入，`<HARD-GATE>` 改为 `> **[强约束]**` 格式，`REQUIRED SUB-SKILL` 改为内联步骤描述。

**模板：**

```markdown
---
name: <PROJECT_PREFIX>-propose
description: "<PROJECT_NAME> 需求提案全流程：自动串联 brainstorming → OpenSpec → writing-plans。在项目中提出新功能、变更需求或重构计划时使用。"
---

# <PROJECT_PREFIX> Propose — 需求提案全流程

将需求从模糊想法转化为完整的 OpenSpec 变更提案和实施计划。

**启动声明：** "我正在使用 <PROJECT_PREFIX>-propose Skill 执行需求提案全流程。"

<HARD-GATE>
在用户明确批准设计之前，禁止执行任何代码编写、项目脚手架搭建或其他实现操作。
</HARD-GATE>

## 流程总览

\`\`\`
用户需求 → [Step 1] 读取项目规范 → [Step 2] brainstorming → [Step 3] OpenSpec 变更 → [Step 4] writing-plans → [Step 5] 规范影响评估
\`\`\`

<!-- 如果 NEED_NVM_SWITCH = true，包含此节；否则删除 -->
## Node.js 版本管理

OpenSpec CLI 依赖 Node.js >= <OPENSPEC_NODE_MAJOR>，而项目业务代码可能使用较低版本（如 <NODE_BUILD_VERSION>）。在执行任何 `openspec` 命令前，必须按以下逻辑检查并临时切换版本：

1. 执行 `node --version` 获取当前版本号
2. **如果主版本号 >= <OPENSPEC_NODE_MAJOR>** → 直接执行 openspec 命令，无需切换
3. **如果主版本号 < <OPENSPEC_NODE_MAJOR>** → 记录当前版本，执行 `nvm use <OPENSPEC_NODE_MAJOR>` 切换后再执行 openspec 命令
4. 该步骤中的 openspec 命令全部执行完毕后，如果之前切换过版本，执行 `nvm use <原版本>` 恢复

**注意：** 版本切换仅针对 `openspec` CLI 命令，项目构建（<BUILD_CMD>）仍使用项目要求的 Node.js 版本。

## Step 1: 读取项目规范（自动执行，无需用户参与）

在开始任何对话之前，先读取当前项目的 OpenSpec 规范文件建立上下文：

1. 读取 `openspec/specs/architecture.md` — 理解项目架构和技术栈
2. 读取 `openspec/specs/coding-conventions.md` — 理解编码约束和命名规范
3. 读取 `openspec/specs/business-domain.md`（如存在）— 理解业务领域
4. 检查 `openspec/changes/` 下是否有活跃的变更 — 避免冲突

这些规范将作为后续 brainstorming 和 writing-plans 的**硬约束**。

## Step 2: 需求澄清与设计（调用 brainstorming Skill）

**REQUIRED SUB-SKILL:** 调用 `brainstorming` Skill
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

将 brainstorming 的输出适配到 OpenSpec 流程：

- 探索需求时，结合 Step 1 读取的项目规范作为约束边界
- 提出方案时，考虑与现有架构的一致性
- **设计文档保存位置改为** `openspec/changes/<change-name>/design.md`（而非默认的 docs/superpowers/specs/）
- 设计文档中必须标注引用了哪些 `openspec/specs/` 中的规范条目

**brainstorming 完成后，不直接调用 writing-plans，而是先进入 Step 3。**

## Step 3: 创建 OpenSpec 变更（自动执行）

brainstorming 设计获批后，执行以下 OpenSpec CLI 操作（注意先按"Node.js 版本管理"节检查版本）：

\`\`\`bash
# [版本检查] 若 node 主版本 < <OPENSPEC_NODE_MAJOR>，先执行: nvm use <OPENSPEC_NODE_MAJOR>

# 创建新变更目录
openspec new change "<change-name>" --json

# 查看变更状态和 schema
openspec status --change "<change-name>" --json

# 获取 propose 指引
openspec instructions propose --change "<change-name>" --json

# [版本恢复] 若之前切换过版本，执行: nvm use <原版本>
\`\`\`

根据 CLI 返回的指引，将 Step 2 的设计文档写入变更目录的对应文件中：

- `proposal.md` — 需求概述（从 brainstorming 提炼）
- `specs/` — 技术规格（从 design.md 展开）
- `design.md` — 设计方案（Step 2 的输出）

**等待用户确认变更提案后，进入 Step 4。**

## Step 4: 生成实施计划（调用 writing-plans Skill）

**REQUIRED SUB-SKILL:** 调用 `writing-plans` Skill
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

将 writing-plans 的输出适配到 OpenSpec 流程：

- 计划文档保存位置改为 `openspec/changes/<change-name>/tasks.md`
- 每个任务必须引用 `openspec/specs/coding-conventions.md` 中的相关约束
- 任务粒度遵循 OpenSpec 的 tasks 格式（使用 `- [ ]` 复选框语法）
- 实施计划中的文件路径必须基于当前项目的目录结构

## Step 5: 全局规范影响评估（自动执行）

基于已生成的 proposal.md、design.md、tasks.md，评估本次变更对全局规范文件的影响，生成影响评估文件。

### 5.1 评估逻辑

逐一评估以下 3 个全局规范文件是否受本次变更影响：

| 规范文件 | 影响判断依据 |
|---|---|
| `architecture.md` | 是否引入新的技术组件、通信模式、目录结构变更、模块依赖变更 |
| `coding-conventions.md` | 是否新增编码约定、修改命名规范、调整分层规则 |
| `business-domain.md` | 是否新增业务概念、修改业务流程、调整业务术语 |

### 5.2 生成 specs-impact.md

在变更目录下创建 `openspec/changes/<change-name>/specs-impact.md`：

\`\`\`markdown
# 全局规范影响评估

> 归档时将据此同步更新 `openspec/specs/` 下的对应文件。

## architecture.md
- **影响区域：** <受影响的章节或区域名称，如"无影响"则直接写"无影响">
- **变更原因：** <为什么需要更新，来源于哪个设计决策>
- **变更类型：** ADD / UPDATE / REMOVE

## coding-conventions.md
- 无影响

## business-domain.md
- **影响区域：** <受影响的章节或区域名称>
- **变更原因：** <为什么需要更新>
- **变更类型：** ADD / UPDATE / REMOVE
\`\`\`

**关键规则：**
- 无影响的规范文件写"无影响"，不要省略
- 变更类型：`ADD`（新增章节）、`UPDATE`（修改现有章节）、`REMOVE`（删除章节）
- 纯业务功能变更可能 3 个规范均为"无影响"，这是正常的
- 影响评估必须基于 design.md 中的实际设计决策，不要过度推断

## 完成标志

全流程完成后，输出：

\`\`\`
## 提案完成

**变更名称：** <change-name>
**变更路径：** openspec/changes/<change-name>/
**产出文件：**
  - proposal.md — 需求概述
  - design.md — 设计方案
  - tasks.md — 实施计划（N 个任务）
  - specs-impact.md — 全局规范影响评估

**规范影响：**
  - architecture.md — 无影响 / ADD / UPDATE / REMOVE
  - coding-conventions.md — 无影响 / ADD / UPDATE / REMOVE
  - business-domain.md — 无影响 / ADD / UPDATE / REMOVE

**下一步：** 使用 `/<PROJECT_PREFIX>-apply` 开始实施
\`\`\`

## 技术红线（贯穿全流程）

在任何步骤中，以下约束不可违反：

- 方案必须符合 `openspec/specs/architecture.md` 中声明的技术栈
- 方案必须符合 `openspec/specs/coding-conventions.md` 中的命名和分层规范
- 不得引入规范文件中未声明的新依赖
<!-- 如果 MCP_TOOLS 不为空，保留此行 -->
- 数据库操作必须通过 MCP 工具（<MCP_TOOLS>）执行

## 错误处理

| 情况 | 处理 |
|---|---|
| `openspec/` 目录不存在 | 提示用户先运行 `openspec init --tools <AI_TOOL>` |
| brainstorming 阶段用户否决所有方案 | 回到 Step 2 重新探索 |
| `openspec new change` 失败 | 检查 CLI 版本和错误输出，报告给用户 |
| writing-plans 发现设计有缺陷 | 回退到 Step 2 补充设计 |
<!-- 如果 NEED_NVM_SWITCH = true，保留此行 -->
| `nvm use <OPENSPEC_NODE_MAJOR>` 失败 | 提示用户安装 Node.js <OPENSPEC_NODE_MAJOR>：`nvm install <OPENSPEC_NODE_MAJOR>` |
```

### 4.3 Apply Skill 模板

#### AI 提示词

```
基于以下模板，根据 AI_TOOL 在对应路径创建 Apply Skill 文件：
# AI_TOOL = qoder       → .qoder/skills/<PROJECT_PREFIX>-apply/SKILL.md
# AI_TOOL = claude-code → .claude/commands/<PROJECT_PREFIX>-apply.md
# AI_TOOL = codex       → 嵌入 AGENTS.md 的 ## Apply 工作流 章节
# AI_TOOL = codebuddy   → .codebuddy/skills/<PROJECT_PREFIX>-apply/SKILL.md
决策点：
- 如果 NEED_NVM_SWITCH = true，保留 "Node.js 版本管理" 节
- 如果 PROJECT_TYPE = frontend/mobile，TDD 检测条件为 jest.config/vitest.config/package.json test 脚本
- 如果 PROJECT_TYPE = backend(Java)，TDD 检测条件为 pom.xml 中的 spring-boot-starter-test / spring-boot-test / JUnit
- Step 3 编译验证命令使用 BUILD_CMD
- 完成报告中的下一步指向 /<PROJECT_PREFIX>-archive
```

> **格式适配说明：**
> <!-- AI_TOOL = qoder / codebuddy --> Qoder 和 CodeBuddy 使用以下模板原样。
> <!-- AI_TOOL = claude-code --> Claude Code：去掉 YAML frontmatter，`REQUIRED SUB-SKILL` 改为内联步骤描述。
> <!-- AI_TOOL = codex --> Codex：去掉 YAML frontmatter，作为 AGENTS.md 章节嵌入，`REQUIRED SUB-SKILL` 改为内联步骤描述。

**模板：**

```markdown
---
name: <PROJECT_PREFIX>-apply
description: "项目代码实施全流程：自动串联 OpenSpec apply → TDD → debugging → parallel-agents。在项目中实现已批准的变更提案时使用。"
---

# <PROJECT_PREFIX> Apply — 代码实施全流程

将 OpenSpec 变更提案中的任务逐个实现，全程遵循项目编码规范和质量要求。

**启动声明：** "我正在使用 <PROJECT_PREFIX>-apply Skill 执行代码实施全流程。"

## 流程总览

\`\`\`
OpenSpec 变更 → [Step 1] 加载上下文 → [Step 2] 执行任务循环 → [Step 3] 完成验证与报告
\`\`\`

<!-- 如果 NEED_NVM_SWITCH = true，包含此节 -->
## Node.js 版本管理
（同 Propose 模板的 Node.js 版本管理节）

## Step 1: 加载项目上下文（自动执行）

### 1.1 读取项目规范
1. 读取 `openspec/specs/architecture.md` — 架构约束
2. 读取 `openspec/specs/coding-conventions.md` — 编码规范
3. 读取 `openspec/specs/business-domain.md`（如存在）— 业务规则

### 1.2 加载变更上下文
\`\`\`bash
# [版本检查] 若 node 主版本 < <OPENSPEC_NODE_MAJOR>，先执行: nvm use <OPENSPEC_NODE_MAJOR>

openspec list --json
openspec status --change "<change-name>" --json
openspec instructions apply --change "<change-name>" --json

# [版本恢复] 若之前切换过版本，执行: nvm use <原版本>
\`\`\`

### 1.3 读取上下文文件
读取 `instructions apply` 返回的 `contextFiles` 中的所有文件。

### 1.4 展示当前进度
\`\`\`
## 实施中: <change-name>
**Schema:** <schema-name>
**进度:** N/M 任务完成
**剩余任务:**
  - [ ] 任务 1: ...
  - [ ] 任务 2: ...
\`\`\`

## Step 2: 执行任务循环

### 2.1 任务评估
- 判断是否需要**并行处理** → 调用 `dispatching-parallel-agents`
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

### 2.2 实现代码

**REQUIRED SUB-SKILL:** 调用 `test-driven-development` Skill（如项目已配置测试框架）
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

首先检测项目是否已配置测试框架（检查 `jest.config.*`、`vitest.config.*` 或 `package.json` 中的 `test` 脚本）：

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

<!-- PROJECT_TYPE = backend(Java) -->
首先检测项目是否已配置单元测试（检查 `pom.xml` 中的 `spring-boot-starter-test`、`spring-boot-test` 或 JUnit 依赖）：

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

<!-- PROJECT_STRUCTURE = monorepo -->
首先根据任务涉及的目录判断当前任务属于前端还是后端：
- **任务涉及 `<FRONTEND_DIR>/`** → 检测前端测试框架（`jest.config.*`、`vitest.config.*` 或 `package.json` test 脚本）
  - 已配置 → TDD 红绿循环（4 步，同前端单栈）
  - 未配置 → 编译驱动（`<FE_BUILD_CMD>` + TS 类型检查）
- **任务涉及 `<BACKEND_DIR>/`** → 检测后端测试框架（`pom.xml` 中 `spring-boot-starter-test` / `spring-boot-test` / JUnit 依赖）
  - 已配置 → TDD 红绿循环（`mvn test`），Controller 层使用 `@SpringBootTest` 编写接口测试
  - 未配置 → 编译驱动（`<BE_BUILD_CMD>` + 接口验证）

**编码过程中必须遵循 `openspec/specs/coding-conventions.md` 的全部约束。**

### 2.3 Bug 处理
遇到 Bug 时调用 `systematic-debugging` Skill，禁止猜测性修复。
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

### 2.4 标记完成
在 tasks 文件中将 `- [ ]` 改为 `- [x]`。

### 2.5 任务间检查点
每完成 3 个任务后暂停，展示进度摘要，等待用户确认后继续。

## Step 3: 完成验证与报告

所有任务完成后，先执行一次编译验证：

<!-- PROJECT_STRUCTURE = single -->
\`\`\`bash
<BUILD_CMD>
\`\`\`

<!-- PROJECT_STRUCTURE = monorepo -->
\`\`\`bash
# 前端编译
<FE_BUILD_CMD>

# 后端编译
<BE_BUILD_CMD>
\`\`\`
**编译通过后**，输出完成报告，提示使用 `/<PROJECT_PREFIX>-archive` 进行归档。

## 暂停条件
- 任务描述不清楚
- 发现设计缺陷 → 建议回退到 `/<PROJECT_PREFIX>-propose`
- 遇到阻塞性错误
- 用户中断

## 技术红线（贯穿全流程）
- 代码必须符合 `openspec/specs/coding-conventions.md`
- 已配置测试框架时，禁止跳过测试步骤
- 禁止猜测性 Bug 修复
<!-- 如果 MCP_TOOLS 不为空 -->
- 数据库操作必须通过 MCP 工具执行
```

### 4.4 Archive Skill 模板

#### AI 提示词

```
基于以下模板，根据 AI_TOOL 在对应路径创建 Archive Skill 文件：
# AI_TOOL = qoder       → .qoder/skills/<PROJECT_PREFIX>-archive/SKILL.md
# AI_TOOL = claude-code → .claude/commands/<PROJECT_PREFIX>-archive.md
# AI_TOOL = codex       → 嵌入 AGENTS.md 的 ## Archive 工作流 章节
# AI_TOOL = codebuddy   → .codebuddy/skills/<PROJECT_PREFIX>-archive/SKILL.md
决策点：
- Step 1.1 编译验证命令：PROJECT_TYPE=frontend → BUILD_CMD + LINT_CMD；PROJECT_TYPE=backend → BUILD_CMD
- Step 1.2 测试命令：HAS_TEST_FRAMEWORK → TEST_CMD + "如无测试脚本则跳过"；否则省略
- Step 1.3 规范合规检查：前端写"页面模块结构、DVA Model 命名、路由注册"；后端写"Controller 命名、分层结构、实体类注解"
- NEED_NVM_SWITCH 决定是否包含 Node.js 版本管理节
- 完成报告中的下一步指向正确的 Skill 名称
```

> **格式适配说明：**
> <!-- AI_TOOL = qoder / codebuddy --> Qoder 和 CodeBuddy 使用以下模板原样。`<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法均可直接使用。
> <!-- AI_TOOL = claude-code --> Claude Code：去掉 YAML frontmatter，`<HARD-GATE>` 改为 `> **MUST**`，`REQUIRED SUB-SKILL` 改为内联步骤描述。
> <!-- AI_TOOL = codex --> Codex：去掉 YAML frontmatter，作为 AGENTS.md 章节嵌入，`<HARD-GATE>` 改为 `> **[强约束]**`，`REQUIRED SUB-SKILL` 改为内联步骤描述。

**模板：**

```markdown
---
name: <PROJECT_PREFIX>-archive
description: "项目归档收尾全流程：自动串联 verification → code-review → OpenSpec archive。在项目中所有任务实现完成后进行验证和归档时使用。"
---

# <PROJECT_PREFIX> Archive — 归档收尾全流程

对已完成的 OpenSpec 变更进行验证、代码审查和归档，确保交付质量。

**启动声明：** "我正在使用 <PROJECT_PREFIX>-archive Skill 执行归档收尾全流程。"

<HARD-GATE>
禁止在未运行验证命令并确认输出的情况下声称工作已完成。证据先于结论。
</HARD-GATE>

## 流程总览

\`\`\`
实施完成 → [Step 1] 验证 → [Step 2] 代码审查 → [Step 3] 规范同步 → [Step 4] 归档
\`\`\`

<!-- 如果 NEED_NVM_SWITCH = true，包含此节 -->
## Node.js 版本管理
（同 Propose 模板）

## Step 1: 全面验证

**REQUIRED SUB-SKILL:** 调用 `verification-before-completion` Skill
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

### 1.1 编译与 Lint 验证

<!-- PROJECT_STRUCTURE = single -->
\`\`\`bash
<BUILD_CMD>
<!-- 如果 LINT_CMD 不为空 -->
<LINT_CMD>
\`\`\`
**判定标准：** 编译退出码为 0<!-- 如果 LINT_CMD 不为空 -->，Lint 0 error（warning 可接受）。

<!-- PROJECT_STRUCTURE = monorepo -->
\`\`\`bash
# 前端
<FE_BUILD_CMD>
<FE_LINT_CMD>

# 后端
<BE_BUILD_CMD>
\`\`\`
**判定标准：** 前后端编译均退出码为 0，前端 Lint 0 error。

### 1.2 测试验证
<!-- PROJECT_STRUCTURE = single -->
<!-- 如果 HAS_TEST_FRAMEWORK = true -->
\`\`\`bash
<TEST_CMD>
\`\`\`
**判定标准：** 所有测试通过，0 失败（如项目无测试脚本则跳过）。
<!-- 如果 HAS_TEST_FRAMEWORK = false，省略此节 -->

<!-- PROJECT_STRUCTURE = monorepo -->
\`\`\`bash
# 前端测试（如已配置）
<FE_TEST_CMD>

# 后端测试（如已配置）
<BE_TEST_CMD>
\`\`\`
**判定标准：** 已配置的测试全部通过（未配置则跳过对应部分）。

### 1.3 规范合规检查
- 检查所有新增/修改的文件是否符合 `openspec/specs/coding-conventions.md`
<!-- PROJECT_TYPE = frontend/mobile -->
- 确认页面模块结构、DVA Model 命名、路由注册等符合规范
- 确认无违反技术红线的代码（如未声明的新依赖、非兼容语法等）
<!-- PROJECT_TYPE = backend -->
- 确认 Controller 命名、分层结构、实体类注解等符合规范
- 确认无违反技术红线的代码（如 Java 11+ 特性、未声明的新依赖等）

### 1.4 变更完整性
- 读取 `openspec/changes/<change-name>/tasks.md`
- 逐行检查每个任务是否标记为 `- [x]`
- 确认无遗漏任务

**验证失败时：** 停止流程，建议使用 `/<PROJECT_PREFIX>-apply` 继续修复。

## Step 2: 代码审查

**REQUIRED SUB-SKILL:** 调用 `requesting-code-review` Skill
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

### 2.1 准备审查上下文

\`\`\`bash
git log --oneline --since="<变更开始时间>"
git diff <base-sha>..<head-sha> --stat
\`\`\`

### 2.2 执行审查

调用 code-reviewer 子代理进行审查，提供：

- **审查范围：** 本次变更涉及的所有提交
- **审查标准：** `openspec/specs/coding-conventions.md` + `openspec/specs/architecture.md`
- **业务上下文：** `openspec/specs/business-domain.md`（如存在）

### 2.3 处理审查反馈

- **Critical** → 立即修复，回到 Step 1 重新验证
- **Important** → 修复后继续
- **Minor** → 记录，不阻塞归档
- **审查者判断错误** → 用技术理由反驳

**REQUIRED SUB-SKILL:** 如有审查反馈需处理，调用 `receiving-code-review` Skill
<!-- AI_TOOL = claude-code / codex: 改为内联步骤描述 -->

## Step 3: 全局规范同步（自动执行）

### 3.1 读取影响评估
读取 `openspec/changes/<change-name>/specs-impact.md`：
- **文件不存在** → 跳过本步骤（兼容旧变更）
- **所有规范均为"无影响"** → 跳过本步骤

### 3.2 执行规范同步
对于每个标记了影响的规范文件：
1. 读取当前 `openspec/specs/<file>.md`
2. 读取变更的 `proposal.md` + `design.md` + `tasks.md`
3. 根据 specs-impact.md 更新对应规范文件（ADD/UPDATE/REMOVE）
4. 保持原有格式风格和章节结构

**关键约束：**
- 只更新 specs-impact.md 中标记了影响的规范文件
- 如果更新会破坏现有规范结构，暂停并提示用户确认

### 3.3 同步输出

\`\`\`
## 规范同步完成
- architecture.md — 无影响 / 已更新（ADD 1 节, UPDATE 0 节, REMOVE 0 节）
- coding-conventions.md — 无影响
- business-domain.md — 已更新（ADD 0 节, UPDATE 1 节, REMOVE 0 节）
\`\`\`

## Step 4: 归档变更（自动执行）

验证、审查和规范同步均通过后：

\`\`\`bash
# [版本检查] 若 node 主版本 < <OPENSPEC_NODE_MAJOR>，先执行: nvm use <OPENSPEC_NODE_MAJOR>

# 执行归档（使用 --skip-specs 跳过 OpenSpec 原生的 spec 更新，因为已在 Step 3 自行处理）
openspec archive --change "<change-name>" --skip-specs --json

# [版本恢复] 若之前切换过版本，执行: nvm use <原版本>
\`\`\`

归档完成后输出：

\`\`\`
## 归档完成

**变更名称：** <change-name>
**归档路径：** openspec/changes/archive/<timestamp>-<change-name>/

### 规范同步
- architecture.md — 无影响 / 已更新（ADD 1 节, UPDATE 0 节）
- coding-conventions.md — 无影响
- business-domain.md — 已更新（ADD 1 节）

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
\`\`\`

## 暂停条件
- Step 1 验证失败 → 停止，报告失败详情
- Step 2 发现 Critical/Important 问题 → 停止，等待修复
- 用户中断 → 停止

## 技术红线（贯穿全流程）
- 验证必须基于实际命令输出，禁止"应该能通过"
- 代码审查必须基于项目规范文件，而非通用最佳实践
- 归档前必须所有任务标记为完成
```

---

## 5. Phase 4: 创建 AGENTS.md

### AI 提示词

```
基于以下模板和 Phase 1 分析结果，创建项目根目录的 AGENTS.md 文件。

填充指引：
- 项目上下文表：填入实际规范文件路径和内容描述
- 快速引用：填入技术栈摘要、构建命令、路径别名、HTTP 方案、路由方案
- 如果 PROJECT_STRUCTURE = monorepo，快速参考和日常编码规则需分为前端/后端两区域
- propose 描述：包含 5 步（读规范 → brainstorming → OpenSpec → writing-plans → 规范影响评估）
- propose 下增加 "design.md 必须包含" 指引（路由设计、状态管理定义、API 映射等，基于 PROJECT_TYPE）
- apply 描述：TDD 条件化表述（"如已配置测试框架/单元测试"或"编译驱动开发"）
- apply 下增加编码约束列表（6-8 条具体约束，引用 coding-conventions.md 的关键条目）
- archive 描述：包含验证 → 审查 → 规范同步 → 归档（--skip-specs）
- 日常编码规则：按 PROJECT_TYPE 填入对应分类（前端：新增页面/修改页面/样式调整/表单开发；后端：修复 Bug/新增 API/修改数据库）
- 技术红线：从 Phase 1 分析结果的 TECH_CONSTRAINTS 填入
```

**模板：**

```markdown
# <PROJECT_NAME> — AI 编码集成规则

> 本文件是 OpenSpec + Superpowers 的连接层，定义 AI 在本项目中编码时必须遵循的工作流和行为规范。

## 项目上下文

- 规范优先级：**团队规范基准文件（codebook.md / architecturebook.md / businessbook.md）> openspec/specs/ 中的项目级规范 > 代码推断**

在编写任何代码前，必须先阅读以下规范文件获取项目知识：

| 规范 | 路径 | 内容 |
|---|---|---|
| 架构规范 | `openspec/specs/architecture.md` | <架构规范内容摘要> |
| 编码规范 | `openspec/specs/coding-conventions.md` | <编码规范内容摘要> |
| 业务域规范 | `openspec/specs/business-domain.md` | <业务域规范内容摘要> |

### 快速参考
- **技术栈**: <TECH_STACK>
- **构建**: <BUILD_CMD>
- **路径别名**: <PATH_ALIAS>
- **HTTP**: <HTTP 通信方案>
- **路由**: <路由方案>

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

> 以下三个阶段各对应一个项目级 Skill，执行时自动串联多个 Superpowers Skill 和 OpenSpec CLI。

### 阶段一：需求提案 → 调用 `<PROJECT_PREFIX>-propose` Skill

当用户提出新功能或变更需求时，**直接调用 `<PROJECT_PREFIX>-propose` Skill**。

该 Skill 自动执行：读取项目规范 → brainstorming → OpenSpec 变更创建 → writing-plans → 规范影响评估（specs-impact.md）。

**design.md 必须包含**：
<!-- 根据 PROJECT_TYPE 填入项目特定的 design.md 要求 -->
<!-- PROJECT_TYPE = frontend/mobile -->
- 路由设计（菜单入口 / 功能页面的路径规划）
- DVA Model / 状态管理的 namespace 和 effects 定义
- 与后端 Controller 的接口映射
<!-- PROJECT_TYPE = backend -->
- Controller 命名和 URL 路径设计
- Service/Mapper/Entity 分层规划
- 数据库表结构变更（如适用）
- API 接口入参/出参定义

### 阶段二：代码实现 → 调用 `<PROJECT_PREFIX>-apply` Skill

当开始实现已批准的变更时，**直接调用 `<PROJECT_PREFIX>-apply` Skill**。

该 Skill 自动执行：加载变更上下文 → TDD（如已配置测试框架）或编译驱动开发 → Bug 系统调试 → 并行任务分发。

代码必须符合 `openspec/specs/coding-conventions.md` 的约束：
<!-- 根据 PROJECT_TYPE 填入 6-8 条具体编码约束，引用 coding-conventions.md 的关键条目 -->
<!-- PROJECT_TYPE = frontend/mobile -->
- 页面模块结构: `index.tsx` + `index.less` + `model.ts` + `service.ts`
- 路由注册: 新增路由在路由配置文件中注册
- 路径别名: 使用项目约定的路径别名导入
- API 调用: 通过项目 HTTP 工具封装调用
- 样式: 使用项目预处理器和主题变量
- 新依赖: 禁止引入规范文件未声明的依赖
<!-- PROJECT_TYPE = backend -->
- 分层结构: controller → service → mapper → entity
- Controller 命名: 按项目命名约定（如 Diy*/Inf*/Itf*）
- ORM: 使用项目 ORM 框架，禁止手写 SQL（复杂查询除外）
- 实体类: 使用 Lombok 注解
- 新依赖: 禁止引入 pom.xml 未声明的依赖
- 数据库操作: 通过 MCP 工具执行

### 阶段三：归档收尾 → 调用 `<PROJECT_PREFIX>-archive` Skill

当所有任务实现完成后，**直接调用 `<PROJECT_PREFIX>-archive` Skill**。

该 Skill 自动执行：验证（编译/Lint/测试/规范检查） → 代码审查 → 全局规范同步 → OpenSpec 归档（--skip-specs）。

## 日常编码行为规则

<!-- 根据 PROJECT_TYPE 选择对应的规则分类 -->
<!-- PROJECT_TYPE = frontend/mobile -->
### 新增页面
1. 在源代码目录下创建对应模块目录
2. 创建标准文件结构（如 index.tsx + model.ts + service.ts）
3. 在路由配置文件中注册路由
4. 如需全局数据，在全局服务中配置

### 修改现有页面
1. 先阅读 model.ts / 状态管理理解数据流
2. 先阅读 service.ts 理解 API 调用链
3. 修改后确保路由参数兼容

### 样式调整
1. 优先使用 UI 组件库内置样式
2. 自定义样式写在对应样式文件中
3. 全局主题色通过主题配置文件修改

### 表单开发
1. 使用项目 UI 组件库的 Form 组件
2. 下拉数据源通过全局服务获取
3. 数据转换使用项目工具方法
<!-- PROJECT_TYPE = backend -->
### 修复 Bug
1. 调用 `systematic-debugging` Skill
2. 检查 `openspec/specs/business-domain.md` 确认业务逻辑正确性
3. 修复后调用 `verification-before-completion` Skill 验证

### 新增 API 接口
1. 确认目标模块（多模块项目需确认子模块）
2. 按编码规范的命名约定创建 Controller
3. 创建对应的 Service + Mapper + Entity/DTO/VO
4. 编写单元测试

### 修改数据库
1. 通过 MCP 工具操作数据库
2. 禁止绕过 MCP 使用脚本方式查询数据
3. DDL 变更需同步更新对应的 Mapper XML
<!-- 通用 -->
### 代码审查反馈
- 调用 `receiving-code-review` Skill 处理审查意见
- 对每条反馈验证技术正确性后再修改

## 技术红线

<!-- 从 Phase 1 分析结果的 TECH_CONSTRAINTS 填入 -->
<版本锁定>
<禁止引入的依赖>
<禁止修改的配置>
<其他硬性约束>
```

### 5.2 工具专属指令文件

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
- 规范优先级：团队规范基准文件 > openspec/specs/ > 代码推断

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
- 规范优先级：团队规范基准文件 > openspec/specs/ > 代码推断

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

## 6. Phase 5: 创建操作指南

### AI 提示词

```
基于以下模板和 Phase 1 分析结果，创建项目根目录的 OPENSPEC_SUPERPOWERS_GUIDE.md 文件。
此文件面向新加入的团队成员，需要清晰易懂。

填充指引：
- 2.1 环境要求：如果 PROJECT_TYPE=frontend/mobile 且 NEED_NVM_SWITCH=true，Node.js 说明写为
  "OpenSpec CLI 依赖 <OPENSPEC_NODE_MAJOR>+；系统环境默认配置前端构建 Node.js 版本，Skill 执行 openspec 时临时切换到 <OPENSPEC_NODE_MAJOR>"
- 如果 PROJECT_STRUCTURE = monorepo，环境要求同时包含前后端工具（Node.js + Java/Maven）
- 2.3 初始化步骤：包含 openspec init --tools <AI_TOOL> 步骤
- 3.2 Skill 说明：propose 5 步、apply TDD 条件化、archive 4 步
- 4.2 操作示例：apply 阶段写 "TDD（如已配置测试框架）或编译驱动方式"
- 5.1 上手清单：包含 openspec init 步骤
- 5.3 FAQ：包含 "openspec-* Skill 不存在" 的排错条目
```

**模板骨架：**

```markdown
# OpenSpec + Superpowers 使用指南

本指南面向新加入的团队成员，介绍如何在 <PROJECT_NAME> 项目中使用 OpenSpec 与 Superpowers 辅助开发。

## 1. 集成概述
### 1.1 什么是 OpenSpec
### 1.2 什么是 Superpowers
### 1.3 集成方式

## 2. 初始化配置
### 2.1 环境要求
| Node.js | >= <OPENSPEC_NODE_MAJOR> | <根据 NEED_NVM_SWITCH 填写说明> |
<!-- 如果 PROJECT_TYPE=backend，增加 Java/Maven 版本要求 -->

### 2.2 安装 OpenSpec CLI
### 2.3 克隆项目后的初始化步骤
1. cd <PROJECT_NAME>
2. <npm install（如前端）/ 无（如后端）>
3. openspec init --tools <AI_TOOL>
4. 确认工具配置目录下包含自定义 Skill（路径根据 AI_TOOL 而定）
5. 确认 openspec/specs/ 下存在规范文件
6. 在 AI 编码工具中以项目为工作区打开
7. 加载/刷新 Skill（方式根据 AI 工具而定）

### 2.4 Superpowers 安装（全局一次性）

## 3. 使用规范
### 3.1 自定义 Skill 列表
### 3.2 每个 Skill 的详细说明
#### <PROJECT_PREFIX>-propose（需求提案）— 5 步
#### <PROJECT_PREFIX>-apply（代码实施）— TDD 条件化
#### <PROJECT_PREFIX>-archive（归档收尾）— 4 步

## 4. 工作流程
### 4.1 标准开发流程
### 4.2 各阶段操作示例（apply 阶段写 TDD 条件化）
### 4.3 AGENTS.md 配置说明

## 5. 团队协作
### 5.1 新成员上手清单（包含 openspec init 步骤）
### 5.2 提交规范
### 5.3 常见问题（包含 openspec-* Skill 不存在 FAQ）
### 5.4 获取帮助
```

---

## 7. Phase 6: Git 配置

### AI 提示词

```
根据 AI_TOOL 在项目根目录的 .gitignore 中添加对应规则（如不存在则创建）：

<!-- AI_TOOL = qoder -->
# OpenSpec auto-generated（不提交）
.qoder/commands/
.qoder/skills/openspec-*/

# OpenSpec committed（提交）
# .qoder/skills/<PROJECT_PREFIX>-*/  ← 需要提交
# openspec/specs/             ← 需要提交
# openspec/changes/            ← 需要提交

<!-- AI_TOOL = claude-code -->
# OpenSpec auto-generated（不提交）
.claude/commands/openspec-*.md

# OpenSpec committed（提交）
# .claude/commands/<PROJECT_PREFIX>-*.md  ← 需要提交
# openspec/specs/                 ← 需要提交
# openspec/changes/                ← 需要提交

<!-- AI_TOOL = codex -->
# Codex 不生成独立配置文件，AGENTS.md 中的 openspec 章节为手动维护
# openspec/specs/   ← 需要提交
# openspec/changes/  ← 需要提交

<!-- AI_TOOL = codebuddy -->
# OpenSpec auto-generated（不提交）
.codebuddy/commands/
.codebuddy/skills/openspec-*/

# OpenSpec committed（提交）
# .codebuddy/skills/<PROJECT_PREFIX>-*/  ← 需要提交
# openspec/specs/                ← 需要提交
# openspec/changes/               ← 需要提交
```

**规则说明（根据 AI_TOOL 选择对应表格）：**

<!-- AI_TOOL = qoder -->
| 路径 | 是否提交 | 原因 |
|---|---|---|
| `.qoder/skills/<PROJECT_PREFIX>-*/` | 提交 | 自定义 Skill，团队共享 |
| `.qoder/skills/openspec-*/` | 不提交 | OpenSpec 自动生成，clone 后 `openspec init` 重新生成 |
| `.qoder/commands/` | 不提交 | OpenSpec 自动生成的斜杠命令 |
| `openspec/specs/` | 提交 | 项目规范，团队共享 |
| `openspec/changes/` | 提交 | 变更历史，可追溯 |
| `AGENTS.md` | 提交 | 项目级 AI 规则 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 提交 | 团队指南 |

<!-- AI_TOOL = claude-code -->
| 路径 | 是否提交 | 原因 |
|---|---|---|
| `.claude/commands/<PROJECT_PREFIX>-*.md` | 提交 | 自定义命令，团队共享 |
| `.claude/commands/openspec-*.md` | 不提交 | 自动生成 |
| `openspec/specs/` | 提交 | 项目规范 |
| `openspec/changes/` | 提交 | 变更历史 |
| `AGENTS.md` | 提交 | 项目级 AI 规则 |
| `CLAUDE.md` | 提交 | Claude Code 专属指令 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 提交 | 团队指南 |

<!-- AI_TOOL = codex -->
| 路径 | 是否提交 | 原因 |
|---|---|---|
| `AGENTS.md` | 提交 | 项目级 AI 规则（含三个工作流章节） |
| `openspec/specs/` | 提交 | 项目规范 |
| `openspec/changes/` | 提交 | 变更历史 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 提交 | 团队指南 |

<!-- AI_TOOL = codebuddy -->
| 路径 | 是否提交 | 原因 |
|---|---|---|
| `.codebuddy/skills/<PROJECT_PREFIX>-*/` | 提交 | 自定义 Skill，团队共享 |
| `.codebuddy/skills/openspec-*/` | 不提交 | 自动生成 |
| `.codebuddy/commands/` | 不提交 | 自动生成的斜杠命令 |
| `openspec/specs/` | 提交 | 项目规范 |
| `openspec/changes/` | 提交 | 变更历史 |
| `AGENTS.md` | 提交 | 项目级 AI 规则 |
| `CODEBUDDY.md` | 提交 | CodeBuddy 专属指令 |
| `OPENSPEC_SUPERPOWERS_GUIDE.md` | 提交 | 团队指南 |

---

## 8. 项目类型适配速查

### 8.1 前端项目（React/Vue）

| 配置项 | 值 |
|---|---|
| BUILD_CMD | `npm run build` |
| LINT_CMD | `npx eslint <src>/ --ext .ts,.tsx` |
| TEST_CMD | `npm test` |
| NEED_NVM_SWITCH | 通常 true（项目用 16/18，OpenSpec 用 <OPENSPEC_NODE_MAJOR>） |
| TDD 检测 | jest.config / vitest.config / package.json test 脚本 |
| 规范合规检查项 | 页面模块结构、Model 命名、路由注册、路径别名 |
| 技术红线示例 | 框架版本锁定、禁止未声明依赖、禁止修改 Webpack 配置 |

### 8.2 后端项目（Java/Spring Boot）

| 配置项 | 值 |
|---|---|
| BUILD_CMD | `mvn clean compile -pl <module> -am` |
| LINT_CMD | 无（或 checkstyle） |
| TEST_CMD | `mvn test -pl <module>` |
| NEED_NVM_SWITCH | 通常 false（后端不需要 nvm，Node.js <OPENSPEC_NODE_MAJOR> 作为系统默认即可） |
| TDD 检测 | pom.xml 中 spring-boot-starter-test / spring-boot-test / JUnit 依赖 |
| 规范合规检查项 | Controller 命名、分层结构、实体类注解 |
| 技术红线示例 | Java 版本锁定、禁止未声明依赖、禁止修改 parent POM |

### 8.3 移动端项目（React Native/H5）

| 配置项 | 值 |
|---|---|
| BUILD_CMD | `npm run build` |
| LINT_CMD | `npx eslint <src>/ --ext .ts,.tsx` |
| TEST_CMD | `npm test` |
| NEED_NVM_SWITCH | 通常 true |
| TDD 检测 | 同前端 |
| 规范合规检查项 | 页面结构、路由（根路径如 /mobile）、移动端交互规则 |
| 技术红线示例 | 框架版本锁定、禁止 hover 效果、禁止绝对路径跳转 |

### 8.4 Monorepo 项目（前后端同仓库）

| 配置项 | 值 |
|---|---|
| PROJECT_STRUCTURE | `monorepo` |
| 需要的额外变量 | `FRONTEND_DIR`、`BACKEND_DIR`、`FE_BUILD_CMD`、`BE_BUILD_CMD` 等（见 Phase 1 变量表） |
| 构建验证 | 前端 `<FE_BUILD_CMD>` + 后端 `<BE_BUILD_CMD>` 分别执行 |
| 测试验证 | 前端 `<FE_TEST_CMD>` + 后端 `<BE_TEST_CMD>` 分别执行 |
| TDD 检测 | 根据任务涉及目录判断前端/后端，分别检测对应测试框架 |
| 规范文件 | architecture.md 和 coding-conventions.md 分前端/后端两区域；business-domain.md 共享 |
| AGENTS.md | 日常编码规则包含前端 + 后端两套分类 |
| Guide 环境要求 | 同时包含 Node.js（前端构建+OpenSpec CLI）和 Java/Maven（后端构建） |
| 技术红线示例 | 前端框架版本锁定 + Java 版本锁定 + 禁止跨层引用（前端直接访问后端 DAO 等） |

---

## 9. 集成验证清单

集成完成后，逐项检查（通过 ✅ 标记，未通过 ❌ 标记）：

```
[ ] Step 0: AI_TOOL 已检测并确认（codebuddy / claude-code / codex / qoder）
[ ] Step 1: 项目分析完成，变量表已确认
[ ] Step 2-A: openspec init --tools <AI_TOOL> 执行成功（工具配置目录和 openspec/ 存在）
[ ] Step 2-B: openspec/specs/ 下有 architecture.md、coding-conventions.md、business-domain.md
[ ] Step 2-C: 已有规范文件未被修改
[ ] Step 2-D: 缺失的规范文件已生成（基于团队基准文件 + 项目分析）
[ ] Step 2-E: 生成规范时已读取对应基准文件（codebook/architecturebook/businessbook）
[ ] Step 2-F: AGENTS.md 中规范优先级包含团队基准文件
[ ] Step 3-A: Propose Skill 已创建（路径匹配 AI_TOOL）且流程含 5 步
[ ] Step 3-B: Apply Skill 已创建（路径匹配 AI_TOOL）且 TDD 已条件化
[ ] Step 3-C: Archive Skill 已创建（路径匹配 AI_TOOL）且含 4 步（验证→审查→规范同步→归档）
[ ] Step 4-A: 项目级指令文件已生成（匹配 AI_TOOL）
[ ] Step 4-B: AGENTS.md 的 propose 描述包含"规范影响评估"
[ ] Step 4-C: AGENTS.md 的 apply 描述包含 TDD 条件化
[ ] Step 4-D: AGENTS.md 的 archive 描述包含"规范同步"和"--skip-specs"
[ ] Step 4-E: AGENTS.md 的 propose 下包含"design.md 必须包含"指引
[ ] Step 4-F: AGENTS.md 的 apply 下包含 6-8 条具体编码约束列表
[ ] Step 4-G: AGENTS.md 包含日常编码行为规则
[ ] Step 5: OPENSPEC_SUPERPOWERS_GUIDE.md 存在
[ ] Step 5 补充: Guide 2.3 包含 openspec init --tools <AI_TOOL> 步骤
[ ] Step 5 补充: Guide 5.1 上手清单包含 openspec init 步骤
[ ] Step 5 补充: Guide 4.2 apply 示例反映 TDD 条件化
[ ] Step 5 补充: Guide 5.3 FAQ 包含 openspec-* Skill 排错条目
[ ] Step 6: .gitignore 包含对应工具的自动生成忽略规则
[ ] 如果 NEED_NVM_SWITCH=true，3 个 Skill 均包含 Node.js 版本管理节
[ ] 如果 NEED_NVM_SWITCH=true，Guide 2.1 的 Node.js 说明为"临时切换"表述
[ ] Propose Skill 的 Step 5 紧接 Step 4，技术红线和错误处理在文件末尾
[ ] Archive Skill 的规范合规检查项与项目类型匹配（前端无 Java 引用）
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
> [ ] Apply Skill Step 2.2 包含按目录判断前端/后端的 TDD 双模式逻辑
> [ ] Archive Skill Step 1.1 包含前端+后端双编译验证
> [ ] architecture.md 和 coding-conventions.md 分前端/后端两区域
> [ ] AGENTS.md 日常编码规则包含前端+后端两套分类
> [ ] Guide 2.1 环境要求同时包含 Node.js 和 Java/Maven
> ```

---

## 10. 执行顺序

```
Phase 0: 检测 AI 工具环境 → 确定 AI_TOOL 变量
    ↓
Phase 1: 项目分析 → 输出变量表
    ↓
Phase 2: openspec init --tools <AI_TOOL> + 创建 3 个规范文件
    ↓
Phase 3: 创建 3 个自定义 Skill（propose → apply → archive）→ 路径和格式匹配 AI_TOOL
    ↓
Phase 4: 创建 AGENTS.md（+ 工具专属指令文件）
    ↓
Phase 5: 创建 OPENSPEC_SUPERPOWERS_GUIDE.md
    ↓
Phase 6: 配置 .gitignore
    ↓
验证清单逐项检查
```

> **提示：** Phase 3-5 中的模板填充可以并行执行，但建议先完成 Phase 3（Skill 创建），因为 AGENTS.md 和 Guide 中的描述需要与 Skill 实际步骤一致。
