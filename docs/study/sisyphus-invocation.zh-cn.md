# Sisyphus 如何调用其他 Agent

## 结论

Sisyphus 通过**两种工具**和**两种执行模式**调用其他 agent:

### 工具
1. **`task`** (`createDelegateTask`): 基于 category 的委托,支持技能加载和模型选择
2. **`call_omo_agent`** (`createCallOmoAgent`): 直接 agent 调用(仅限 explore/librarian)

### 执行模式
1. **同步** (`run_in_background=false`): 创建会话 → 发送提示 → 轮询直到空闲 → 返回结果
2. **后台** (`run_in_background=true`): 通过 BackgroundManager 启动 → 异步轮询 → 系统通知

### 调用路径
```
Sisyphus → task(category="quick") → Sisyphus-Junior (使用 category 配置生成)
Sisyphus → task(subagent_type="oracle") → Oracle (直接 agent 调用)
Sisyphus → call_omo_agent(subagent_type="explore") → Explore (直接,无技能)
```

---

## 详细说明

### 1. 两种委托工具

#### 1.1 `task` 工具 — 完整委托

`task` 工具是主要的委托机制。它支持:

- **基于 category 的委托**: 生成 Sisyphus-Junior,使用 category 特定的模型和提示
- **直接 subagent 调用**: 按名称调用特定 agent
- **技能加载**: 将技能内容注入 agent 上下文
- **会话延续**: 使用 `session_id` 恢复现有会话

```typescript
// src/tools/delegate-task/tools.ts
export function createDelegateTask(options: DelegateTaskToolOptions): ToolDefinition {
  return tool({
    description: `Spawn agent task with category-based or direct agent selection.
    
    REQUIRED: Provide ONE of:
    - category: For task delegation (uses Sisyphus-Junior with category-optimized model)
    - subagent_type: For direct agent invocation (explore, librarian, oracle, etc.)
    
    Available categories: visual-engineering, ultrabrain, deep, artistry, quick, 
                          unspecified-low, unspecified-high, writing
    `,
    args: {
      load_skills: tool.schema.array(tool.schema.string())
        .describe("Skill names to inject. REQUIRED - pass [] if no skills needed."),
      description: tool.schema.string().describe("Short task description (3-5 words)"),
      prompt: tool.schema.string().describe("Full detailed prompt for the agent"),
      run_in_background: tool.schema.boolean()
        .describe("true=async (returns task_id), false=sync (waits). Default: false"),
      category: tool.schema.string().optional()
        .describe("REQUIRED if subagent_type not provided."),
      subagent_type: tool.schema.string().optional()
        .describe("REQUIRED if category not provided."),
      session_id: tool.schema.string().optional()
        .describe("Existing Task session to continue"),
      command: tool.schema.string().optional()
        .describe("The command that triggered this task"),
    },
    async execute(args: DelegateTaskArgs, toolContext) {
      // ... execution logic
    },
  })
}
```

#### 1.2 `call_omo_agent` 工具 — 直接调用

更简单的工具,用于快速 agent 调用,无需 category/技能系统:

```typescript
// src/tools/call-omo-agent/tools.ts
export function createCallOmoAgent(
  ctx: PluginInput,
  backgroundManager: BackgroundManager,
  disabledAgents: string[] = []
): ToolDefinition {
  return tool({
    description: `Invoke specialized agents directly...`,
    args: {
      description: tool.schema.string().describe("A short (3-5 words) description"),
      prompt: tool.schema.string().describe("The task for the agent to perform"),
      subagent_type: tool.schema.string()
        .describe("The type of specialized agent (explore or librarian only)"),
      run_in_background: tool.schema.boolean()
        .describe("REQUIRED. true: async, false: sync"),
      session_id: tool.schema.string().describe("Existing session to continue").optional(),
    },
    async execute(args: CallOmoAgentArgs, toolContext) {
      // Only explore and librarian allowed
      if (!ALLOWED_AGENTS.some(name => name.toLowerCase() === args.subagent_type.toLowerCase())) {
        return `Error: Invalid agent type. Only ${ALLOWED_AGENTS.join(", ")} are allowed.`
      }
      // ... execution logic
    },
  })
}
```

### 2. Category vs Subagent 解析

#### 2.1 Category 解析 → Sisyphus-Junior

当提供 `category` 时,工具会:
1. 解析 category 配置(模型、变体、prompt_append)
2. 设置 `agentToUse = "sisyphus-junior"`
3. 构建 system 内容,包含 category 提示追加

```typescript
// src/tools/delegate-task/tools.ts (lines 112-120)
if (args.category) {
  if (args.subagent_type && args.subagent_type !== SISYPHUS_JUNIOR_AGENT) {
    log("[task] category provided - overriding subagent_type to sisyphus-junior", {
      category: args.category,
      subagent_type: args.subagent_type,
    })
  }
  args.subagent_type = SISYPHUS_JUNIOR_AGENT
}
```

Category 解析详情:

```typescript
// src/tools/delegate-task/category-resolver.ts
export async function resolveCategoryExecution(
  args: DelegateTaskArgs,
  executorCtx: ExecutorContext,
  inheritedModel: string | undefined,
  systemDefaultModel: string | undefined
): Promise<CategoryResolutionResult> {
  // 1. Get available models
  const availableModels = await getAvailableModelsForDelegateTask(client)
  
  // 2. Resolve category config
  const resolved = resolveCategoryConfig(categoryName, {
    userCategories,
    inheritedModel,
    systemDefaultModel,
    availableModels,
  })
  
  // 3. Model resolution with fallback chain
  const resolution = resolveModelForDelegateTask({
    userModel: explicitCategoryModel ?? overrideModel,
    categoryDefaultModel: resolved.model,
    fallbackChain: requirement.fallbackChain,
    availableModels,
    systemDefaultModel,
  })
  
  // 4. Build category model config
  categoryModel = { providerID, modelID, variant }
  
  return {
    agentToUse: SISYPHUS_JUNIOR_AGENT,  // ← Always Sisyphus-Junior
    categoryModel,
    categoryPromptAppend,
    modelInfo,
    actualModel,
    fallbackChain,
  }
}
```

#### 2.2 Subagent 解析 → 直接 Agent

当提供 `subagent_type`(不带 category)时,工具会:
1. 验证 agent 存在且可调用(非 primary)
2. 从其配置解析 agent 的模型
3. 从 `AGENT_MODEL_REQUIREMENTS` 获取回退链

```typescript
// src/tools/delegate-task/subagent-resolver.ts
export async function resolveSubagentExecution(
  args: DelegateTaskArgs,
  executorCtx: ExecutorContext,
  parentAgent: string | undefined,
  categoryExamples: string
): Promise<{...}> {
  const agentName = args.subagent_type.trim()
  
  // Cannot call Sisyphus-Junior directly
  if (agentName.toLowerCase() === SISYPHUS_JUNIOR_AGENT.toLowerCase()) {
    return {
      error: `Cannot use subagent_type="sisyphus-junior" directly. 
              Use category parameter instead.`
    }
  }
  
  // Plan family blocking
  if (isPlanFamily(agentName) && isPlanFamily(parentAgent)) {
    return {
      error: `You are a plan-family agent. You cannot delegate to other plan-family agents.`
    }
  }
  
  // Get callable agents from OpenCode
  const agents = await client.app.agents()
  const callableAgents = agents.filter(a => a.mode !== "primary")
  
  // Match agent
  const matchedAgent = callableAgents.find(
    a => a.name.toLowerCase() === agentName.toLowerCase()
  )
  
  if (!matchedAgent) {
    return { error: `Unknown agent: "${agentName}". Available: ${availableAgents}` }
  }
  
  // Resolve model
  const agentRequirement = AGENT_MODEL_REQUIREMENTS[agentConfigKey]
  fallbackChain = agentRequirement?.fallbackChain
  
  // ... model resolution
  
  return { agentToUse: matchedAgent.name, categoryModel, fallbackChain }
}
```

### 3. 同步 vs 后台执行

#### 3.1 同步执行流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        executeSyncTask()                                 │
│                (src/tools/delegate-task/sync-task.ts)                    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 1: Create Session                                                  │
│                                                                          │
│  createSyncSession(client, {                                            │
│    parentSessionID: parentContext.sessionID,                            │
│    agentToUse,                                                          │
│    description: args.description,                                       │
│    defaultDirectory: directory,                                         │
│  })                                                                     │
│  → Returns { ok: true, sessionID: "ses_xxx" }                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 2: Register Session                                                │
│                                                                          │
│  subagentSessions.add(sessionID)                                        │
│  syncSubagentSessions.add(sessionID)                                    │
│  setSessionAgent(sessionID, agentToUse)                                 │
│  setSessionFallbackChain(sessionID, fallbackChain)                      │
│  SessionCategoryRegistry.register(sessionID, args.category)             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 3: Show Toast                                                      │
│                                                                          │
│  toastManager.addTask({                                                 │
│    id: taskId,                                                          │
│    sessionID,                                                           │
│    description,                                                         │
│    agent: agentToUse,                                                   │
│    isBackground: false,                                                 │
│    category,                                                            │
│    skills,                                                              │
│    modelInfo,                                                           │
│  })                                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 4: Send Prompt                                                     │
│                                                                          │
│  sendSyncPrompt(client, {                                               │
│    sessionID,                                                           │
│    agentToUse,                                                          │
│    args,                                                                │
│    systemContent,     // Built from skills + category prompt_append     │
│    categoryModel,                                                       │
│  })                                                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 5: Poll Until Idle                                                 │
│                                                                          │
│  pollSyncSession(ctx, client, {                                         │
│    sessionID,                                                           │
│    agentToUse,                                                          │
│  }, syncPollTimeoutMs)                                                  │
│                                                                          │
│  // Loops: session.status !== "idle" → sleep → check again             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 6: Fetch Result                                                    │
│                                                                          │
│  fetchSyncResult(client, sessionID)                                     │
│  → Returns last assistant message content                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 7: Return                                                          │
│                                                                          │
│  return `Task completed in ${duration}.                                 │
│                                                                          │
│  Agent: ${agentToUse}${category ? ` (category: ${category})` : ""}      │
│  ---                                                                     │
│  ${result.textContent}                                                   │
│                                                                          │
│  <task_metadata>                                                         │
│  session_id: ${sessionID}                                               │
│  </task_metadata>`                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 3.2 后台执行流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      executeBackgroundTask()                             │
│              (src/tools/delegate-task/background-task.ts)                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 1: Launch via BackgroundManager                                    │
│                                                                          │
│  const task = await manager.launch({                                    │
│    description: args.description,                                       │
│    prompt: args.prompt,                                                 │
│    agent: agentToUse,                                                   │
│    parentSessionID: parentContext.sessionID,                            │
│    parentMessageID: parentContext.messageID,                            │
│    parentModel: parentContext.model,                                    │
│    parentAgent: parentContext.agent,                                    │
│    parentTools: getSessionTools(parentContext.sessionID),               │
│    model: categoryModel,                                                │
│    fallbackChain,                                                       │
│    skills: args.load_skills,                                            │
│    skillContent: systemContent,                                         │
│    category: args.category,                                             │
│  })                                                                     │
│  → Returns { id: "bg_xxx", sessionID: "ses_xxx", status: "pending" }    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 2: Wait for Session Creation                                       │
│                                                                          │
│  // BackgroundManager.launch() returns immediately before session exists│
│  while (!sessionId && Date.now() - waitStart < WAIT_FOR_SESSION_TIMEOUT) {│
│    await sleep(WAIT_FOR_SESSION_INTERVAL)                               │
│    const updated = manager.getTask(task.id)                             │
│    sessionId = updated?.sessionID                                       │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 3: Register Category                                               │
│                                                                          │
│  if (args.category && sessionId) {                                      │
│    SessionCategoryRegistry.register(sessionId, args.category)           │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 4: Return Immediately                                              │
│                                                                          │
│  return `Background task launched.                                      │
│                                                                          │
│  Background Task ID: ${task.id}                                         │
│  Description: ${task.description}                                       │
│  Agent: ${task.agent}${category ? ` (category: ${category})` : ""}      │
│  Status: ${task.status}                                                 │
│                                                                          │
│  System notifies on completion.                                         │
│  Use background_output with task_id="${task.id}" to check.`             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (async, in background)
┌─────────────────────────────────────────────────────────────────────────┐
│  Background: Async Polling                                               │
│                                                                          │
│  // BackgroundManager polls session status                              │
│  // When complete, system notification sent to parent session           │
│  // Parent can use background_output to fetch results                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4. System 内容构建

注入到 agent 的 system 内容包括:

```typescript
// src/tools/delegate-task/prompt-builder.ts
export function buildSystemContent(input: BuildSystemContentInput): string | undefined {
  const parts: string[] = []
  
  // 1. Skill content (if load_skills provided)
  if (skillContent) {
    parts.push(skillContent)
  }
  
  // 2. Category prompt append (if category provided)
  if (categoryPromptAppend) {
    parts.push(categoryPromptAppend)
  }
  
  // 3. Available categories (for Sisyphus-Junior context)
  if (availableCategories.length > 0) {
    parts.push(buildCategoriesSection(availableCategories))
  }
  
  // 4. Available skills (for Sisyphus-Junior context)
  if (availableSkills.length > 0) {
    parts.push(buildSkillsSection(availableSkills))
  }
  
  return parts.length > 0 ? parts.join("\n\n") : undefined
}
```

Category 提示追加定义在 `constants.ts`:

```typescript
// src/tools/delegate-task/constants.ts
export const CATEGORY_PROMPT_APPENDS: Record<string, string> = {
  "visual-engineering": `<Category_Context>
You are working on VISUAL/UI tasks.
Design-first mindset:
- Bold aesthetic choices over safe defaults
- Unexpected layouts, asymmetry, grid-breaking elements
...
</Category_Context>`,
  
  "deep": `<Category_Context>
You are working on GOAL-ORIENTED AUTONOMOUS tasks.
**CRITICAL - AUTONOMOUS EXECUTION MINDSET**:
You are NOT an interactive assistant. You are an autonomous problem-solver.
...
</Category_Context>`,
  
  "quick": `<Category_Context>
You are working on SMALL / QUICK tasks.
Efficient execution mindset:
- Fast, focused, minimal overhead
...
</Category_Context>`,
  
  // ... more categories
}
```

---

## 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Sisyphus Agent                                 │
│                        (Main Orchestrator)                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
        ┌──────────────────┐            ┌──────────────────┐
        │   task tool      │            │ call_omo_agent   │
        │ (full delegation)│            │ (direct call)    │
        └──────────────────┘            └──────────────────┘
                    │                               │
                    │                               │
          ┌─────────┴─────────┐                     │
          ▼                   ▼                     ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ category param?  │ │ subagent_type?   │ │ subagent_type    │
│                  │ │                  │ │ (explore/        │
│                  │ │                  │ │  librarian only) │
└──────────────────┘ └──────────────────┘ └──────────────────┘
          │                   │                     │
          ▼                   ▼                     │
┌──────────────────┐ ┌──────────────────┐          │
│ resolveCategory  │ │ resolveSubagent  │          │
│ Execution()      │ │ Execution()      │          │
│                  │ │                  │          │
│ → Sisyphus-Junior│ │ → Named agent    │          │
│ → Category model │ │ → Agent's model  │          │
│ → Category prompt│ → Fallback chain  │          │
└──────────────────┘ └──────────────────┘          │
          │                   │                     │
          └─────────┬─────────┘                     │
                    ▼                               │
┌──────────────────────────────────┐                │
│     Build System Content         │                │
│                                  │                │
│  1. Resolve skill content        │                │
│  2. Add category prompt_append   │                │
│  3. Add available categories     │                │
│  4. Add available skills         │                │
└──────────────────────────────────┘                │
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
                    ┌───────────────────────────────┐
                    │     run_in_background?        │
                    └───────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │ false                         │ true
                    ▼                               ▼
        ┌──────────────────┐            ┌──────────────────┐
        │  executeSyncTask │            │executeBackground │
        │                  │            │      Task        │
        └──────────────────┘            └──────────────────┘
                    │                               │
                    ▼                               ▼
        ┌──────────────────┐            ┌──────────────────┐
        │ 1. Create session│            │ 1. manager.launch│
        │ 2. Register      │            │ 2. Wait for      │
        │    session       │            │    session       │
        │ 3. Show toast    │            │ 3. Return task_id│
        │ 4. Send prompt   │            │                  │
        │ 5. Poll until    │            │ (async polling   │
        │    idle          │            │  in background)  │
        │ 6. Fetch result  │            │                  │
        │ 7. Return result │            │ System notifies  │
        └──────────────────┘            │ on completion    │
                    │                   └──────────────────┘
                    ▼
        ┌──────────────────┐
        │ Return to        │
        │ Sisyphus with    │
        │ session_id for   │
        │ continuation     │
        └──────────────────┘
```

---

## 关键文件参考

| 文件 | 用途 |
|------|------|
| `src/tools/delegate-task/tools.ts` | `createDelegateTask()` 工厂 |
| `src/tools/delegate-task/executor.ts` | 导出执行器函数 |
| `src/tools/delegate-task/category-resolver.ts` | Category → Sisyphus-Junior 解析 |
| `src/tools/delegate-task/subagent-resolver.ts` | 直接 agent 解析 |
| `src/tools/delegate-task/sync-task.ts` | 同步执行流程 |
| `src/tools/delegate-task/background-task.ts` | 后台执行流程 |
| `src/tools/delegate-task/prompt-builder.ts` | 构建 system 内容 |
| `src/tools/delegate-task/constants.ts` | Categories、提示追加 |
| `src/tools/call-omo-agent/tools.ts` | `createCallOmoAgent()` 工厂 |
| `src/features/background-agent/manager.ts` | `BackgroundManager.launch()` |

---

## 使用示例

### 基于 category 的委托(Sisyphus-Junior)

```
task(
  category="visual-engineering",
  load_skills=["frontend-ui-ux"],
  description="Redesign landing page",
  prompt="Redesign the landing page with a modern, bold aesthetic...",
  run_in_background=false
)
→ 使用 gemini-3.1-pro(高变体)生成 Sisyphus-Junior
→ 注入 frontend-ui-ux 技能内容
→ 添加 visual-engineering category 提示
→ 同步返回结果
```

### 直接 subagent 调用

```
task(
  subagent_type="oracle",
  load_skills=[],
  description="Review architecture",
  prompt="Review the authentication flow architecture...",
  run_in_background=false
)
→ 直接调用 Oracle agent
→ 使用 Oracle 的模型(gpt-5.2)
→ 无技能注入
→ 同步返回结果
```

### 后台探索

```
call_omo_agent(
  subagent_type="explore",
  description="Find auth patterns",
  prompt="Search for authentication patterns in the codebase...",
  run_in_background=true
)
→ 后台调用 Explore agent
→ 立即返回 task_id
→ 完成时系统通知
→ 使用 background_output 获取结果
```

### 会话延续

```
task(
  category="deep",
  load_skills=[],
  description="Continue implementation",
  prompt="The previous attempt failed because...",
  run_in_background=false,
  session_id="ses_abc123"
)
→ 恢复现有会话
→ 保留完整上下文
→ 节省 tokens,保持连续性
```
