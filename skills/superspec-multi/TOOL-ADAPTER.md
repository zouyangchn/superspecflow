# 工具适配参考手册

本文件是 superspec-multi Skill 跨工具适配的速查参考，供 SKILL.md 和 PLAYBOOK.md 在执行时按需读取。

**本期支持工具**: Qoder、Claude Code、Codex、CodeBuddy

---

## 1. 工具检测速查

通过运行时信号（环境变量、进程、工作区目录）判断当前执行环境，而非检查系统中安装了哪些工具。

| 优先级 | 检测信号 | 判定结果 |
|---|---|---|
| 1 | 环境变量 `CODEBUDDY_SESSION_ID` / 进程 `codebuddy*` / 工作区 `.codebuddy/` | `AI_TOOL = codebuddy` |
| 2 | 环境变量 `CLAUDE_CODE_ENTRY` 或 `ANTHROPIC_API_KEY` / 进程 `claude` | `AI_TOOL = claude-code` |
| 3 | 环境变量 `CODEX` / 进程 `codex` | `AI_TOOL = codex` |
| 4（兜底） | 以上均不匹配 | `AI_TOOL = qoder`（默认） |

> **检测原理：** 优先通过运行时信号判断当前 Skill 执行环境，而非检查系统级安装目录。如果自动检测不正确，可设置环境变量 `AI_TOOL` 覆盖。

**检测脚本（PowerShell）：**

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
```

**检测脚本（bash）：**

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
```

---

## 2. 工具配置映射表

| 配置项 | Qoder | Claude Code | Codex | CodeBuddy |
|---|---|---|---|---|
| Skill 存放路径 | `.qoder/skills/<PROJECT_PREFIX>-{phase}/SKILL.md` | `.claude/commands/<PROJECT_PREFIX>-{phase}.md` | `AGENTS.md` 内章节 | `.codebuddy/skills/<PROJECT_PREFIX>-{phase}/SKILL.md` |
| 文件格式 | YAML frontmatter + Markdown | Markdown + 可选 frontmatter | 纯 Markdown（嵌入 AGENTS.md） | YAML frontmatter + Markdown |
| 触发方式 | `/<PROJECT_PREFIX>-{phase}` | `/<PROJECT_PREFIX>-{phase}` | 自然语言指令引用 AGENTS.md 章节 | `/<PROJECT_PREFIX>-{phase}` 或 AI 自动选择 |
| 约束标签 | `<HARD-GATE>...</HARD-GATE>` | CLAUDE.md 中的 `MUST` 指令区 | AGENTS.md 中的强约束章节 | `<HARD-GATE>...</HARD-GATE>`（同 Qoder） |
| 子 Skill 调用 | `REQUIRED SUB-SKILL:` 语法 | 内联步骤描述 | 内联步骤描述 | `REQUIRED SUB-SKILL:` 语法（同 Qoder） |
| OpenSpec init | `openspec init --tools qoder` | `openspec init --tools claude` | `openspec init --tools codex` | `openspec init --tools codebuddy` |
| 项目级指令文件 | `AGENTS.md` | `AGENTS.md` + `CLAUDE.md` | `AGENTS.md` | `AGENTS.md` + `CODEBUDDY.md` |
| init 产出目录 | `.qoder/` + `openspec/` | `.claude/` + `openspec/` | `AGENTS.md` + `openspec/` | `.codebuddy/` + `openspec/` |
| Superpowers 支持 | 完整（原生子代理编排） | 完整（原生子代理编排） | 完整（框架层支持） | 完整（原生子代理编排） |

---

## 3. Skill/命令文件格式示例

### 3.1 Qoder 格式

**路径**: `.qoder/skills/<PROJECT_PREFIX>-propose/SKILL.md`

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
...（后续内容）
```

**要点**:
- YAML frontmatter 包含 `name` 和 `description`
- `<HARD-GATE>` 标签用于强约束声明
- `REQUIRED SUB-SKILL:` 语法用于调用子 Skill
- 通过 `/<PROJECT_PREFIX>-propose` 斜杠命令触发

### 3.2 Claude Code 格式

**路径**: `.claude/commands/<PROJECT_PREFIX>-propose.md`

```markdown
# Propose — 多项目需求提案全流程

**启动声明：** "我正在使用 <PROJECT_PREFIX>-propose 命令执行需求提案全流程。"

> **MUST** 在用户明确批准设计之前，禁止执行任何代码编写或实现操作。

## 流程总览
...（后续内容）

## Step 2: 需求澄清与设计
按以下步骤执行需求澄清：
1. 读取所有子项目规范文件
2. 列出 3-5 个方案选项，评估利弊
3. 推荐最优方案并等待用户确认
4. 设计文档保存到 `openspec/changes/<change-name>/design.md`

...（后续内容）
```

**要点**:
- 无需 YAML frontmatter（可选）
- 约束使用 `> **MUST**` 格式，同时写入 `CLAUDE.md` 的 MUST 指令区
- 子 Skill 调用改为内联步骤描述（Claude Code 无 `REQUIRED SUB-SKILL` 语法）
- 通过 `/<PROJECT_PREFIX>-propose` 斜杠命令触发
- 需额外生成 `CLAUDE.md` 引用 `AGENTS.md`

### 3.3 Codex 格式

**路径**: 嵌入 `AGENTS.md` 的独立章节（无独立文件）

```markdown
# AGENTS.md — <PARENT_DIR_NAME> 多项目统一规则

...（通用规则等章节）

## Propose 工作流 — 多项目需求提案全流程

> **[强约束]** 在用户明确批准设计之前，禁止执行任何代码编写或实现操作。

### 流程总览
...（后续内容）

### Step 2: 需求澄清与设计
按以下步骤执行需求澄清：
1. 读取所有子项目规范文件
2. 列出 3-5 个方案选项，评估利弊
3. 推荐最优方案并等待用户确认
4. 设计文档保存到 `openspec/changes/<change-name>/design.md`

...（后续内容）

## Apply 工作流 — 多项目代码实施全流程
...

## Archive 工作流 — 多项目归档收尾全流程
...
```

**要点**:
- 三个工作流全部嵌入 `AGENTS.md`，作为独立章节
- 约束使用 `> **[强约束]**` 格式
- 子 Skill 调用改为内联步骤描述
- 通过自然语言指令触发（如 "执行 propose 流程"），Codex 自动读取 AGENTS.md 对应章节
- Codex 原生读取 `AGENTS.md`，无需额外配置文件

### 3.4 CodeBuddy 格式

**路径**: `.codebuddy/skills/<PROJECT_PREFIX>-propose/SKILL.md`

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
...（后续内容）
```

**要点**:
- 格式与 Qoder 完全一致（YAML frontmatter + Markdown）
- `<HARD-GATE>` 标签和 `REQUIRED SUB-SKILL:` 语法均可直接使用
- 通过 `/<PROJECT_PREFIX>-propose` 斜杠命令触发，也支持 AI 根据任务上下文自动选择
- 需额外生成 `CODEBUDDY.md` 引用 `AGENTS.md`（类似 Claude Code 的 CLAUDE.md）
- CodeBuddy 还识别 `${CLAUDE_SKILL_DIR}`、`${CLAUDE_PLUGIN_ROOT}` 等别名（与 Claude Code 兼容）

---

## 4. 子 Skill 调用适配

Qoder 和 CodeBuddy 使用 `REQUIRED SUB-SKILL:` 语法调用 Superpowers 子代理（两者格式相同，无需内联替代）。Claude Code 和 Codex 需要将子 Skill 的行为内联为逐步指令。

### 4.1 brainstorming（Propose Step 2）

**Qoder 原生语法:**
```
REQUIRED SUB-SKILL: 调用 `brainstorming` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
按以下步骤执行需求澄清与设计：
1. 读取所有涉及子项目的 openspec/specs/ 规范文件
2. 基于需求和现有规范，列出 3-5 个可行方案
3. 对每个方案评估：技术可行性、对现有规范的影响、实施复杂度
4. 推荐最优方案并说明理由
5. 等待用户确认方案后继续
6. 将确认的方案写入 `openspec/changes/<change-name>/design.md`
```

### 4.2 writing-plans（Propose Step 4）

**Qoder 原生语法:**
```
REQUIRED SUB-SKILL: 调用 `writing-plans` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
按以下步骤生成实施计划：
1. 读取 design.md 中的确认方案
2. 将方案分解为具体任务，每个任务包含：描述、涉及子项目、预估复杂度
3. 按依赖关系排序任务
4. 将计划写入 `openspec/changes/<change-name>/tasks.md`
```

### 4.3 test-driven-development（Apply Step 2.2）

**Qoder 原生语法:**
```
REQUIRED SUB-SKILL: 调用 `test-driven-development` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
按 TDD 红绿循环执行：
1. 阅读当前任务的实现要求
2. 先写失败测试（覆盖核心逻辑）
3. 运行测试确认失败
4. 写最小实现代码
5. 运行测试确认通过
6. 如有需要，重构代码并保持测试通过

未配置测试框架时，按编译驱动开发执行：
1. 先写实现代码
2. 运行 BUILD_CMD 确认编译通过
3. 手动验证核心逻辑
```

### 4.4 systematic-debugging（Apply Step 2.3）

**Qoder 原生语法:**
```
遇到 Bug 时调用 `systematic-debugging` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
遇到 Bug 时按以下步骤系统调试：
1. 复现问题：记录错误信息和复现步骤
2. 定位范围：确定问题所在的子项目和模块
3. 收集证据：查看日志、断点调试、检查数据流
4. 形成假设：基于证据提出可能原因
5. 验证假设：逐一验证，禁止猜测性修复
6. 修复并验证：修复后运行测试确认
```

### 4.5 verification-before-completion（Archive Step 1）

**Qoder 原生语法:**
```
REQUIRED SUB-SKILL: 调用 `verification-before-completion` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
按以下步骤执行全面验证：
1. 对每个涉及的子项目运行 BUILD_CMD，确认编译通过
2. 对已配置测试框架的子项目运行 TEST_CMD，确认测试通过
3. 检查所有新增/修改文件是否符合 coding-conventions.md
4. 读取 tasks.md 确认所有任务标记为 [x]
5. 汇总验证结果，任何失败项停止流程
```

### 4.6 code-review（Archive Step 2）

**Qoder 原生语法:**
```
REQUIRED SUB-SKILL: 调用 `requesting-code-review` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
按以下步骤执行代码审查：
1. 收集本次变更涉及的所有子项目 git diff
2. 审查标准：各子项目 coding-conventions.md + architecture.md
3. 逐文件检查：命名规范、分层结构、技术红线、测试覆盖
4. 问题分级：Critical（阻塞）、Important（需修复）、Minor（记录）
5. 输出审查报告
```

### 4.7 receiving-code-review（Archive Step 2.3）

**Qoder 原生语法:**
```
REQUIRED SUB-SKILL: 如有审查反馈须处理，调用 `receiving-code-review` Skill
```

**Claude Code / Codex 内联替代:**
```markdown
按以下步骤处理审查反馈：
1. 逐条阅读审查意见，理解反馈意图
2. 对每条反馈验证技术正确性（对照规范文件判断）
3. Critical 反馈：立即修复，回到验证步骤重新确认
4. Important 反馈：修复后继续流程
5. Minor 反馈：记录，不阻塞归档
6. 审查者判断错误：用技术理由反驳并记录
```

### 4.8 dispatching-parallel-agents（Apply Step 2.1，条件性）

**Qoder 原生语法:**
```
判断是否需要并行处理 → 调用 `dispatching-parallel-agents`
```

**Claude Code / Codex 内联替代:**
```markdown
判断任务是否可并行执行：
1. 检查任务列表中是否存在相互独立的任务（无依赖关系）
2. 如存在独立任务，按子项目分组，分别执行
3. 各组独立完成后，汇总结果并检查冲突
4. 如任务存在依赖，按顺序逐个执行
```

---

## 5. OpenSpec init 适配

### 5.1 Qoder

```bash
openspec init --tools qoder
```

产出：
- `.qoder/`（skills + commands 目录）
- `openspec/`（changes 目录）

### 5.2 Claude Code

```bash
openspec init --tools claude
```

产出：
- `.claude/`（commands 目录）
- `openspec/`（changes 目录）

> **注意**: 如果 openspec 不支持 `--tools claude` 参数，需手动创建 `.claude/commands/` 目录，并将 Skill 文件写入其中。

### 5.3 Codex

```bash
openspec init --tools codex
```

产出：
- `AGENTS.md`（如不存在则创建）
- `openspec/`（changes 目录）

> **注意**: 如果 openspec 不支持 `--tools codex` 参数，需手动创建 `openspec/changes/` 目录，并将工作流嵌入现有 `AGENTS.md`。

### 5.4 CodeBuddy

```bash
openspec init --tools codebuddy
```

产出：
- `.codebuddy/`（skills + agents 目录）
- `openspec/`（changes 目录）

---

## 6. 项目级指令文件适配

### 6.1 Qoder

只需生成 `AGENTS.md`。Qoder 通过 `AGENTS.md` + `.qoder/skills/` 配合工作。

### 6.2 Claude Code

需要生成两个文件：
- `AGENTS.md` — 项目上下文和编码规则（与 Qoder 版本内容一致）
- `CLAUDE.md` — Claude Code 专属指令文件

**CLAUDE.md 模板：**

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

## 命令

- `/<PROJECT_PREFIX>-propose` — 需求提案全流程
- `/<PROJECT_PREFIX>-apply` — 代码实施全流程
- `/<PROJECT_PREFIX>-archive` — 归档收尾全流程
```

### 6.3 Codex

只需生成 `AGENTS.md`。Codex 原生读取 `AGENTS.md`，三个工作流作为章节嵌入其中。

与 Qoder 版本的区别：
- 无 `<HARD-GATE>` 标签，改用 `> **[强约束]**` 格式
- 无 `REQUIRED SUB-SKILL:` 语法，子 Skill 行为内联为逐步指令
- propose / apply / archive 作为 AGENTS.md 的章节标题，而非独立文件

### 6.4 CodeBuddy

需要生成两个文件：
- `AGENTS.md` — 项目上下文和编码规则（与 Qoder 版本内容一致）
- `CODEBUDDY.md` — CodeBuddy 专属指令文件（CodeBuddy 在每次会话开始时读取）

**CODEBUDDY.md 模板：**

```markdown
# CODEBUDDY.md — <PARENT_DIR_NAME> 项目指令

> 本文件为 CodeBuddy 专属指令，引用 AGENTS.md 中的完整规则。
> CodeBuddy 在每次会话开始时自动读取本文件。

## 强制约束

- 每步操作完成后暂停，等待用户确认
- 禁止跳过项目结构检测直接创建文件
- 已有的 openspec/specs/*.md 文件严禁覆盖
- 所有功能开发遵循 propose → apply → archive 流程
- 规范优先级：团队规范基准文件 > openspec/specs/ > 代码推断

## 参考文件

- 完整项目规则见 `AGENTS.md`
- 各子项目规范见 `<子项目>/openspec/specs/`
- 操作指南见 `OPENSPEC_SUPERPOWERS_GUIDE.md`

## 命令

- `/<PROJECT_PREFIX>-propose` — 需求提案全流程
- `/<PROJECT_PREFIX>-apply` — 代码实施全流程
- `/<PROJECT_PREFIX>-archive` — 归档收尾全流程
```

> **注意**: CODEBUDDY.md 支持使用 `@docs/xxx.md` 语法引用外部文档，如需引用额外文档可使用此语法。

---

## 7. Shell 脚本双版本

SKILL.md 中的检测脚本默认提供 PowerShell 版本（Windows），以下为对应的 bash 版本（macOS/Linux 通用，Claude Code / Codex 环境常用）。

### 7.1 multi-repo 结构检测

**PowerShell:**
```powershell
Test-Path ".git"
Get-ChildItem -Directory | ForEach-Object {
    $hasGit = Test-Path "$($_.Name)/.git"
    Write-Output "$($_.Name) | git=$hasGit"
}
```

**bash:**
```bash
test -d .git && echo "HAS_GIT" || echo "NO_GIT"
for d in */; do
    [ -d "$d/.git" ] && echo "$d → 独立仓库" || echo "$d → 无 git"
done
```

### 7.2 子项目技术栈检测

**PowerShell:**
```powershell
Get-ChildItem -Directory | ForEach-Object {
    $name = $_.Name
    $tech = @()
    if (Test-Path "$name/package.json") { $tech += "Node/前端" }
    if (Test-Path "$name/pom.xml") { $tech += "Java/Maven" }
    if (Test-Path "$name/build.gradle") { $tech += "Java/Gradle" }
    if (Test-Path "$name/go.mod") { $tech += "Go" }
    if (Test-Path "$name/requirements.txt") { $tech += "Python" }
    Write-Output "$name | [$($tech -join ', ')]"
}
```

**bash:**
```bash
for d in */; do
    name="${d%/}"
    tech=""
    [ -f "$d/package.json" ] && tech="${tech}Node/前端, "
    [ -f "$d/pom.xml" ] && tech="${tech}Java/Maven, "
    [ -f "$d/build.gradle" ] && tech="${tech}Java/Gradle, "
    [ -f "$d/go.mod" ] && tech="${tech}Go, "
    [ -f "$d/requirements.txt" ] && tech="${tech}Python, "
    echo "$name | [${tech%, }]"
done
```

### 7.3 规范文件检查

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

---

## 8. 后续扩展指引

### 如何为新工具添加条件分支

当需要支持 Trae 或其他新 AI 工具时，按以下步骤扩展：

1. **Step 0 检测逻辑**: 在检测脚本中增加新的检测条件（如 `~/.trae/`）
2. **AI_TOOL 变量**: 添加新的值（如 `trae`）
3. **工具配置映射表**: 在本文第 2 节增加新列
4. **SKILL.md 条件分支**: 在每个 `<!-- AI_TOOL = ... -->` 块中增加新的分支
5. **文件格式**: 根据新工具的 Skill/命令机制选择对应格式
6. **Superpowers 兼容性**: 评估新工具是否支持子代理编排，不支持则需使用内联步骤替代（参考本文第 4 节的内联模板）
7. **PLAYBOOK.md**: 在对应 Phase 的条件分支中增加新工具的模板

### Trae 扩展备忘（待实现）

- Skill 路径: `.trae/rules/<PROJECT_PREFIX>-{phase}.md`
- 触发方式: `#<PROJECT_PREFIX>-{phase}`
- 约束标签: Rules `alwaysApply: true`
- 项目级指令文件: `AGENTS.md` + `.trae/rules/main.md`
- Superpowers: 不支持原生子代理编排，需使用内联步骤
