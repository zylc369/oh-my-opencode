# oh-my-opencode 如何创建 Agent

## 结论

oh-my-opencode 通过**工厂模式与组合模式**创建 agent:

1. **工厂函数**: 每个 agent 都有一个 `createXXXAgent(model)` 工厂,返回 `AgentConfig`
2. **通过 `buildAgent()` 组合**: 将工厂输出与 categories、skills 和覆盖配置组合
3. **注册表模式**: `createBuiltinAgents()` 将所有 agent 工厂收集到注册表中
4. **条件创建**: `maybeCreateXXXConfig()` 函数在实例化前应用模型可用性检查
5. **多源合并**: 最终配置合并 built-in + user + project + plugin agents

核心洞察:**Agent 不是类,而是通过将工厂函数与运行时上下文组合构建的配置对象。**

---

## 详细说明

### 1. Agent 工厂模式

每个 agent 都定义为遵循此签名的工厂函数:

```typescript
// src/agents/types.ts
export type AgentFactory = ((model: string) => AgentConfig) & {
  mode: AgentMode  // "primary" | "subagent" | "all"
}
```

工厂示例:

```typescript
// src/agents/oracle.ts
export const createOracleAgent: AgentFactory = (model: string) => ({
  instructions: `You are Oracle, a read-only consultation agent...`,
  model,
  temperature: 0.1,
  // ... additional config
})
createOracleAgent.mode = "subagent"
```

`mode` 属性决定 UI 行为:
- **primary**: 尊重用户在 UI 中选择的模型(sisyphus、atlas)
- **subagent**: 使用自己的回退链,忽略 UI 选择(oracle、explore 等)
- **all**: 在两种上下文中都可用(Sisyphus-Junior)

### 2. Agent 注册表 (`agentSources`)

所有工厂都注册在单个对象中:

```typescript
// src/agents/builtin-agents.ts
const agentSources: Record<BuiltinAgentName, AgentSource> = {
  sisyphus: createSisyphusAgent,
  hephaestus: createHephaestusAgent,
  oracle: createOracleAgent,
  librarian: createLibrarianAgent,
  explore: createExploreAgent,
  "multimodal-looker": createMultimodalLookerAgent,
  metis: createMetisAgent,
  momus: createMomusAgent,
  atlas: createAtlasAgent as AgentFactory,
}
```

### 3. 使用 `buildAgent()` 组合

`buildAgent()` 函数将工厂输出与额外层组合:

```typescript
// src/agents/agent-builder.ts
export function buildAgent(
  source: AgentSource,        // Factory or raw config
  model: string,
  categories?: CategoriesConfig,
  gitMasterConfig?: GitMasterConfig,
  browserProvider?: BrowserAutomationProvider,
  disabledSkills?: Set<string>
): AgentConfig {
  // 1. Call factory if it's a function
  const base = isFactory(source) ? source(model) : { ...source }
  
  // 2. Apply category config if agent has a category
  if (agentWithCategory.category) {
    const categoryConfig = categoryConfigs[agentWithCategory.category]
    if (categoryConfig) {
      if (!base.model) base.model = categoryConfig.model
      if (base.temperature === undefined) {
        base.temperature = categoryConfig.temperature
      }
      if (base.variant === undefined) {
        base.variant = categoryConfig.variant
      }
    }
  }
  
  // 3. Resolve and inject skills
  if (agentWithCategory.skills?.length) {
    const { resolved } = resolveMultipleSkills(agentWithCategory.skills, ...)
    if (resolved.size > 0) {
      const skillContent = Array.from(resolved.values()).join("\n\n")
      base.prompt = skillContent + (base.prompt ? "\n\n" + base.prompt : "")
    }
  }
  
  return base
}
```

### 4. 使用 `maybeCreateXXXConfig()` 条件创建

特殊 agent(sisyphus、hephaestus、atlas)使用条件工厂检查:

1. agent 是否被禁用?
2. 所需模型是否可用?
3. 回退模型是否可用?

```typescript
// src/agents/builtin-agents/sisyphus-agent.ts
export function maybeCreateSisyphusConfig(input: {...}): AgentConfig | undefined {
  const {
    disabledAgents,
    agentOverrides,
    uiSelectedModel,
    availableModels,
    systemDefaultModel,
    isFirstRunNoCache,
    // ... other params
  } = input

  // 1. Check if disabled
  if (disabledAgents.includes("sisyphus")) return undefined
  
  // 2. Check model availability
  const meetsRequirement = isAnyFallbackModelAvailable(
    sisyphusRequirement.fallbackChain,
    availableModels
  )
  if (!meetsRequirement && !isFirstRunNoCache) return undefined
  
  // 3. Resolve model
  let resolution = applyModelResolution({
    uiSelectedModel,
    userModel: sisyphusOverride?.model,
    requirement: sisyphusRequirement,
    availableModels,
    systemDefaultModel,
  })
  
  // 4. Build config
  let config = createSisyphusAgent(
    resolution.model,
    availableAgents,
    availableSkills,
    availableCategories,
    useTaskSystem
  )
  
  // 5. Apply overrides
  config = applyOverrides(config, sisyphusOverride, mergedCategories, directory)
  
  return config
}
```

### 5. 主编排器: `createBuiltinAgents()`

创建所有 agent 的主函数:

```typescript
// src/agents/builtin-agents.ts
export async function createBuiltinAgents(
  disabledAgents: string[] = [],
  agentOverrides: AgentOverrides = {},
  directory?: string,
  systemDefaultModel?: string,
  categories?: CategoriesConfig,
  gitMasterConfig?: GitMasterConfig,
  discoveredSkills: LoadedSkill[] = [],
  customAgentSummaries?: unknown,
  browserProvider?: BrowserAutomationProvider,
  uiSelectedModel?: string,
  disabledSkills?: Set<string>,
  useTaskSystem = false,
  disableOmoEnv = false
): Promise<Record<string, AgentConfig>> {
  const result: Record<string, AgentConfig> = {}
  
  // 1. Fetch available models
  const availableModels = await fetchAvailableModels(...)
  
  // 2. Merge categories
  const mergedCategories = mergeCategories(categories)
  
  // 3. Build available skills
  const availableSkills = buildAvailableSkills(...)
  
  // 4. Collect general agents (oracle, librarian, explore, etc.)
  const { pendingAgentConfigs, availableAgents } = collectPendingBuiltinAgents({
    agentSources,
    agentMetadata,
    disabledAgents,
    agentOverrides,
    // ...other params
  })
  
  // 5. Create special agents in order
  const sisyphusConfig = maybeCreateSisyphusConfig({...})
  if (sisyphusConfig) result["sisyphus"] = sisyphusConfig
  
  const hephaestusConfig = maybeCreateHephaestusConfig({...})
  if (hephaestusConfig) result["hephaestus"] = hephaestusConfig
  
  // 6. Add pending agents
  for (const [name, config] of pendingAgentConfigs) {
    result[name] = config
  }
  
  // 7. Add atlas last
  const atlasConfig = maybeCreateAtlasConfig({...})
  if (atlasConfig) result["atlas"] = atlasConfig
  
  return result
}
```

### 6. 模型解析链

创建 agent 时,模型解析遵循此优先级:

```
1. User override (agentOverrides[agentName].model)
2. UI selected model (for primary agents only)
3. Category default model (if agent has category)
4. Agent's fallback chain (from AGENT_MODEL_REQUIREMENTS)
5. System default model
```

```typescript
// src/agents/builtin-agents/model-resolution.ts
export function applyModelResolution(input: {...}): { model: string; variant?: string } | undefined {
  const { uiSelectedModel, userModel, requirement, availableModels, systemDefaultModel } = input
  
  // Priority 1: User override
  if (userModel && isModelAvailable(userModel, availableModels)) {
    return { model: userModel }
  }
  
  // Priority 2: UI selected (for primary agents)
  if (uiSelectedModel && isModelAvailable(uiSelectedModel, availableModels)) {
    return { model: uiSelectedModel }
  }
  
  // Priority 3: Fallback chain
  if (requirement?.fallbackChain) {
    for (const entry of requirement.fallbackChain) {
      if (isModelAvailable(entry.model, availableModels)) {
        return { model: entry.model, variant: entry.variant }
      }
    }
  }
  
  // Priority 4: System default
  if (systemDefaultModel && isModelAvailable(systemDefaultModel, availableModels)) {
    return { model: systemDefaultModel }
  }
  
  return undefined
}
```

---

## 流程图

### Agent 创建流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Plugin Initialization                             │
│                     (src/plugin-handlers/agent-config-handler.ts)       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     createBuiltinAgents()                                │
│                     (src/agents/builtin-agents.ts)                       │
│                                                                          │
│  1. Fetch available models                                               │
│  2. Merge user + default categories                                      │
│  3. Build available skills list                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
│ maybeCreate      │    │ collectPending   │    │ maybeCreate      │
│ SisyphusConfig() │    │ BuiltinAgents()  │    │ AtlasConfig()    │
└──────────────────┘    └──────────────────┘    └──────────────────┘
          │                         │                         │
          │                         ▼                         │
          │              ┌──────────────────┐                │
          │              │ For each agent:  │                │
          │              │ 1. Check disabled│                │
          │              │ 2. Resolve model │                │
          │              │ 3. buildAgent()  │                │
          │              │ 4. applyOverrides│                │
          │              └──────────────────┘                │
          │                         │                         │
          └─────────────────────────┼─────────────────────────┘
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Result: AgentConfig Registry                   │
│                                                                          │
│   {                                                                      │
│     sisyphus: AgentConfig,                                               │
│     hephaestus: AgentConfig,                                             │
│     oracle: AgentConfig,                                                 │
│     librarian: AgentConfig,                                              │
│     explore: AgentConfig,                                                │
│     ...                                                                  │
│   }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### `buildAgent()` 组合流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          buildAgent(source, model, ...)                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 1: Get Base Config                                                 │
│                                                                          │
│  if (isFactory(source))                                                  │
│    base = source(model)  // Call factory function                        │
│  else                                                                    │
│    base = { ...source }  // Use raw config                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 2: Apply Category Config (if agent has category)                   │
│                                                                          │
│  categoryConfig = categories[agent.category]                             │
│  base.model ??= categoryConfig.model                                     │
│  base.temperature ??= categoryConfig.temperature                         │
│  base.variant ??= categoryConfig.variant                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Step 3: Inject Skills (if agent has skills)                             │
│                                                                          │
│  skillContent = resolveMultipleSkills(agent.skills)                      │
│  base.prompt = skillContent + base.prompt                                │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Return: AgentConfig                             │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 关键文件参考

| 文件 | 用途 |
|------|------|
| `src/agents/types.ts` | `AgentFactory`、`AgentMode`、`AgentOverrides` 类型定义 |
| `src/agents/agent-builder.ts` | `buildAgent()` 组合函数 |
| `src/agents/builtin-agents.ts` | `createBuiltinAgents()` 主编排器 |
| `src/agents/builtin-agents/general-agents.ts` | `collectPendingBuiltinAgents()` |
| `src/agents/builtin-agents/sisyphus-agent.ts` | `maybeCreateSisyphusConfig()` |
| `src/agents/builtin-agents/hephaestus-agent.ts` | `maybeCreateHephaestusConfig()` |
| `src/agents/builtin-agents/atlas-agent.ts` | `maybeCreateAtlasConfig()` |
| `src/agents/builtin-agents/model-resolution.ts` | `applyModelResolution()` |
| `src/agents/builtin-agents/agent-overrides.ts` | `applyOverrides()` |
| `src/shared/model-requirements.ts` | `AGENT_MODEL_REQUIREMENTS` 回退链 |
