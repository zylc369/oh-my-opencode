# Oh My OpenCode 是什么？

Oh My OpenCode 是 OpenCode 的多模型代理编排系统。它将单个 AI 代理转化为真正能够交付代码的协调开发团队。

不被 Claude 锁定。不被 OpenAI 锁定。不被任何人锁定。

只有更好的结果、更便宜的模型、真正的编排。

---

## 快速开始

### 安装

将其粘贴到您的 LLM 代理会话中：

```
按照以下说明安装和配置 oh-my-opencode：
https://raw.githubusercontent.com/code-yeongyu/oh-my-opencode/refs/heads/dev/docs/guide/installation.md
```

或者阅读完整的[安装指南](./installation.zh-cn.md)以进行手动设置、提供程序身份验证和故障排除。

### 您的第一个任务

安装完成后，只需输入：

```
ultrawork
```

就是这样。代理会自动处理一切——探索您的代码库、研究模式、实现功能、通过诊断验证。它会一直工作直到完成。

想要更多控制？按 **Tab** 进入 [Prometheus 模式](./orchestration.zh-cn.md) 进行基于访谈的规划，然后运行 `/start-work` 进行完整编排。

---

## 理念：打破束缚

我们曾经称之为"Claude Code 的增强版"。那是错误的。

这并不是要让 Claude Code 更好。而是要打破"一个模型、一个提供商、一种工作方式就足够"的观念。Anthropic 想要锁定您。OpenAI 想要锁定您。每个人都想要锁定您。

Oh My OpenCode 不玩这种游戏。它在模型之间编排，为每项工作选择合适的大脑。Claude 用于编排。GPT 用于深度推理。Gemini 用于前端。Haiku 用于快速任务。所有模型自动协作。

---

## 工作原理：代理编排

不是一个代理做所有事情，Oh My OpenCode 使用**根据任务类型相互委派的专用代理**。

**架构：**

```
用户请求
    ↓
[意图门] — 分类您的实际需求
    ↓
[Sisyphus] — 主编排器，规划和委派
    ↓
    ├─→ [Prometheus] — 战略规划（访谈模式）
    ├─→ [Atlas] — 待办事项编排和执行
    ├─→ [Oracle] — 架构咨询
    ├─→ [Librarian] — 文档/代码搜索
    ├─→ [Explore] — 快速代码库搜索
    └─→ [基于类别的代理] — 按任务类型专门化
```

当 Sisyphus 委派给子代理时，它不会选择模型名称。它选择一个**类别**——`visual-engineering`、`ultrabrain`、`quick`。类别会自动映射到正确的模型。您无需任何操作。

要深入了解代理如何协作，请参阅[编排系统指南](./orchestration.zh-cn.md)。

---

## 了解代理

### Sisyphus：纪律代理

以希腊神话命名。他每天都推着巨石。从不停止。从不放弃。

Sisyphus 是您的主要编排器。他规划工作、委派给专家，并通过激进的并行执行推动任务完成。他不会半途而废。他不会分心。他会完成工作。

**推荐的模型：**

- **Claude Opus 4.6** — 最佳整体体验。Sisyphus 是为 Claude 优化的提示词构建的。
- **Claude Sonnet 4.6** — 能力和成本的良好平衡。
- **Kimi K2.5** — 出色的 Claude 替代方案。许多用户完全使用此组合。
- **GLM 5** — 稳健的选项，特别是通过 Z.ai。

Sisyphus 在 Claude 系列模型、Kimi 和 GLM 上仍然效果最好。GPT-5.4 现在具有专门的提示词路径，但较旧的 GPT 模型仍然不合适，应该路由到 Hephaestus。

### Hephaestus：合法的工匠

这个名字是有意为之的讽刺。因为本项目，Anthropic 阻止了 OpenCode 使用他们的 API。因此团队构建了一个自主的 GPT 原生代理。

Hephaestus 运行在 GPT-5.3 Codex 上。给他一个目标，而不是配方。他会探索代码库、研究模式，并在无需手把手指导的情况下端到端地执行。他是合法的工匠，因为他是出于必要性诞生的，而不是特权。

当您需要深度架构推理、跨多个文件的复杂调试或跨领域知识综合时，使用 Hephaestus。当工作需要 GPT-5.3 Codex 的特定优势时，明确切换到他。

**为什么这比纯 Codex CLI 更好：**

- **多模型编排。** 纯 Codex 是单模型的。OmO 自动将不同的任务路由到不同的模型。GPT 用于深度推理。Gemini 用于前端。Haiku 用于速度。为每项工作选择合适的大脑。
- **后台代理。** 并发触发 5+ 个代理。Codex 根本无法做到这一点。当一个代理编写代码时，另一个研究模式，另一个检查文档。就像一个真正的开发团队。
- **类别系统。** 任务按意图路由，而不是按模型名称。`visual-engineering` 获得 Gemini。`ultrabrain` 获得 GPT-5.3 Codex。`quick` 获得 Haiku。无需手动调整。
- **累积智慧。** 子代理从之前的结果中学习。在任务 1 中发现的约定会传递给任务 5。早期犯的错误不会重复。系统在工作中会变得更聪明。

### Prometheus：战略规划者

Prometheus 像真正的工程师一样面试您。提出澄清问题。识别范围和模糊性。在接触任何代码行之前构建详细计划。

按 **Tab** 进入 Prometheus 模式，或从 Sisyphus 输入 `@plan "your task"`。

### Atlas：指挥家

Atlas 执行 Prometheus 计划。将任务分发给专门化的子代理。跨任务积累学习。独立验证完成情况。

运行 `/start-work` 在您的最新计划上激活 Atlas。

### Oracle：顾问

用于架构决策和复杂调试的只读高智商顾问。当面临不熟悉的模式、安全顾虑或多系统权衡时，咨询 Oracle。

### 支持阵容

- **Metis** — 差距分析器。在计划最终化之前发现 Prometheus 漏掉的内容。
- **Momus** — 无情的审查者。根据清晰度、可验证性和上下文标准验证计划。
- **Explore** — 快速代码库搜索。使用速度优化的模型进行模式发现。
- **Librarian** — 文档和开源代码搜索。随时了解库 API 和最佳实践。
- **Multimodal Looker** — 视觉和屏幕截图分析。

---

## 工作模式

### Ultrawork 模式：为懒人设计

输入 `ultrawork` 或只是 `ulw`。就是这样。

代理会处理一切。探索您的代码库。研究模式。实现功能。通过诊断验证。一直工作直到完成。

这是"直接做"的模式。全自动。您不需要深入思考，因为代理会为您深入思考。

### Prometheus 模式：为精确设计

按 **Tab** 进入 Prometheus 模式。

Prometheus 像真正的工程师一样面试您。提出澄清问题。识别范围和模糊性。在接触任何代码行之前构建详细计划。

然后运行 `/start-work`，Atlas 接管。任务被分发给专门化的子代理。每个完成都是独立验证的。学习跨任务累积。进度跨会话跟踪。

将 Prometheus 用于多日项目、关键的生产更改、复杂的重构，或者当您想要有记录的决策追踪时。

---

## 代理-模型匹配

不同的代理与不同的模型配合最佳。Oh My OpenCode 自动分配最佳模型，但您可以自定义一切。

### 默认配置

模型在安装时自动配置。交互式安装程序会询问您有哪些提供商，然后为每个代理和类别生成最佳模型分配。

在运行时，回退链确保即使您的首选提供商宕机，工作也能继续。每个代理都有一个提供商优先级链。系统按顺序尝试提供商，直到找到可用的模型。

### 自定义模型配置

您可以在配置中覆盖特定代理或类别：

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
  },

  "categories": {
    // 前端工作：Gemini 在视觉任务上占主导地位
    "visual-engineering": {
      "model": "google/gemini-3.1-pro",
      "variant": "high",
    },

    // 一般高投入工作
    "unspecified-high": { "model": "openai/gpt-5.4", "variant": "high" },

    // 快速任务：使用最便宜的模型
    "quick": { "model": "anthropic/claude-haiku-4-5" },

    // 深度推理：GPT-5.3-codex
    "ultrabrain": { "model": "openai/gpt-5.3-codex", "variant": "xhigh" },
  },
}
```

### 模型系列

**Claude 类模型**（遵循指令、结构化输出）：

- Claude Opus 4.6、Claude Sonnet 4.6、Claude Haiku 4.5
- Kimi K2.5 — 行为非常类似于 Claude
- GLM 5 — Claude 类行为，适用于广泛任务

**GPT 模型**（明确推理、原则驱动）：

- GPT-5.3-codex — 深度编码强项，Hephaestus 必需
- GPT-5.4 — 高智商，Oracle 默认
- GPT-5-Nano — 超便宜，快速实用任务

**不同行为模型**：

- Gemini 3 Pro — 在视觉/前端任务上表现出色
- MiniMax M2.5 — 快速且智能，适用于实用任务
- Grok Code Fast 1 — 针对代码 grep/搜索进行了优化

请参阅[代理-模型匹配指南](./agent-model-matching.zh-cn.md)，了解哪个模型最适合每个代理、安全与危险覆盖以及提供商优先级链的完整详细信息。

---

## 为什么比纯 Claude Code 更好

Claude Code 很好。但它是一个代理运行一个模型，独自做所有事情。

Oh My OpenCode 将其转化为协调的团队：

**并行执行。** Claude Code 一次处理一件事。OmO 在后台并发触发代理——研究、实施和验证同时发生。就像有 5 个工程师而不是 1 个。

**哈希锚定编辑。** Claude Code 的编辑工具在模型无法准确重现行时失败。OmO 的 `LINE#ID` 内容哈希在应用之前验证每次编辑。Grok Code Fast 1 的成功率仅因此项更改就从 6.7% 提高到 68.3%。

**意图门。** Claude Code 获取您的提示词并运行。OmO 首先对您的真实意图进行分类——研究、实施、调查、修复——然后相应路由。更少的误解，更好的结果。

**LSP + AST 工具。** 工作区级别的重命名、转到定义、查找引用、预构建诊断、AST 感知的代码重写。纯 Claude Code 没有的 IDE 精度。

**带嵌入 MCP 的技能。** 每个技能都带来自己的 MCP 服务器，范围限定到任务。上下文窗口保持干净，而不是因为每个工具而膨胀。

**纪律执行。** 待办事项执行器将空闲代理拉回工作。注释检查器剥离 AI 垃圾。Ralph 循环一直持续到 100% 完成。系统不会让代理懈怠。

**根本优势。** 模型有不同的性情。Claude 思考深入。GPT 推理架构。Gemini 可视化。Haiku 快速移动。单模型工具强制您为所有任务选择一种个性。Oh My OpenCode 利用它们全部，按任务类型路由。这不是临时技巧——这是随着模型进一步专业化唯一有意义的架构。多模型编排和单模型限制之间的差距每个月都在扩大。我们押注在这个未来。

---

## 意图门

在根据任何请求行动之前，Sisyphus 会对您的真实意图进行分类。

您是在询问研究？实施？调查？修复？意图门弄清楚您实际上想要什么，而不仅仅是您输入的字面意思。这意味着代理理解上下文、细微差别以及您请求背后的真实目标。

Claude Code 没有这个。它获取您的提示词并运行。Oh My OpenCode 先思考，然后行动。

---

## 接下来是什么

- **[安装指南](./installation.zh-cn.md)** — 完整的设置说明、提供商身份验证和故障排除
- **[编排系统指南](./orchestration.zh-cn.md)** — 深入了解代理协作、使用 Prometheus 规划和使用 Atlas 执行
- **[代理-模型匹配指南](./agent-model-matching.zh-cn.md)** — 哪个模型最适合每个代理以及如何自定义
- **[配置参考](../reference/configuration.zh-cn.md)** — 带有示例的完整配置选项
- **[功能参考](../reference/features.zh-cn.md)** — 完整的功能文档
- **[宣言](../manifesto.zh-cn.md)** — 项目背后的理念

---

**准备开始了吗？** 输入 `ultrawork`，看看协调的 AI 团队能做什么。
