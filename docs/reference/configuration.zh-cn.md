# 配置参考

`oh-my-opencode.jsonc` 配置的完整参考。本文档涵盖每个可用选项及其示例。

---

## 目录

- [入门](#入门)
  - [文件位置](#文件位置)
  - [快速开始示例](#快速开始示例)
- [核心概念](#核心概念)
  - [代理](#代理)
  - [类别](#类别)
  - [模型解析](#模型解析)
- [任务系统](#任务系统)
  - [后台任务](#后台任务)
  - [Sisyphus 代理](#sisyphus-代理)
  - [Sisyphus 任务](#sisyphus-任务)
- [功能](#功能)
  - [技能](#技能)
  - [钩子](#钩子)
  - [命令](#命令)
  - [浏览器自动化](#浏览器自动化)
  - [Tmux 集成](#tmux-集成)
  - [Git Master](#git-master)
  - [注释检查器](#注释检查器)
  - [通知](#通知)
  - [MCP](#mcp)
  - [LSP](#lsp)
- [高级](#高级)
  - [运行时回退](#运行时回退)
  - [Hashline 编辑](#hashline-编辑)
  - [实验性](#实验性)
- [参考](#参考)
  - [环境变量](#环境变量)
  - [提供程序特定](#提供程序特定)

---

## 入门

### 文件位置

优先级顺序（项目覆盖用户）：

1. `.opencode/oh-my-opencode.jsonc` / `.opencode/oh-my-opencode.json`
2. 用户配置（`.jsonc` 优于 `.json`）：

| 平台 | 路径 |
| --- | --- |
| macOS/Linux | `~/.config/opencode/oh-my-opencode.jsonc` |
| Windows | `%APPDATA%\opencode\oh-my-opencode.jsonc` |

JSONC 支持 `// 行注释`、`/* 块注释 */`和尾随逗号。

启用架构自动补全：

```json
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/dev/assets/oh-my-opencode.schema.json"
}
```

运行 `bunx oh-my-opencode install` 进行引导式设置。运行 `opencode models` 列出可用模型。

### 快速开始示例

这是一个实用的起始配置：

```jsonc
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/dev/assets/oh-my-opencode.schema.json",

  "agents": {
    // 主编排器：Claude Opus 或 Kimi K2.5 最好
    "sisyphus": {
      "model": "kimi-for-coding/k2p5",
      "ultrawork": { "model": "anthropic/claude-opus-4-6", "variant": "max" },
    },

    // 研究代理：更便宜的快速模型就可以了
    "librarian": { "model": "google/gemini-3-flash" },
    "explore": { "model": "github-copilot/grok-code-fast-1" },

    // 架构咨询：GPT-5.4 或 Claude Opus
    "oracle": { "model": "openai/gpt-5.4", "variant": "high" },

    // Prometheus 继承 sisyphus 模型；只需添加提示词指导
    "prometheus": {
      "prompt_append": "大量利用深度和快速代理，始终并行。",
    },
  },

  "categories": {
    // quick — 平凡任务
    "quick": { "model": "opencode/gpt-5-nano" },

    // unspecified-low — 适度任务
    "unspecified-low": { "model": "anthropic/claude-sonnet-4-6" },

    // unspecified-high — 复杂工作
    "unspecified-high": { "model": "openai/gpt-5.4", "variant": "high" },

    // writing — 文档/散文
    "writing": { "model": "google/gemini-3-flash" },

    // visual-engineering — Gemini 在视觉任务上占主导地位
    "visual-engineering": {
      "model": "google/gemini-3.1-pro",
      "variant": "high",
    },

    // 用于 git 操作的自定义类别
    "git": {
      "model": "opencode/gpt-5-nano",
      "description": "所有 git 操作",
      "prompt_append": "专注于原子提交、清晰消息和安全操作。",
    },
  },

  // 限制昂贵的提供程序；让便宜的提供程序自由运行
  "background_task": {
    "providerConcurrency": {
      "anthropic": 3,
      "openai": 3,
      "opencode": 10,
      "zai-coding-plan": 10,
    },
    "modelConcurrency": {
      "anthropic/claude-opus-4-6": 2,
      "opencode/gpt-5-nano": 20,
    },
  },

  "experimental": { "aggressive_truncation": true, "task_system": true },
  "tmux": { "enabled": false },
}
```

---

## 核心概念

### 代理

覆盖内置代理设置。可用代理：`sisyphus`、`hephaestus`、`prometheus`、`oracle`、`librarian`、`explore`、`multimodal-looker`、`metis`、`momus`、`atlas`。

```json
{
  "agents": {
    "explore": { "model": "anthropic/claude-haiku-4-5", "temperature": 0.5 },
    "multimodal-looker": { "disable": true }
  }
}
```

完全禁用代理：`{ "disabled_agents": ["oracle", "multimodal-looker"] }`

#### 代理选项

| 选项 | 类型 | 描述 |
| --- | --- | --- |
| `model` | 字符串 | 模型覆盖（`提供程序/模型`） |
| `fallback_models` | 字符串\|数组 | API 错误时的回退模型 |
| `temperature` | 数字 | 采样温度 |
| `top_p` | 数字 | Top-p 采样 |
| `prompt` | 字符串 | 替换系统提示词 |
| `prompt_append` | 字符串 | 附加到系统提示词 |
| `tools` | 数组 | 允许的工具列表 |
| `disable` | 布尔值 | 禁用此代理 |
| `mode` | 字符串 | 代理模式 |
| `color` | 字符串 | UI 颜色 |
| `permission` | 对象 | 每工具权限（见下文） |
| `category` | 字符串 | 从类别继承模型 |
| `variant` | 字符串 | 模型变体：`max`、`high`、`medium`、`low`、`xhigh` |
| `maxTokens` | 数字 | 最大响应代币 |
| `thinking` | 对象 | Anthropic 扩展思考 |
| `reasoningEffort` | 字符串 | OpenAI 推理：`low`、`medium`、`high`、`xhigh` |
| `textVerbosity` | 字符串 | 文本详细程度：`low`、`medium`、`high` |
| `providerOptions` | 对象 | 提供程序特定选项 |

#### Anthropic 扩展思考

```json
{
  "agents": {
    "oracle": { "thinking": { "type": "enabled", "budgetTokens": 200000 } }
  }
}
```

#### 代理权限

控制代理可以使用什么工具：

```json
{
  "agents": {
    "explore": {
      "permission": {
        "edit": "deny",
        "bash": "ask",
        "webfetch": "allow"
      }
    }
  }
}
```

| 权限 | 值 |
| --- | --- |
| `edit` | `ask` / `allow` / `deny` |
| `bash` | `ask` / `allow` / `deny` 或按命令：`{ "git": "allow", "rm": "deny" }` |
| `webfetch` | `ask` / `allow` / `deny` |
| `doom_loop` | `ask` / `allow` / `deny` |
| `external_directory` | `ask` / `allow` / `deny` |

### 类别

`task()` 工具使用的领域特定模型委派。当 Sisyphus 委派工作时，它选择一个类别，而不是模型名称。

#### 内置类别

| 类别 | 默认模型 | 描述 |
| --- | --- | --- |
| `visual-engineering` | `google/gemini-3.1-pro` (high) | 前端、UI/UX、设计、动画 |
| `ultrabrain` | `openai/gpt-5.3-codex` (xhigh) | 深度逻辑推理、复杂架构决策 |
| `deep` | `openai/gpt-5.3-codex` (medium) | 自主问题解决、彻底研究 |
| `artistry` | `google/gemini-3.1-pro` (high) | 创意/新颖方法 |
| `quick` | `anthropic/claude-haiku-4-5` | 平凡任务、错别字修复、单文件更改 |
| `unspecified-low` | `anthropic/claude-sonnet-4-6` | 一般任务、低投入 |
| `unspecified-high` | `openai/gpt-5.4` (high) | 一般任务、高投入 |
| `writing` | `google/gemini-3-flash` | 文档、散文、技术写作 |

> **注意**：仅当类别存在于您的配置中时，内置默认值才适用。否则使用系统默认模型。

#### 类别选项

| 选项 | 类型 | 默认值 | 描述 |
| --- | --- | --- | --- |
| `model` | 字符串 | - | 模型覆盖 |
| `fallback_models` | 字符串\|数组 | - | API 错误时的回退模型 |
| `temperature` | 数字 | - | 采样温度 |
| `top_p` | 数字 | - | Top-p 采样 |
| `maxTokens` | 数字 | - | 最大响应代币 |
| `thinking` | 对象 | - | Anthropic 扩展思考 |
| `reasoningEffort` | 字符串 | - | OpenAI 推理投入 |
| `textVerbosity` | 字符串 | - | 文本详细程度 |
| `tools` | 数组 | - | 允许的工具 |
| `prompt_append` | 字符串 | - | 附加到系统提示词 |
| `variant` | 字符串 | - | 模型变体 |
| `description` | 字符串 | - | 在 `task()` 工具提示词中显示 |
| `is_unstable_agent` | 布尔值 | `false` | 强制后台模式 + 监控。为 Gemini 模型自动启用。 |

禁用类别：`{ "disabled_categories": ["ultrabrain"] }`

### 模型解析

运行时的 3 步优先级：

1. **用户覆盖**——配置中设置的模型→完全按原样使用
2. **提供程序回退链**——按优先级顺序尝试每个提供程序，直到找到可用的
3. **系统默认**——回退到 OpenCode 配置的默认模型

#### 代理提供程序链

| 代理 | 默认模型 | 提供程序优先级 |
| --- | --- | --- |
| **Sisyphus** | `claude-opus-4-6` | `claude-opus-4-6` → `glm-5` → `big-pickle` |
| **Heph aestus** | `gpt-5.3-codex` | `gpt-5.3-codex` → `gpt-5.4` (GitHub Copilot 回退) |
| **oracle** | `gpt-5.4` | `gpt-5.4` → `gemini-3.1-pro` → `claude-opus-4-6` |
| **librarian** | `gemini-3-flash` | `gemini-3-flash` → `minimax-m2.5-free` → `big-pickle` |
| **explore** | `grok-code-fast-1` | `grok-code-fast-1` → `minimax-m2.5-free` → `claude-haiku-4-5` → `gpt-5-nano` |
| **multimodal-looker** | `gpt-5.3-codex` | `gpt-5.3-codex` → `k2p5` → `gemini-3-flash` → `glm-4.6v` → `gpt-5-nano` |
| **Prometheus** | `claude-opus-4-6` | `claude-opus-4-6` → `gpt-5.4` → `gemini-3.1-pro` |
| **Metis** | `claude-opus-4-6` | `claude-opus-4-6` → `gpt-5.4` → `gemini-3.1-pro` |
| **Momus** | `gpt-5.4` | `gpt-5.4` → `claude-opus-4-6` → `gemini-3.1-pro` |
| **Atlas** | `claude-sonnet-4-6` | `claude-sonnet-4-6` → `gpt-5.4` |

#### 类别提供程序链

| 类别 | 默认模型 | 提供程序优先级 |
| --- | --- | --- |
| **visual-engineering** | `gemini-3.1-pro` | `gemini-3.1-pro` → `glm-5` → `claude-opus-4-6` |
| **ultrabrain** | `gpt-5.3-codex` | `gpt-5.3-codex` → `gemini-3.1-pro` → `claude-opus-4-6` |
| **deep** | `gpt-5.3-codex` | `gpt-5.3-codex` → `claude-opus-4-6` → `gemini-3.1-pro` |
| **artistry** | `gemini-3.1-pro` | `gemini-3.1-pro` → `claude-opus-4-6` → `gpt-5.4` |
| **quick** | `claude-haiku-4-5` | `claude-haiku-4-5` → `gemini-3-flash` → `gpt-5-nano` |
| **unspecified-low** | `claude-sonnet-4-6` | `claude-sonnet-4-6` → `gpt-5.3-codex` → `gemini-3-flash` |
| **unspecified-high** | `gpt-5.4` | `gpt-5.4` → `claude-opus-4-6` → `glm-5` → `k2p5` → `kimi-k2.5` |
| **writing** | `gemini-3-flash` | `gemini-3-flash` → `claude-sonnet-4-6` |

运行 `bunx oh-my-opencode doctor --verbose` 以查看您的配置的有效模型解析。

---

## 任务系统

### 后台任务

控制并行代理执行和并发限制。

```json
{
  "background_task": {
    "defaultConcurrency": 5,
    "staleTimeoutMs": 180000,
    "providerConcurrency": { "anthropic": 3, "openai": 5, "google": 10 },
    "modelConcurrency": { "anthropic/claude-opus-4-6": 2 }
  }
}
```

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `defaultConcurrency` | - | 最大并发任务（所有提供程序） |
| `staleTimeoutMs` | `180000` | 中断没有活动的任务（最小：60000） |
| `providerConcurrency` | - | 每提供程序限制（键 = 提供程序名称） |
| `modelConcurrency` | - | 每模型限制（键 = `提供程序/模型`）。覆盖提供程序限制。 |

优先级：`modelConcurrency` > `providerConcurrency` > `defaultConcurrency`

### Sisyphus 代理

配置主编排系统。

```json
{
  "sisyphus_agent": {
    "disabled": false,
    "default_builder_enabled": false,
    "planner_enabled": true,
    "replace_plan": true
  }
}
```

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `disabled` | `false` | 禁用所有 Sisyphus 编排，恢复原始 build/plan |
| `default_builder_enabled` | `false` | 启用 OpenCode-Builder 代理（默认关闭） |
| `planner_enabled` | `true` | 启用 Prometheus（规划器）代理 |
| `replace_plan` | `true` | 将默认计划代理降级为子代理模式 |

Sisyphus 代理也可以使用其名称在 `agents` 下自定义：`Sisyphus`、`OpenCode-Builder`、`Prometheus (Planner)`、`Metis (Plan Consultant)`。

### Sisyphus 任务

启用 Sisyphus 任务系统以进行跨会话任务跟踪。

```json
{
  "sisyphus": {
    "tasks": {
      "enabled": false,
      "storage_path": ".sisyphus/tasks",
      "claude_code_compat": false
    }
  }
}
```

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `enabled` | `false` | 启用 Sisyphus 任务系统 |
| `storage_path` | `.sisyphus/tasks` | 存储路径（相对于项目根目录） |
| `claude_code_compat` | `false` | 启用 Claude Code 路径兼容模式 |

---

## 功能

### 技能

技能带来领域特定专业知识和嵌入的 MCP。

内置技能：`playwright`、`playwright-cli`、`agent-browser`、`dev-browser`、`git-master`、`frontend-ui-ux`

禁用内置技能：`{ "disabled_skills": ["playwright"] }`

#### 技能配置

```json
{
  "skills": {
    "sources": [
      { "path": "./my-skills", "recursive": true },
      "https://example.com/skill.yaml"
    ],
    "enable": ["my-skill"],
    "disable": ["other-skill"],
    "my-skill": {
      "description": "它做什么",
      "template": "自定义提示词模板",
      "from": "source-file.ts",
      "model": "custom/model",
      "agent": "custom-agent",
      "subtask": true,
      "argument-hint": "使用提示",
      "license": "MIT",
      "compatibility": ">= 3.0.0",
      "metadata": { "author": "您的姓名" },
      "allowed-tools": ["read", "bash"]
    }
  }
}
```

| `sources` 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `path` | - | 本地路径或远程 URL |
| `recursive` | `false` | 递归到子目录 |
| `glob` | - | 用于文件选择的 glob 模式 |

### 钩子

通过 `disabled_hooks` 禁用内置钩子：

```json
{ "disabled_hooks": ["comment-checker", "agent-usage-reminder"] }
```

可用钩子：`todo-continuation-enforcer`、`context-window-monitor`、`session-recovery`、`session-notification`、`comment-checker`、`grep-output-truncator`、`tool-output-truncator`、`directory-agents-injector`、`directory-readme-injector`、`empty-task-response-detector`、`think-mode`、`anthropic-context-window-limit-recovery`、`rules-injector`、`background-notification`、`auto-update-checker`、`startup-toast`、`keyword-detector`、`agent-usage-reminder`、`non-interactive-env`、`interactive-bash-session`、`compaction-context-injector`、`thinking-block-validator`、`claude-code-hooks`、`ralph-loop`、`preemptive-compaction`、`auto-slash-command`、`sisyphus-junior-notepad`、`no-sisyphus-gpt`、`start-work`、`runtime-fallback`

**注意：**

- `directory-agents-injector` — 在 OpenCode 1.1.37+ 上自动禁用（原生 AGENTS.md 支持）
- `no-sisyphus-gpt` — **不要禁用**。它在不兼容的 GPT 模型上阻止 Sisyphus，同时允许专用的 GPT-5.4 提示词路径。
- `startup-toast` 是 `auto-update-checker` 的子功能。只需将 `startup-toast` 添加到 `disabled_hooks` 即可禁用 toast。

### 命令

通过 `disabled_commands` 禁用内置命令：

```json
{ "disabled_commands": ["init-deep", "start-work"] }
```

可用命令：`init-deep`、`ralph-loop`、`ulw-loop`、`cancel-ralph`、`refactor`、`start-work`、`stop-continuation`、`handoff`

### 浏览器自动化

| 提供程序 | 接口 | 安装 |
| --- | --- | --- |
| `playwright`（默认） | MCP 工具 | 通过 npx 自动安装 |
| `agent-browser` | Bash CLI | `bun add -g agent-browser && agent-browser install` |

切换提供程序：

```json
{ "browser_automation_engine": { "provider": "agent-browser" } }
```

### Tmux 集成

在单独的 tmux 窗格中运行后台子代理。需要在运行 `opencode --port <port>` 的 tmux 内运行。

```json
{
  "tmux": {
    "enabled": true,
    "layout": "main-vertical",
    "main_pane_size": 60,
    "main_pane_min_width": 120,
    "agent_pane_min_width": 40
  }
}
```

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `enabled` | `false` | 启用 tmux 窗格生成 |
| `layout` | `main-vertical` | `main-vertical` / `main-horizontal` / `tiled` / `even-horizontal` / `even-vertical` |
| `main_pane_size` | `60` | 主窗格 %（20–80） |
| `main_pane_min_width` | `120` | 最小主窗格列 |
| `agent_pane_min_width` | `40` | 最小代理窗格列 |

### Git Master

配置 git 提交行为：

```json
{ "git_master": { "commit_footer": true, "include_co_authored_by": true } }
```

### 注释检查器

自定义注释质量检查器：

```json
{
  "comment_checker": {
    "custom_prompt": "您的消息。使用 {{comments}} 占位符。"
  }
}
```

### 通知

强制启用会话通知：

```json
{ "notification": { "force_enable": true } }
```

`force_enable`（`false`）——即使检测到外部通知插件也强制会话通知。

### MCP

内置 MCP（默认启用）：`websearch`（Exa AI）、`context7`（库文档）、`grep_app`（GitHub 搜索）。

```json
{ "disabled_mcps": ["websearch", "context7", "grep_app"] }
```

### LSP

配置语言服务器协议集成：

```json
{
  "lsp": {
    "typescript-language-server": {
      "command": ["typescript-language-server", "--stdio"],
      "extensions": [".ts", ".tsx"],
      "priority": 10,
      "env": { "NODE_OPTIONS": "--max-old-space-size=4096" },
      "initialization": {
        "preferences": { "includeInlayParameterNameHints": "all" }
      }
    },
    "pylsp": { "disabled": true }
  }
}
```

| 选项 | 类型 | 描述 |
| --- | --- | --- |
| `command` | 数组 | 启动 LSP 服务器的命令 |
| `extensions` | 数组 | 文件扩展名（例如，`[".ts"]`） |
| `priority` | 数字 | 多个服务器匹配时的优先级 |
| `env` | 对象 | 环境变量 |
| `initialization` | 对象 | 传递给服务器的初始化选项 |
| `disabled` | 布尔值 | 禁用此服务器 |

---

## 高级

### 运行时回退

在 API 错误时自动切换到备份模型。

**简单配置**（使用默认值启用/禁用）：

```json
{ "runtime_fallback": true }
{ "runtime_fallback": false }
```

**高级配置**（完全控制）：

```json
{
  "runtime_fallback": {
    "enabled": true,
    "retry_on_errors": [400, 429, 503, 529],
    "max_fallback_attempts": 3,
    "cooldown_seconds": 60,
    "timeout_seconds": 30,
    "notify_on_fallback": true
  }
}
```

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `enabled` | `false` | 启用运行时回退 |
| `retry_on_errors` | `[400,429,503,529]` | 触发回退的 HTTP 代码。还处理分类的提供程序密钥错误。 |
| `max_fallback_attempts` | `3` | 每会话最大回退尝试次数（1–20） |
| `cooldown_seconds` | `60` | 重试失败模型之前的秒数 |
| `timeout_seconds` | `30` | 强制下一次回退之前的秒数。**设置为 `0` 以禁用基于超时的升级和提供程序重试消息检测。** |
| `notify_on_fallback` | `true` | 模型切换时的 toast 通知 |

按代理或类别定义 `fallback_models`：

```json
{
  "agents": {
    "sisyphus": {
      "model": "anthropic/claude-opus-4-6",
      "fallback_models": ["openai/gpt-5.4", "google/gemini-3.1-pro"]
    }
  }
}
```

### Hashline 编辑

用使用 `LINE#ID` 引用的哈希锚定版本替换内置的 `Edit` 工具以防止陈旧行编辑。默认禁用。

```json
{ "hashline_edit": true }
```

启用时，两个配套钩子处于活动状态：`hashline-read-enhancer`（注释 Read 输出）和 `hashline-edit-diff-enhancer`（显示差异）。通过设置 `hashline_edit: true` 选择加入。如果需要，可通过 `disabled_hooks` 单独禁用配套钩子。

### 实验性

```json
{
  "experimental": {
    "truncate_all_tool_outputs": false,
    "aggressive_truncation": false,
    "auto_resume": false,
    "disable_omo_env": false,
    "task_system": false,
    "dynamic_context_pruning": {
      "enabled": false,
      "notification": "detailed",
      "turn_protection": { "enabled": true, "turns": 3 },
      "protected_tools": [
        "task",
        "todowrite",
        "todoread",
        "lsp_rename",
        "session_read",
        "session_write",
        "session_search"
      ],
      "strategies": {
        "deduplication": { "enabled": true },
        "supersede_writes": { "enabled": true, "aggressive": false },
        "purge_errors": { "enabled": true, "turns": 5 }
      }
    }
  }
}
```

| 选项 | 默认值 | 描述 |
| --- | --- | --- |
| `truncate_all_tool_outputs` | `false` | 截断所有工具输出（不仅仅是白名单） |
| `aggressive_truncation` | `false` | 在代币限制超出时激进地截断 |
| `auto_resume` | `false` | 思考块恢复后自动恢复 |
| `disable_omo_env` | `false` | 禁用自动注入的 `<omo-env>` 块（日期/时间/区域设置）。提高缓存命中率。 |
| `task_system` | `false` | 启用 Sisyphus 任务系统 |
| `dynamic_context_pruning.enabled` | `false` | 自动修剪旧工具输出以管理上下文窗口 |
| `dynamic_context_pruning.notification` | `detailed` | 修剪通知：`off` / `minimal` / `detailed` |
| `turn_protection.turns` | `3` | 最近轮次受到修剪保护（1–10） |
| `strategies.deduplication` | `true` | 删除重复的工具调用 |
| `strategies.supersede_writes` | `true` | 当稍后读取文件时修剪写入输入 |
| `strategies.supersede_writes.aggressive` | `false` | 如果任何后续读取存在，修剪任何写入 |
| `strategies.purge_errors.turns` | `5` | 修剪前错误工具输入的轮次 |

---

## 参考

### 环境变量

| 变量 | 描述 |
| --- | --- |
| `OPENCODE_CONFIG_DIR` | 覆盖 OpenCode 配置目录（适用于配置文件隔离） |

### 提供程序特定

#### Google 身份验证

安装 [`opencode-antigravity-auth`](https://github.com/NoeFabris/opencode-antigravity-auth) 用于 Google Gemini。提供多账户负载均衡、双重配额和基于变体的思考。

#### Ollama

**必须**禁用流式传输以避免 JSON 解析错误：

```json
{
  "agents": {
    "explore": { "model": "ollama/qwen3-coder", "stream": false }
  }
}
```

常见模型：`ollama/qwen3-coder`、`ollama/ministral-3:14b`、`ollama/lfm2.5-thinking`

请参阅 [Ollama 故障排除](../troubleshooting/ollama.zh-cn.md) 以了解 `JSON 解析错误：意外的 EOF` 问题。
