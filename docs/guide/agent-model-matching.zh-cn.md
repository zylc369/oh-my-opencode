# 代理-模型匹配指南

> **面向代理和用户**：为什么每个代理需要特定模型——以及如何在不破坏功能的情况下进行自定义。

## 核心洞察：模型是开发者

将 AI 模型视为团队中的开发者。每个都有不同的大脑、不同的个性、不同的优势。**模型不仅仅是"更聪明"或"更笨"。它的思维方式不同。** 给 Claude 和 GPT 相同的指令，它们会以根本不同的方式解释。

这不是错误。这是整个系统的基础。

Oh My OpenCode 为每个代理分配一个与其_工作风格_相匹配的模型——就像组建一个团队，每个人都处于适合其个性的角色。

### Sisyphus：善于交际的主管

Sisyphus 是认识每个人、去任何地方、并通过沟通和协调完成工作的开发者。与其他代理交谈，理解整个代码库的上下文，智能地委派工作，编码也很好。但深度的、纯粹的技术问题？他会有点困难。

**这就是为什么 Sisyphus 使用 Claude / Kimi / GLM。** 这些模型擅长于：

- 遵循复杂的多步指令（Sisyphus 的提示词约 1,100 行）
- 在许多工具调用中保持对话流程
- 理解细微的委派和编排模式
- 产生结构良好、可交流的输出

将 Sisyphus 与较旧的 GPT 模型一起使用就像将您最好的项目经理——那个协调每个人、主持站立会议、让整个团队对齐的人——单独放在一个房间里调试竞态条件。不合适。GPT-5.4 现在有专门的 Sisyphus 提示词路径，但 GPT 仍然不是编排器的默认推荐。

### Hephaestus：深度专家

Heph aestus 是整天待在房间里编码的开发者。不怎么说话。可能看起来社交尴尬。但给他们一个困难的技术问题，他们会在三小时后带着没人能找到的解决方案出现。

**这就是为什么 Hephaestus 使用 GPT-5.3 Codex。** Codex 正是为这个构建的：

- 深度的自主探索，无需手把手指导
- 跨复杂代码库的多文件推理
- 原则驱动执行（给出目标，而不是配方）
- 长时间独立工作

将 Hephaestus 与 GLM 或 Kimi 一起使用就像将您最能沟通、善于交际的开发者分配单独做深度技术工作。他们最终会完成，但不会发光——您会浪费正是使他们有价值的技能。

### 要点

每个代理的提示词都经过调整以匹配其模型的个性。**当您更改模型时，您会更改大脑——相同的指令会被完全不同地理解。** 模型匹配不是关于"更好"或"更差"。它是关于适配。

---

## Claude 和 GPT 思维方式的不同

这对于理解为什么一些代理支持两个模型系列而其他不支持很重要。

**Claude** 响应**机制驱动**的提示词——详细的检查清单、模板、分步程序。更多规则 = 更多合规。您可以编写一个 1,100 行的提示词，带有嵌套工作流程，Claude 会遵循每一步。

**GPT**（特别是 5.2+）响应**原则驱动**的提示词——简洁的原则、XML 结构、明确的决策标准。更多规则 = 更多矛盾表面 = 更多漂移。GPT 在您说明目标并让它弄清楚机制时效果最好。

真实示例：Prometheus 的 Claude 提示词约 1,100 行，跨 7 个文件。GPT 提示词用约 121 行中的 3 个原则实现相同的行为。结果相同，方法完全不同。

支持两个系列（Prometheus、Atlas）的代理在运行时自动检测您的模型并通过 `isGptModel()` 切换提示词。您不需要考虑它。

---

## 代理简介

### 沟通者 → Claude / Kimi / GLM

这些代理具有 Claude 优化的提示词——长、详细、机制驱动。它们需要可靠地遵循复杂的多层指令的模型。

| 代理 | 角色 | 回退链 | 注意 |
| --- | --- | --- | --- |
| **Sisyphus** | 主编排器 | Claude Opus → GLM 5 → Big Pickle | Claude 系列优先。GPT-5.4 有专门支持，但 Claude/Kimi/GLM 仍然是首选的适配。 |
| **Metis** | 计划差距分析器 | Claude Opus → GPT-5.4 → Gemini 3.1 Pro | Claude 首选，GPT 可接受的回退。 |

### 双提示词代理 → Claude 首选，GPT 支持

这些代理为 Claude 和 GPT 系列提供单独的提示词。它们在运行时检测您的模型并切换。

| 代理 | 角色 | 回退链 | 注意 |
| --- | --- | --- | --- |
| **Prometheus** | 战略规划者 | Claude Opus → GPT-5.4 → Gemini 3.1 Pro | 访谈模式规划。GPT 提示词紧凑且原则驱动。 |
| **Atlas** | 待办事项编排器 | Claude Sonnet 4.6 → GPT-5.4 | Claude 首选，GPT-5.4 作为当前回退路径。 |

### 深度专家 → GPT

这些代理是为 GPT 的原则驱动风格构建的。它们的提示词假设自主的、目标导向的执行。不要覆盖到 Claude。

| 代理 | 角色 | 回退链 | 注意 |
| --- | --- | --- | --- |
| **Heph aestus** | 自主深度工作者 | 仅 GPT-5.3 Codex | 没有回退。需要 GPT 访问。工匠。 |
| **Oracle** | 架构顾问 | GPT-5.4 → Gemini 3.1 Pro → Claude Opus | 只读高智商咨询。 |
| **Momus** | 无情审查者 | GPT-5.4 → Claude Opus → Gemini 3.1 Pro | 验证和计划审查。 |

### 实用运行者 → 速度优于智能

这些代理进行 grep、搜索和检索。它们有意使用可用的最快、最便宜的模型。**不要"升级"它们到 Opus**——那是雇用高级工程师来处理文件。

| 代理 | 角色 | 回退链 | 注意 |
| --- | --- | --- | --- |
| **Explore** | 快速代码库 grep | Grok Code Fast → MiniMax → Haiku → GPT-5-Nano | 速度就是一切。并发触发 10 个。 |
| **Librarian** | 文档/代码搜索 | Gemini Flash → MiniMax → Big Pickle | 文档检索不需要深度推理。 |
| **Multimodal Looker** | 视觉/屏幕截图 | GPT-5.3 Codex → K2P5 → Gemini Flash → GLM-4.6v | 使用第一个可用的多模态回退。 |

---

## 模型系列

### Claude 系列

善于沟通、遵循指令、结构化输出。最适合需要遵循复杂多步提示词的代理。

| 模型 | 优势 |
| --- | --- |
| **Claude Opus 4.6** | 最佳整体。对复杂提示词的最高合规。Sisyphus 默认。 |
| **Claude Sonnet 4.6** | 更快、更便宜。日常任务的良好平衡。 |
| **Claude Haiku 4.5** | 快速且便宜。快速任务和实用工作的良好选择。 |
| **Kimi K2.5** | 行为非常类似于 Claude。以更低成本的优秀全能选手。 |
| **GLM 5** | Claude 类行为。适用于编排任务的稳健选项。 |

### GPT 系列

原则驱动、明确推理、深度技术能力。最适合在复杂问题上自主工作的代理。

| 模型 | 优势 |
| --- | --- |
| **GPT-5.3 Codex** | 深度编码强项。自主探索。Heph aestus 必需。 |
| **GPT-5.4** | 高智商、战略推理。Oracle 默认。 |
| **GPT-5.4** | 强大的原则驱动推理。Momus 默认以及 Prometheus / Atlas 的关键回退。 |
| **GPT-5-Nano** | 超便宜、快速。简单实用任务的良好选择。 |

### 其他模型

| 模型 | 优势 |
| --- | --- |
| **Gemini 3.1 Pro** | 在视觉/前端任务上表现出色。不同的推理风格。`visual-engineering` 和 `artistry` 的默认。 |
| **Gemini 3 Flash** | 快速。文档搜索和轻任务的良好选择。 |
| **Grok Code Fast 1** | 极速代码 grep。Explore 代理的默认。 |
| **MiniMax M2.5** | 快速且智能。实用任务和搜索/检索的良好选择。 |

### 关于免费层回退

您可能会在源代码或日志中看到模型名称如 `kimi-k2.5-free`、`minimax-m2.5-free` 或 `big-pickle`（GLM 4.6）。这些是相同模型系列的免费层版本，通过 OpenCode Zen 提供商提供服务。它们作为较低优先级条目存在于回退链中。

您不需要配置它们。系统包括它们，以便当您没有每个付费订阅时优雅降级。如果您有付费版本，付费版本总是首选。

---

## 任务类别

当代理委派工作时，它们不会选择模型名称——它们选择一个**类别**。类别自动映射到正确的模型。

| 类别 | 何时使用 | 回退链 |
| --- | --- | --- |
| `visual-engineering` | 前端、UI、CSS、设计 | Gemini 3.1 Pro → GLM 5 → Claude Opus |
| `ultrabrain` | 需要最大推理 | GPT-5.3 Codex → Gemini 3.1 Pro → Claude Opus |
| `deep` | 深度编码、复杂逻辑 | GPT-5.3 Codex → Claude Opus → Gemini 3.1 Pro |
| `artistry` | 创意、新颖方法 | Gemini 3.1 Pro → Claude Opus → GPT-5.4 |
| `quick` | 简单、快速任务 | Claude Haiku → Gemini Flash → GPT-5-Nano |
| `unspecified-high` | 一般复杂工作 | GPT-5.4 → Claude Opus → GLM 5 → K2P5 |
| `unspecified-low` | 一般标准工作 | Claude Sonnet → GPT-5.3 Codex → Gemini Flash |
| `writing` | 文本、文档、散文 | Gemini Flash → Claude Sonnet |

请参阅[编排系统指南](./orchestration.zh-cn.md)以了解代理如何向类别分发任务。

---

## 自定义

### 示例配置

```jsonc
{
  "$schema": "https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/dev/assets/oh-my-opencode.schema.json",

  "agents": {
    // 主编排器：Claude Opus 或 Kimi K2.5 最好
    "sisyphus": {
      "model": "kimi-for-coding/k2p5",
      "ultrawork": { "model": "anthropic/claude-opus-4-6", "variant": "max" },
    },

    // 研究代理：更便宜的模型就可以了
    "librarian": { "model": "google/gemini-3-flash" },
    "explore": { "model": "github-copilot/grok-code-fast-1" },

    // 架构咨询：GPT 或 Claude Opus
    "oracle": { "model": "openai/gpt-5.4", "variant": "high" },

    // Prometheus 继承 sisyphus 模型；只需添加提示词指导
    "prometheus": {
      "prompt_append": "大量利用深度和快速代理，始终并行。",
    },
  },

  "categories": {
    "quick": { "model": "opencode/gpt-5-nano" },
    "unspecified-low": { "model": "anthropic/claude-sonnet-4-6" },
    "unspecified-high": { "model": "openai/gpt-5.4", "variant": "high" },
    "visual-engineering": {
      "model": "google/gemini-3.1-pro",
      "variant": "high",
    },
    "writing": { "model": "google/gemini-3-flash" },
  },

  // 限制昂贵的提供商；让便宜的提供商自由运行
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
}
```

运行 `opencode models` 查看可用模型，`opencode auth login` 以验证提供程序。

### 安全与危险覆盖

**安全**——相同的个性类型：

- Sisyphus：Opus → Sonnet、Kimi K2.5、GLM 5（所有善于沟通的模型）
- Prometheus：Opus → GPT-5.4（自动切换到 GPT 提示词）
- Atlas：Claude Sonnet 4.6 → GPT-5.4（自动切换到 GPT 提示词）

**危险**——个性不匹配：

- Sisyphus → 较旧的 GPT 模型：**仍然是一个糟糕的适配。GPT-5.4 是唯一的专用 GPT 提示词路径。**
- Hephaestus → Claude：**为 Codex 的自主风格构建。Claude 无法复制这个。**
- Explore → Opus：**巨大的成本浪费。Explore 需要速度，而不是智能。**
- Librarian → Opus：**相同。文档搜索不需要 Opus 级别的推理。**

### 模型解析工作原理

每个代理都有一个回退链。系统按优先级顺序尝试模型，直到通过您连接的提供程序找到一个可用的。您不需要每个模型配置提供程序——只需验证（`opencode auth login`），系统会弄清楚哪些模型可用以及在哪里。

```
代理请求 → 用户覆盖（如果配置）→ 回退链 → 系统默认
```

---

## 另见

- [安装指南](./installation.zh-cn.md) —— 设置和身份验证
- [编排系统指南](./orchestration.zh-cn.md) —— 代理如何向类别分发任务
- [配置参考](../reference/configuration.zh-cn.md) —— 完整配置选项
- [`src/shared/model-requirements.ts`](../../src/shared/model-requirements.ts) —— 回退链的真实来源
