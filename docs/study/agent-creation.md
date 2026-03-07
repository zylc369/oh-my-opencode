# How oh-my-opencode Creates Agents

## Conclusion

Oh-my-opencode creates agents through a **factory pattern with compositing**:

1. **Factory Functions**: Each agent has a `createXXXAgent(model)` factory that returns `AgentConfig`
2. **Compositing via `buildAgent()`**: Composes factory output with categories, skills, and overrides
3. **Registry Pattern**: `createBuiltinAgents()` collects all agent factories into a registry
4. **Conditional Creation**: `maybeCreateXXXConfig()` functions apply model availability checks before instantiation
5. **Multi-source Merge**: Final config merges built-in + user + project + plugin agents

The key insight: **Agents are not classes, they're configurations built by composing factory functions with runtime context.**

---

## Detailed Explanation

### 1. Agent Factory Pattern

Every agent is defined as a factory function following this signature:

```typescript
// src/agents/types.ts
export type AgentFactory = ((model: string) => AgentConfig) & {
  mode: AgentMode  // "primary" | "subagent" | "all"
}
```

Example factory:

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

The `mode` property determines UI behavior:
- **primary**: Respects user's UI-selected model (sisyphus, atlas)
- **subagent**: Uses own fallback chain, ignores UI selection (oracle, explore, etc.)
- **all**: Available in both contexts (Sisyphus-Junior)

### 2. Agent Registry (`agentSources`)

All factories are registered in a single object:

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

### 3. Compositing with `buildAgent()`

The `buildAgent()` function composes factory output with additional layers:

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

### 4. Conditional Creation with `maybeCreateXXXConfig()`

Special agents (sisyphus, hephaestus, atlas) use conditional factories that check:

1. Is the agent disabled?
2. Is the required model available?
3. Are fallback models available?

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

### 5. Main Orchestrator: `createBuiltinAgents()`

The main function that creates all agents:

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
    // ... other params
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

### 6. Model Resolution Chain

When creating an agent, model resolution follows this priority:

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

## Flow Diagrams

### Agent Creation Flow

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

### `buildAgent()` Composition Flow

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

## Key Files Reference

| File | Purpose |
|------|---------|
| `src/agents/types.ts` | `AgentFactory`, `AgentMode`, `AgentOverrides` types |
| `src/agents/agent-builder.ts` | `buildAgent()` compositing function |
| `src/agents/builtin-agents.ts` | `createBuiltinAgents()` main orchestrator |
| `src/agents/builtin-agents/general-agents.ts` | `collectPendingBuiltinAgents()` |
| `src/agents/builtin-agents/sisyphus-agent.ts` | `maybeCreateSisyphusConfig()` |
| `src/agents/builtin-agents/hephaestus-agent.ts` | `maybeCreateHephaestusConfig()` |
| `src/agents/builtin-agents/atlas-agent.ts` | `maybeCreateAtlasConfig()` |
| `src/agents/builtin-agents/model-resolution.ts` | `applyModelResolution()` |
| `src/agents/builtin-agents/agent-overrides.ts` | `applyOverrides()` |
| `src/shared/model-requirements.ts` | `AGENT_MODEL_REQUIREMENTS` fallback chains |
