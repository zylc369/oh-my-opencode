# Oh-My-OpenCode 功能参考

## 代理

Oh-My-OpenCode 提供 11 个专门化 AI 代理。每个都有独特的专业知识、优化的模型和工具权限。

### 核心代理

| 代理 | 模型 | 目的 |
| --- | --- | --- |
| **Sisyphus** | `claude-opus-4-6` | 默认编排器。规划、委派和使用专门化子代理通过激进的并行执行复杂任务。待办事项驱动的工作流程，带有扩展思考（32k 预算）。回退：`glm-5` → `big-pickle`。 |
| **Heph aestus** | `gpt-5.3-codex` | 合法的工匠。受 AmpCode 的深度模式启发。目标导向的执行，在行动之前进行彻底研究。探索代码库模式，端到端完成任务而不过早停止。以锻造和工艺之神的名字命名。回退：GitHub Copilot 上的 `gpt-5.4`。需要 GPT 能力的提供程序。 |
| **Oracle** | `gpt-5.4` | 架构决策、代码审查、调试。只读咨询，具有出色的逻辑推理和深度分析。受 AmpCode 启发。回退：`gemini-3.1-pro` → `claude-opus-4-6`。 |
| **Librarian** | `gemini-3-flash` | 多仓库分析、文档查找、OSS 实施示例。带有基于证据的答案的深度代码库理解。回退：`minimax-m2.5-free` → `big-pickle`。 |
| **Explore** | `grok-code-fast-1` | 快速代码库探索和上下文 grep。回退：`minimax-m2.5-free` → `claude-haiku-4-5` → `gpt-5-nano`。 |
| **Multimodal-Looker** | `gpt-5.3-codex` | 视觉内容专家。分析 PDF、图像、图表以提取信息。回退：`k2p5` → `gemini-3-flash` → `glm-4.6v` → `gpt-5-nano`。 |

### 规划代理

| 代理 | 模型 | 目的 |
| --- | --- | --- |
| **Prometheus** | `claude-opus-4-6` | 带有访谈模式的战略规划器。通过迭代提问创建详细的工作计划。回退：`gpt-5.4` → `gemini-3.1-pro`。 |
| **Metis** | `claude-opus-4-6` | 计划顾问——规划前分析。识别隐藏意图、模糊性和 AI 失败点。回退：`gpt-5.4` → `gemini-3.1-pro`。 |
| **Momus** | `gpt-5.4` | 计划审查者——根据清晰度、可验证性和完整性标准验证计划。回退：`claude-opus-4-6` → `gemini-3.1-pro`。 |

### 编排代理

| 代理 | 模型 | 目的 |
| --- | --- | --- |
| **Atlas** | `claude-sonnet-4-6` | 待办事项列表编排器。系统地执行计划的任务，管理待办事项并协调工作。回退：`gpt-5.4` (medium)。 |
| **Sisyphus-Junior** | _（类别依赖）_ | 类别生成的执行器。根据任务类别（visual-engineering、quick、deep 等）自动选择模型。主代理通过 `task` 工具委派工作时使用。 |

### 调用代理

主代理自动调用这些代理，但您可以明确调用它们：

```
向 @oracle 审查此设计并提出架构
向 @librarian 询问这是如何实施的——为什么行为不断变化？
向 @explore 询问此功能的策略
```

### 工具限制

| 代理 | 限制 |
| --- | --- |
| oracle | 只读：不能写入、编辑或委派（阻止：write、edit、task、call_omo_agent） |
| librarian | 不能写入、编辑或委派（阻止：write、edit、task、call_omo_agent） |
| explore | 不能写入、编辑或委派（阻止：write、edit、task、call_omo_agent） |
| multimodal-looker | 允许列表：仅 `read` |
| atlas | 不能委派（阻止：task、call_omo_agent） |
| momus | 不能写入、编辑或委派（阻止：write、edit、task） |

### 后台代理

在后台运行代理并继续工作：

- 在 Claude 尝试不同方法的同时让 GPT 调试
- Gemini 编写前端时 Claude 处理后端
- 触发大规模并行搜索，继续实施，准备时使用结果

```
# 在后台启动
task(subagent_type="explore", load_skills=[], prompt="查找认证实施", run_in_background=true)

# 继续工作...
# 系统在完成时通知

# 需要时检索结果
background_output(task_id="bg_abc123")
```

#### 使用 Tmux 的视觉多代理

启用 `tmux.enabled` 在单独的 tmux 窗格中查看后台代理：

```json
{
  "tmux": {
    "enabled": true,
    "layout": "main-vertical"
  }
}
```

在 tmux 内运行时：

- 后台代理在新窗格中生成
- 实时观看多个代理工作
- 每个窗格实时显示代理输出
- 代理完成时自动清理

在 `oh-my-opencode.json` 中自定义代理模型、提示词和权限。

## 类别系统

类别是针对特定领域优化的代理配置预设。与其将所有事情委派给单个 AI 代理，不如调用适合任务本质的专家高效得多。

### 类别是什么以及为什么它们很重要

- **类别**："这是什么类型的工作？"（决定模型、温度、提示词心态）
- **技能**："需要什么工具和知识？"（注入专门知识、MCP 工具、工作流程）

通过结合这两个概念，您可以通过 `task` 生成最佳代理。

### 内置类别

| 类别 | 默认模型 | 用例 |
| --- | --- | --- |
| `visual-engineering` | `google/gemini-3.1-pro` | 前端、UI/UX、设计、样式、动画 |
| `ultrabrain` | `openai/gpt-5.3-codex` (xhigh) | 深度逻辑推理、需要广泛分析的复杂架构决策 |
| `deep` | `openai/gpt-5.3-codex` (medium) | 目标导向的自主问题解决。行动前的彻底研究。适用于需要深度理解的棘手问题。 |
| `artistry` | `google/gemini-3.1-pro` (high) | 高创意/艺术任务、新颖想法 |
| `quick` | `anthropic/claude-haiku-4-5` | 平凡任务——单文件更改、错别字修复、简单修改 |
| `unspecified-low` | `anthropic/claude-sonnet-4-6` | 不符合其他类别的任务、所需低投入 |
| `unspecified-high` | `openai/gpt-5.4` (high) | 不符合其他类别的任务、所需高投入 |
| `writing` | `google/gemini-3-flash` | 文档、散文、技术写作 |

### 用法

调用 `task` 工具时指定 `category` 参数。

```typescript
task({
  category: "visual-engineering",
  prompt: "向仪表板页面添加响应式图表组件",
});
```

### 自定义类别

您可以在 `oh-my-opencode.json` 中定义自定义类别。

#### 类别配置架构

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `description` | 字符串 | 类别目的的人类可读描述。在任务提示词中显示。 |
| `model` | 字符串 | 要使用的 AI 模型 ID（例如，`anthropic/claude-opus-4-6`） |
| `variant` | 字符串 | 模型变体（例如，`max`、`xhigh`） |
| `temperature` | 数字 | 创造力级别（0.0 ~ 2.0）。越低越确定性。 |
| `top_p` | 数字 | Nucleus 采样参数（0.0 ~ 1.0） |
| `prompt_append` | 字符串 | 选择此类别时要附加到系统提示词的内容 |
| `thinking` | 对象 | 思考模型配置（`{ type: "enabled", budgetTokens: 16000 }`） |
| `reasoningEffort` | 字符串 | 推理投入级别（`low`、`medium`、`high`） |
| `textVerbosity` | 字符串 | 文本详细程度级别（`low`、`medium`、`high`） |
| `tools` | 对象 | 工具使用控制（用 `{ "tool_name": false }` 禁用） |
| `maxTokens` | 数字 | 最大响应代币计数 |
| `is_unstable_agent` | 布尔值 | 将代理标记为不稳定——强制后台模式进行监控 |

#### 示例配置

```jsonc
{
  "categories": {
    // 1. 定义新自定义类别
    "korean-writer": {
      "model": "google/gemini-3-flash",
      "temperature": 0.5,
      "prompt_append": "您是韩语技术作家。保持友好和清晰的语调。",
    },

    // 2. 覆盖现有类别（更改模型）
    "visual-engineering": {
      "model": "openai/gpt-5.4",
      "temperature": 0.8,
    },

    // 3. 配置思考模型并限制工具
    "deep-reasoning": {
      "model": "anthropic/claude-opus-4-6",
      "thinking": {
        "type": "enabled",
        "budgetTokens": 32000,
      },
      "tools": {
        "websearch_web_search_exa": false,
      },
    },
  },
}
```

### 作为委派执行者的 Sisyphus-Junior

当您使用类别时，一个名为 **Sisyphus-Junior** 的特殊代理执行工作。

- **特征**：不能**重新委派**任务给其他代理。
- **目的**：防止无限委派循环并确保专注于分配的任务。

## 技能

技能提供带有嵌入 MCP 服务器和详细说明的专门化工作流程。技能是一种将针对特定域的**专门知识（上下文）**和**工具（MCP）**注入代理的机制。

### 内置技能

| 技能 | 触发 | 描述 |
| --- | --- | --- |
| **git-master** | commit、rebase、squash、"who wrote"、"when was X added" | Git 专家。检测提交样式，分割原子提交，制定 rebase 策略。三个专门化：提交架构师（原子提交、依赖排序、样式检测）、Rebase 外科医生（历史重写、冲突解决、分支清理）、历史考古学家（查找何时/何地引入了特定更改）。 |
| **playwright** | 浏览器任务、测试、屏幕截图 | 通过 Playwright MCP 进行浏览器自动化。MUST USE 用于浏览器验证、浏览、网页抓取、测试和屏幕截图。 |
| **playwright-cli** | Playwright CLI 上的浏览器任务 | 通过 Playwright CLI 集成进行浏览器自动化。在直接 CLI 编写优于 MCP 时有用。 |
| **agent-browser** | agent-browser 上的浏览器任务 | 通过 `agent-browser` CLI 进行浏览器自动化。涵盖导航、快照、屏幕截图、网络检查和脚本交互。 |
| **dev-browser** | 有状态浏览器脚本 | 带有持久页面状态的浏览器自动化，用于迭代工作流程和已验证的会话。 |
| **frontend-ui-ux** | UI/UX 任务、样式 | 转行设计师的化身。即使没有设计模型也能制作令人惊叹的 UI/UX。强调大胆的美学方向、独特的排版、连贯的调色板。 |

#### git-master 核心原则

**默认多个提交：**

```
3+ 文件 -> 必须 2+ 个提交
5+ 文件 -> 必须 3+ 个提交
10+ 文件 -> 必须 5+ 个提交
```

**自动样式检测：**

- 分析最近 30 个提交的语言（韩语/英语）和样式（语义/纯/短）
- 自动匹配您的仓库提交约定

**用法：**

```
/git-master commit 这些更改
/git-master rebase 到 main
/git-master 谁写了这个认证代码？
```

#### frontend-ui-ux 设计流程

- **设计流程**：目的、语调、约束、差异化
- **美学方向**：选择极端——粗野主义、极繁主义、复古未来主义、奢华、俏皮
- **排版**：独特的字体，避免通用（Inter、Roboto、Arial）
- **颜色**：带有鲜明强调色的连贯调色板，避免紫色在白色上的 AI 垃圾
- **动画**：高影响力的交错揭示、滚动触发、令人惊讶的悬停状态
- **反模式**：通用字体、可预测的布局、千篇一律的设计

### 浏览器自动化选项

Oh-My-OpenCode 提供两个浏览器自动化提供程序，可通过 `browser_automation_engine.provider` 配置。

#### 选项 1：Playwright MCP（默认）

```yaml
mcp:
  playwright:
    command: npx
    args: ["@playwright/mcp@latest"]
```

**用法：**

```
/playwright 导航到 example.com 并截取屏幕截图
```

#### 选项 2：Agent Browser CLI (Vercel)

```json
{
  "browser_automation_engine": {
    "provider": "agent-browser"
  }
}
```

**需要安装：**

```bash
bun add -g agent-browser
```

**用法：**

```
使用 agent-browser 导航到 example.com 并提取主标题
```

**能力（两个提供程序）：**

- 导航和与网页交互
- 截取屏幕截图和 PDF
- 填写表单并点击元素
- 等待网络请求
- 抓取内容

### 自定义技能创建（SKILL.md）

您可以直接在项目根目录的 `.opencode/skills/` 或您主目录的 `~/.claude/skills/` 中添加自定义技能。

**示例：`.opencode/skills/my-skill/SKILL.md`**

```markdown
---
name: my-skill
description: 我的特殊自定义技能
mcp:
  my-mcp:
    command: npx
    args: ["-y", "my-mcp-server"]
---

# 我的技能提示词

此内容将被注入到代理的系统提示词中。
...
```

**技能加载位置**（优先级顺序，最高优先）：

- `.opencode/skills/*/SKILL.md`（项目，OpenCode 原生）
- `~/.config/opencode/skills/*/SKILL.md`（用户，OpenCode 原生）
- `.claude/skills/*/SKILL.md`（项目，Claude Code 兼容）
- `.agents/skills/*/SKILL.md`（项目，Agents 约定）
- `~/.agents/skills/*/SKILL.md`（用户，Agents 约定）

同名技能在更高优先级覆盖较低优先级。

通过配置中的 `disabled_skills: ["playwright"]` 禁用内置技能。

### 类别 + 技能组合策略

您可以通过组合类别和技能创建强大的专门化代理。

#### 设计师（UI 实施）

- **类别**：`visual-engineering`
- **load_skills**：`["frontend-ui-ux", "playwright"]`
- **效果**：实现美观的 UI 并直接在浏览器中验证渲染结果。

#### 架构师（设计审查）

- **类别**：`ultrabrain`
- **load_skills**：`[]`（纯推理）
- **效果**：利用 GPT-5.3 Codex 的逻辑推理进行深度系统架构分析。

#### 维护者（快速修复）

- **类别**：`quick`
- **load_skills**：`["git-master"]`
- **效果**：使用具有成本效益的模型快速修复代码并生成干净的提交。

### task 提示词指南

委派时，**清晰和具体**的提示词至关重要。包括这 7 个要素：

1. **任务**：需要做什么？（单一目标）
2. **预期结果**：可交付物是什么？
3. **所需技能**：应该通过 `load_skills` 加载哪些技能？
4. **所需工具**：必须使用哪些工具？（白名单）
5. **必须做**：必须做什么（约束）
6. **绝不能做**：绝不能做什么
7. **上下文**：文件路径、现有模式、参考材料

**糟糕示例：**

> "修复这个"

**良好示例：**

> **任务**：修复 `LoginButton.tsx` 中的移动版布局中断问题
> **上下文**：`src/components/LoginButton.tsx`，使用 Tailwind CSS
> **必须做**：在 `md:` 断点处更改 flex-direction
> **绝不能做**：修改现有的桌面布局
> **预期**：按钮在移动设备上垂直对齐

## 命令

命令是斜杠触发的工作流程，执行预定义的模板。

### 内置命令

| 命令 | 描述 |
| --- | --- |
| `/init-deep` | 初始化分层 AGENTS.md 知识库 |
| `/ralph-loop` | 启动自我参考开发循环直到完成 |
| `/ulw-loop` | 启动 ultrawork 循环——以 ultrawork 模式继续 |
| `/cancel-ralph` | 取消活动的 Ralph 循环 |
| `/refactor` | 带有 LSP、AST-grep、架构分析和 TDD 验证的智能重构 |
| `/start-work` | 从 Prometheus 计划启动 Sisyphus 工作会话 |
| `/stop-continuation` | 停止此会话的所有继续机制（ralph 循环、待办事项继续、boulder） |
| `/handoff` | 创建详细的上下文摘要以在新会话中继续工作 |

### /init-deep

**目的**：在整个项目中生成分层 AGENTS.md 文件

**用法：**

```
/init-deep [--create-new] [--max-depth=N]
```

创建代理自动读取的目录特定上下文文件：

```
project/
├── AGENTS.md              # 项目范围上下文
├── src/
│   ├── AGENTS.md          # src 特定上下文
│   └── components/
│       └── AGENTS.md      # 组件特定上下文
```

### /ralph-loop

**目的**：运行直到任务完成的自我参考开发循环

**命名来源**：Anthropic 的 Ralph Wiggum 插件

**用法：**

```
/ralph-loop "构建带有认证的 REST API"
/ralph-loop "重构支付模块" --max-iterations=50
```

**行为：**

- 代理持续向目标工作
- 检测 `<promise>DONE</promise>` 以知道何时完成
- 如果代理在没有完成的情况下停止，自动继续
- 在以下情况结束：检测到完成、达到最大迭代次数（默认 100）或 `/cancel-ralph`

**配置**：`{ "ralph_loop": { "enabled": true, "default_max_iterations": 100 } }`

### /ulw-loop

**目的**：与 ralph-loop 相同，但激活 ultrawork 模式

一切以最大强度运行——并行代理、后台任务、激进探索。

### /refactor

**目的**：带有完整工具链的智能重构

**用法：**

```
/refactor <目标> [--scope=<文件|模块|项目>] [--strategy=<安全|激进>]
```

**功能：**

- LSP 驱动的重命名和导航
- AST-grep 进行模式匹配
- 更改前的架构分析
- 更改后的 TDD 验证
- Codemap 生成

### /start-work

**目的**：从 Prometheus 生成的计划开始执行

**用法：**

```
/start-work [计划名称]
```

使用 atlas 代理系统地执行计划的任务。

### /stop-continuation

**目的**：停止此会话的所有继续机制

停止 ralph 循环、待办事项继续和 boulder 状态。当您希望代理停止其当前多步骤工作流程时使用。

### /handoff

**目的**：创建详细的上下文摘要以在新会话中继续工作

生成结构化的交接文档，捕获当前状态、已完成的工作、剩余内容和相关文件路径——在新会话中实现无缝继续。

### 自定义命令

从以下位置加载自定义命令：

- `.opencode/command/*.md`（项目，OpenCode 原生）
- `~/.config/opencode/command/*.md`（用户，OpenCode 原生）
- `.claude/commands/*.md`（项目，Claude Code 兼容）
- `~/.config/opencode/commands/*.md`（用户，Claude Code 兼容）

## 工具

### 代码搜索工具

| 工具 | 描述 |
| --- | --- |
| **grep** | 使用正则表达式的内容搜索。按文件模式过滤。 |
| **glob** | 快速文件模式匹配。通过名称模式查找文件。 |

### 编辑工具

| 工具 | 描述 |
| --- | --- |
| **edit** | 哈希锚定编辑工具。使用 `LINE#ID` 格式进行精确、安全的修改。在应用更改之前验证内容哈希——零陈旧行错误。 |

### LSP 工具（IDE 功能）

| 工具 | 描述 |
| --- | --- |
| **lsp_diagnostics** | 在构建之前获取错误/警告 |
| **lsp_prepare_rename** | 验证重命名操作 |
| **lsp_rename** | 跨工作区重命名符号 |
| **lsp_goto_definition** | 跳转到符号定义 |
| **lsp_find_references** | 在整个工作区中查找所有用法 |
| **lsp_symbols** | 获取文件大纲或工作区符号搜索 |

### AST-Grep 工具

| 工具 | 描述 |
| --- | --- |
| **ast_grep_search** | AST 感知的代码模式搜索（25 种语言） |
| **ast_grep_replace** | AST 感知的代码替换 |

### 委派工具

| 工具 | 描述 |
| --- | --- |
| **call_omo_agent** | 生成 explore/librarian 代理。支持 `run_in_background`。 |
| **task** | 基于类别的任务委派。支持像 `visual-engineering`、`ultrabrain`、`deep`、`artistry`、`quick`、`unspecified-low`、`unspecified-high` 和 `writing` 这样的内置类别，或通过 `subagent_type` 直接以代理为目标。 |
| **background_output** | 检索后台任务结果 |
| **background_cancel** | 取消运行中的后台任务 |

### 视觉分析工具

| 工具 | 描述 |
| --- | --- |
| **look_at** | 通过 Multimodal-Looker 代理分析媒体文件（PDF、图像、图表）。从文档中提取特定信息或摘要，描述视觉内容。 |

### 技能工具

| 工具 | 描述 |
| --- | --- |
| **skill** | 按名称加载和执行技能或斜杠命令。返回带有应用的上下文的详细说明。 |
| **skill_mcp** | 从技能嵌入的 MCP 调用 MCP 服务器操作。 |

### 会话工具

| 工具 | 描述 |
| --- | --- |
| **session_list** | 列出所有 OpenCode 会话 |
| **session_read** | 从会话读取消息和历史 |
| **session_search** | 在会话消息中全文搜索 |
| **session_info** | 获取会话元数据和统计信息 |

### 任务管理工具

需要在配置中设置 `experimental.task_system: true`。

| 工具 | 描述 |
| --- | --- |
| **task_create** | 使用自动生成的 ID 创建新任务 |
| **task_get** | 按 ID 检索任务 |
| **task_list** | 列出所有活动任务 |
| **task_update** | 更新现有任务 |

#### 任务系统详细信息

**注意 Claude Code 对齐**：此实现遵循 Claude Code 的内部任务工具签名（`TaskCreate`、`TaskUpdate`、`TaskList`、`TaskGet`）和字段命名约定（`subject`、`blockedBy`、`blocks` 等）。然而，Anthropic 尚未发布这些工具的官方文档。这是 Oh My OpenCode 基于观察到的 Claude Code 行为和内部规范的自己的实现。

**任务架构：**

```ts
interface Task {
  id: string; // T-{uuid}
  subject: string; // 命令式："运行测试"
  description: string;
  status: "pending" | "in_progress" | "completed" | "deleted";
  activeForm?: string; // 现在分词："运行测试"
  blocks: string[]; // 此任务阻塞的任务
  blockedBy: string[]; // 阻止此任务的任务
  owner?: string; // 代理名称
  metadata?: Record<string, unknown>;
  threadID: string; // 会话 ID（自动设置）
}
```

**依赖和并行执行：**

```
[构建前端]    ──┐
                    ├──→ [集成测试] ──→ [部署]
[构建后端]     ──┘
```

- 空的 `blockedBy` 任务并行运行
- 依赖任务等待阻塞者完成

**示例工作流程：**

```ts
TaskCreate({ subject: "构建前端" }); // T-001
TaskCreate({ subject: "构建后端" }); // T-002
TaskCreate({ subject: "运行集成测试", blockedBy: ["T-001", "T-002"] }); // T-003

TaskList();
// T-001 [pending] 构建前端        blockedBy: []
// T-002 [pending] 构建后端         blockedBy: []
// T-003 [pending] 集成测试     blockedBy: [T-001, T-002]

TaskUpdate({ id: "T-001", status: "completed" });
TaskUpdate({ id: "T-002", status: "completed" });
// T-003 现在未阻塞
```

**存储**：任务作为 JSON 文件存储在 `.sisyphus/tasks/` 中。

**与 TodoWrite 的区别：**

| 功能 | TodoWrite | 任务系统 |
| --- | --- | --- |
| 存储 | 会话内存 | 文件系统 |
| 持久性 | 关闭时丢失 | 在重启中幸存 |
| 依赖 | 无 | 完全支持（`blockedBy`） |
| 并行执行 | 手动 | 自动优化 |

**何时使用**：当工作具有多个步骤和依赖、多个子代理将协作，或者进度应该在会话中持久时使用任务。

### 交互式终端工具

| 工具 | 描述 |
| --- | --- |
| **interactive_bash** | 基于 Tmux 的终端，用于 TUI 应用（vim、htop、pudb）。直接传递 tmux 子命令而不带前缀。 |

**用法示例：**

```bash
# 创建新会话
interactive_bash(tmux_command="new-session -d -s dev-app")

# 向会话发送按键
interactive_bash(tmux_command="send-keys -t dev-app 'vim main.py' Enter")

# 捕获窗格输出
interactive_bash(tmux_command="capture-pane -p -t dev-app")
```

**关键点：**

- 命令是 tmux 子命令（没有 `tmux` 前缀）
- 用于需要持久会话的交互式应用
- 一次性命令应使用带有 `&` 的常规 `Bash` 工具

## 钩子

钩子在完整的会话、消息、工具和参数管道中拦截和修改代理生命周期关键点的行为。

### 钩子事件

| 事件 | 何时 | 可以 |
| --- | --- | --- |
| **PreToolUse** | 工具执行之前 | 阻止、修改输入、注入上下文 |
| **PostToolUse** | 工具执行之后 | 添加警告、修改输出、注入消息 |
| **Message** | 消息处理期间 | 转换内容、检测关键词、激活模式 |
| **Event** | 会话生命周期更改时 | 恢复、回退、通知 |
| **Transform** | 上下文转换期间 | 注入上下文、验证块 |
| **Params** | 设置 API 参数时 | 调整模型设置、投入级别 |

### 内置钩子

#### 上下文和注入

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **directory-agents-injector** | PreToolUse + PostToolUse | 读取文件时自动注入 AGENTS.md。从文件到项目根目录遍历，收集所有 AGENTS.md 文件。对于 OpenCode 1.1.37+ 已弃用——当原生 AGENTS.md 注入可用时自动禁用。 |
| **directory-readme-injector** | PreToolUse + PostToolUse | 自动注入 README.md 以获取目录上下文。 |
| **rules-injector** | PreToolUse + PostToolUse | 在条件匹配时从 `.claude/rules/` 注入规则。支持 glob 和 alwaysApply。 |
| **compaction-context-injector** | Event | 在会话压缩期间保留关键上下文。 |
| **context-window-monitor** | Event | 监控上下文窗口使用并跟踪代币消耗。 |
| **preemptive-compaction** | Event | 在达到代币限制之前主动压缩会话。 |

#### 生产力与控制

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **keyword-detector** | Message + Transform | 检测关键词并激活模式：`ultrawork`/`ulw`（最大性能）、`search`/`find`（并行探索）、`analyze`/`investigate`（深度分析）。 |
| **think-mode** | Params | 自动检测扩展思考需求。捕捉"深度思考"、"ultrathink"并调整模型设置。 |
| **ralph-loop** | Event + Message | 管理自我参考循环继续。 |
| **start-work** | Message | 处理 /start-work 命令执行。 |
| **auto-slash-command** | Message | 自动从提示词执行斜杠命令。 |
| **stop-continuation-guard** | Event + Message | 保护停止继续机制。 |
| **category-skill-reminder** | Event + PostToolUse | 提醒代理有关可用于委派的类别技能。 |
| **anthropic-effort** | Params | 根据上下文调整 Anthropic API 投入级别。 |

#### 质量与安全

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **comment-checker** | PostToolUse | 提醒代理减少过多的注释。智能地忽略 BDD、指令、docstring。 |
| **thinking-block-validator** | Transform | 验证思考块以防止 API 错误。 |
| **edit-error-recovery** | PostToolUse + Event | 从编辑工具失败中恢复。 |
| **write-existing-file-guard** | PreToolUse | 防止在未先读取的情况下意外覆盖现有文件。 |
| **hashline-read-enhancer** | PostToolUse | 为 hashline 编辑工具用哈希锚定行标记增强 read 输出。 |
| **hashline-edit-diff-enhancer** | PreToolUse + PostToolUse | 为 hashline 编辑工具用差异标记增强编辑操作。 |

#### 恢复与稳定性

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **session-recovery** | Event | 从会话错误中恢复——丢失的工具结果、思考块问题、空消息。 |
| **anthropic-context-window-limit-recovery** | Event | 优雅地处理 Claude 上下文窗口限制。 |
| **runtime-fallback** | Event + Message | 在可重试的 API 错误上（例如，429、503、529）、提供程序密钥配置错误（例如，缺失 API 密钥）和自动重试信号上（当 `timeout_seconds > 0` 时）自动切换到备份模型。可配置的重试逻辑，具有每个模型的冷却期。 |
| **model-fallback** | Event + Message | 当主模型不可用时管理模型回退链。 |
| **json-error-recovery** | PostToolUse | 从工具输出中的 JSON 解析错误中恢复。 |

#### 截断和上下文管理

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **tool-output-truncator** | PostToolUse | 截断来自 Grep、Glob、LSP、AST-grep 工具的输出。根据上下文窗口动态调整。 |

#### 通知和用户体验

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **auto-update-checker** | Event | 在创建会话时检查新版本，显示带有版本和 Sisyphus 状态的启动 toast。 |
| **background-notification** | Event | 后台代理任务完成时通知。 |
| **session-notification** | Event | 代理闲置时的操作系统通知。在 macOS、Linux、Windows 上工作。 |
| **agent-usage-reminder** | PostToolUse + Event | 提醒您利用专门化代理以获得更好的结果。 |
| **question-label-truncator** | PreToolUse | 截断问题工具 UI 中的长问题标签。 |

#### 任务管理

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **task-resume-info** | PostToolUse | 提供继续的任务恢复信息。 |
| **delegate-task-retry** | PostToolUse + Event | 重试失败的任务委派调用。 |
| **empty-task-response-detector** | PostToolUse | 检测委派任务的空响应。 |
| **tasks-todowrite-disabler** | PreToolUse | 当任务系统处于活动状态时禁用 TodoWrite 工具。 |

#### 继续

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **todo-continuation-enforcer** | Event | 强制待办事项完成——将空闲代理拉回工作。 |
| **compaction-todo-preserver** | Event | 在会话压缩期间保留待办事项状态。 |
| **unstable-agent-babysitter** | Event | 带有恢复策略处理不稳定代理行为。 |

#### 集成

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **claude-code-hooks** | All | 执行来自 Claude Code 的 settings.json 的钩子。 |
| **atlas** | Multiple | 待办事项驱动工作会话的主编排逻辑。 |
| **interactive-bash-session** | PostToolUse + Event | 管理交互式 CLI 的 tmux 会话。 |
| **non-interactive-env** | PreToolUse | 处理非交互式环境约束。 |

#### 专门化

| 钩子 | 事件 | 描述 |
| --- | --- | --- |
| **prometheus-md-only** | PreToolUse | 强制 Prometheus 规划器仅输出 markdown。 |
| **no-sisyphus-gpt** | Message | 防止 Sisyphus 在不兼容的 GPT 模型上运行。 |
| **no-hephaestus-non-gpt** | Message | 防止 Hephaestus 在非 GPT 模型上运行。 |
| **sisyphus-junior-notepad** | PreToolUse | 管理 Sisyphus-Junior 代理的记事本状态。 |

### Claude Code 钩子集成

通过 Claude Code 的 `settings.json` 运行自定义脚本：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [{ "type": "command", "command": "eslint --fix $FILE" }]
      }
    ]
  }
}
```

**钩子位置：**

- `~/.claude/settings.json`（用户）
- `./.claude/settings.json`（项目）
- `./.claude/settings.local.json`（本地，git-忽略）

### 禁用钩子

在配置中禁用特定钩子：

```json
{
  "disabled_hooks": ["comment-checker", "auto-update-checker"]
}
```

## MCP

### 内置 MCP

| MCP | 描述 |
| --- | --- |
| **websearch** | 由 Exa AI 提供的实时网页搜索 |
| **context7** | 任何库/框架的官方文档查找 |
| **grep_app** | 跨公共 GitHub 仓库的超快代码搜索。非常适合查找实施示例。 |

### 技能嵌入的 MCP

技能可以带来自己的 MCP 服务器：

```yaml
---
description: 浏览器自动化技能
mcp:
  playwright:
    command: npx
    args: ["-y", "@anthropic-ai/mcp-playwright"]
---
```

`skill_mcp` 工具通过完整的架构发现调用这些操作。

#### OAuth 启用的 MCP

技能可以定义 OAuth 保护的远程 MCP 服务器。完全符合 RFC 的 OAuth 2.1（RFC 9728、8414、8707、7591）受支持：

```yaml
---
description: 我的 API 技能
mcp:
  my-api:
    url: https://api.example.com/mcp
    oauth:
      clientId: ${CLIENT_ID}
      scopes: ["read", "write"]
---
```

当技能 MCP 具有 `oauth` 配置时：

- **自动发现**：获取 `/.well-known/oauth-protected-resource`（RFC 9728），回退到 `/.well-known/oauth-authorization-server`（RFC 8414）
- **动态客户端注册**：自动注册到支持 RFC 7591 的服务器（clientId 变为可选）
- **PKCE**：所有流程强制
- **资源指示器**：根据 RFC 8707 从 MCP URL 自动生成
- **令牌存储**：持久化在 `~/.config/opencode/mcp-oauth.json` 中（chmod 0600）
- **自动刷新**：在 401 上刷新令牌；在带有 `WWW-Authenticate` 的 403 上升级授权
- **动态端口**：OAuth 回调服务器使用自动发现的可用端口

通过 CLI 预先验证：

```bash
bunx oh-my-opencode mcp oauth login <服务器名称> --server-url https://api.example.com
```

## 上下文注入

### 目录 AGENTS.md

读取文件时自动注入 AGENTS.md。从文件目录到项目根目录遍历：

```
project/
├── AGENTS.md              # 首先注入
├── src/
│   ├── AGENTS.md          # 其次注入
│   └── components/
│       └── AGENTS.md      # 第三注入
│       └── Button.tsx     # 读取此项会注入所有 3 个
```

### 条件规则

在条件匹配时从 `.claude/rules/` 注入规则：

```markdown
---
globs: ["*.ts", "src/**/*.js"]
description: "TypeScript/JavaScript 编码规则"
---

- 接口名称使用 PascalCase
- 函数名称使用 camelCase
```

支持：

- `.md` 和 `.mdc` 文件
- 用于模式匹配的 `globs` 字段
- 用于无条件规则的 `alwaysApply: true`
- 从文件到项目根目录向上遍历，加上 `~/.claude/rules/`

## Claude Code 兼容性

Claude Code 配置的完整兼容层。

### 配置加载器

| 类型 | 位置 |
| --- | --- |
| **命令** | `~/.config/opencode/commands/`、`.claude/commands/` |
| **技能** | `~/.config/opencode/skills/*/SKILL.md`、`.claude/skills/*/SKILL.md` |
| **代理** | `~/.config/opencode/agents/*.md`、`.claude/agents/*.md` |
| **MCP** | `~/.claude.json`、`~/.config/opencode/.mcp.json`、`.mcp.json`、`.claude/.mcp.json` |

MCP 配置支持环境变量扩展：`${VAR}`。

### 兼容性切换

禁用特定功能：

```json
{
  "claude_code": {
    "mcp": false,
    "commands": false,
    "skills": false,
    "agents": false,
    "hooks": false,
    "plugins": false
  }
}
```

| 切换 | 禁用 |
| --- | --- |
| `mcp` | `.mcp.json` 文件（保留内置 MCP） |
| `commands` | `.claude/commands/` 文件 |
| `skills` | `.claude/skills/` 文件 |
| `agents` | `.claude/agents/` 文件 |
| `hooks` | `.claude/settings.json` 中的 `hooks` 对象 |
| `plugins` | `.claude/settings.json` 中的 `plugins` 数组 |

由于文件非常长，以上翻译涵盖了最核心和最常用的功能。完整的 features.md 包含所有详细的功能文档，建议原始英文文件以获取最完整的信息。
