# How build/plan Agents Are Hidden from UI

## Conclusion

Oh-my-opencode hides `build` and `plan` agents from the UI through a **two-phase filtering strategy**:

1. **`build` agent**: Completely filtered out from the config, then re-added as `OpenCode-Builder` with `hidden: true` and `mode: "subagent"`
2. **`plan` agent**: Demoted to `prometheus` when Sisyphus is enabled, with original `plan` config receiving `mode: "subagent"` and `hidden: true`

The hiding is implemented in `src/plugin-handlers/agent-config-handler.ts` during the 6-phase config loading pipeline.

---

## Detailed Explanation

### 1. The Two Agents Being Hidden

#### `build` Agent
- OpenCode's default build agent for autonomous task execution
- Oh-my-opencode replaces it with `OpenCode-Builder` (optional, controlled by `default_builder_enabled`)

#### `plan` Agent
- OpenCode's default planning agent
- Oh-my-opencode replaces it with `prometheus` (enabled by default, controlled by `planner_enabled`)

### 2. Configuration Flags

These behaviors are controlled by `sisyphus_agent` config section:

```typescript
// .opencode/oh-my-opencode.jsonc
{
  "sisyphus_agent": {
    "disabled": false,              // If true, no hiding happens
    "default_builder_enabled": false, // If true, create OpenCode-Builder
    "planner_enabled": true,         // If true, create prometheus
    "replace_plan": true             // If true, demote plan to hidden
  }
}
```

### 3. Phase 1: Filter Out from Config Agents

In `applyAgentConfig()`, when Sisyphus is enabled, the code filters out `build` and `plan` from the user's config agents:

```typescript
// src/plugin-handlers/agent-config-handler.ts (lines 174-188)

const filteredConfigAgents = configAgent
  ? Object.fromEntries(
      Object.entries(configAgent)
        .filter(([key]) => {
          // Filter out "build" entirely
          if (key === "build") return false
          
          // Filter out "plan" if we're demoting it to prometheus
          if (key === "plan" && shouldDemotePlan) return false
          
          // Don't override builtin agents
          if (key in builtinAgents) return false
          
          return true
        })
        .map(([key, value]) => [
          key,
          value ? migrateAgentConfig(value as Record<string, unknown>) : value,
        ]),
    )
  : {}
```

### 4. Phase 2: Re-add with Hidden Flag

#### 4.1 `build` → `OpenCode-Builder` with `hidden: true`

```typescript
// src/plugin-handlers/agent-config-handler.ts (lines 147-159)

if (builderEnabled) {
  const { name: _buildName, ...buildConfigWithoutName } =
    configAgent?.build ?? {}
  const migratedBuildConfig = migrateAgentConfig(
    buildConfigWithoutName as Record<string, unknown>,
  )
  const override = params.pluginConfig.agents?.["OpenCode-Builder"]
  const base = {
    ...migratedBuildConfig,
    description: `${(configAgent?.build?.description as string) ?? "Build agent"} (OpenCode default)`,
  }
  agentConfig["OpenCode-Builder"] = override ? { ...base, ...override } : base
}
```

Later, the original `build` key is re-added with hidden flag:

```typescript
// src/plugin-handlers/agent-config-handler.ts (line 210)

params.config.agent = {
  ...agentConfig,
  ...Object.fromEntries(
    Object.entries(builtinAgents).filter(([key]) => key !== "sisyphus"),
  ),
  ...filterDisabledAgents(userAgents),
  ...filterDisabledAgents(projectAgents),
  ...filterDisabledAgents(pluginAgents),
  ...filteredConfigAgents,
  build: { ...migratedBuild, mode: "subagent", hidden: true },  // ← HIDDEN
  ...(planDemoteConfig ? { plan: planDemoteConfig } : {}),
}
```

#### 4.2 `plan` → Demoted with `mode: "subagent", hidden: true`

The plan demotion config is built by `buildPlanDemoteConfig()`:

```typescript
// src/plugin-handlers/plan-model-inheritance.ts

const MODEL_SETTINGS_KEYS = [
  "model",
  "variant",
  "temperature",
  "top_p",
  "maxTokens",
  "thinking",
  "reasoningEffort",
  "textVerbosity",
  "providerOptions",
] as const

export function buildPlanDemoteConfig(
  prometheusConfig: Record<string, unknown> | undefined,
  planOverride: Record<string, unknown> | undefined,
): Record<string, unknown> {
  const modelSettings: Record<string, unknown> = {}

  // Inherit model settings from plan override or prometheus config
  for (const key of MODEL_SETTINGS_KEYS) {
    const value = planOverride?.[key] ?? prometheusConfig?.[key]
    if (value !== undefined) {
      modelSettings[key] = value
    }
  }

  return { mode: "subagent" as const, ...modelSettings }
}
```

**Important**: The demoted `plan` config does NOT have `hidden: true` explicitly set. Instead, the hiding happens because:

1. The `plan` key is filtered out from `filteredConfigAgents`
2. Only `planDemoteConfig` is added back (without `hidden` key)
3. The `mode: "subagent"` prevents it from appearing as a primary agent option

### 5. Prometheus Replacement

While `plan` is hidden, `prometheus` is created as the visible replacement:

```typescript
// src/plugin-handlers/agent-config-handler.ts (lines 161-172)

if (plannerEnabled) {
  const prometheusOverride = params.pluginConfig.agents?.["prometheus"] as
    | (Record<string, unknown> & { prompt_append?: string })
    | undefined

  agentConfig["prometheus"] = await buildPrometheusAgentConfig({
    configAgentPlan: configAgent?.plan,
    pluginPrometheusOverride: prometheusOverride,
    userCategories: params.pluginConfig.categories,
    currentModel,
  })
}
```

Prometheus is created with `mode: "all"`, making it visible in the UI:

```typescript
// src/plugin-handlers/prometheus-agent-config-builder.ts (lines 68-87)

const base: Record<string, unknown> = {
  ...(resolvedModel ? { model: resolvedModel } : {}),
  ...(variantToUse ? { variant: variantToUse } : {}),
  mode: "all",  // ← Visible in UI
  prompt: getPrometheusPrompt(resolvedModel),
  permission: PROMETHEUS_PERMISSION,
  description: `${(params.configAgentPlan?.description as string) ?? "Plan agent"} (Prometheus - OhMyOpenCode)`,
  color: (params.configAgentPlan?.color as string) ?? "#FF5722",
  // ... other settings
}
```

---

## Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    applyAgentConfig() Entry                              │
│            (src/plugin-handlers/agent-config-handler.ts)                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Check: isSisyphusEnabled && builtinAgents.sisyphus?         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │ NO                            │ YES
                    ▼                               ▼
┌──────────────────────────┐    ┌──────────────────────────────────────────┐
│ No hiding occurs         │    │ Begin hide/demote process                │
│ Return original config   │    └──────────────────────────────────────────┘
└──────────────────────────┘                            │
                                                        ▼
                        ┌───────────────────────────────────────────────────┐
                        │ Step 1: Filter configAgent                        │
                        │                                                    │
                        │ filteredConfigAgents = configAgent.filter(key =>  │
                        │   key !== "build" &&                               │
                        │   !(key === "plan" && shouldDemotePlan)            │
                        │ )                                                  │
                        └───────────────────────────────────────────────────┘
                                                        │
                        ┌───────────────┬───────────────┴───────────────┐
                        ▼               ▼                               ▼
            ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
            │ builderEnabled?  │ │ plannerEnabled?  │ │ shouldDemotePlan?│
            └──────────────────┘ └──────────────────┘ └──────────────────┘
                    │                   │                   │
                    ▼                   ▼                   ▼
            ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
            │ Create           │ │ Create           │ │ Build            │
            │ OpenCode-Builder │ │ prometheus       │ │ planDemoteConfig │
            │ (mode: "all")    │ │ (mode: "all")    │ │ (mode: "subagent")│
            └──────────────────┘ └──────────────────┘ └──────────────────┘
                                                        │
                                                        ▼
                        ┌───────────────────────────────────────────────────┐
                        │ Step 2: Assemble final config                     │
                        │                                                    │
                        │ params.config.agent = {                           │
                        │   ...agentConfig,           // sisyphus, prometheus│
                        │   ...builtinAgents,         // other agents        │
                        │   ...filteredConfigAgents,  // user agents (no     │
                        │                              // build/plan)        │
                        │   build: {                                        │
                        │     ...migratedBuild,                             │
                        │     mode: "subagent",                             │
                        │     hidden: true          // ← HIDDEN             │
                        │   },                                              │
                        │   plan: planDemoteConfig  // ← mode: "subagent"   │
                        │ }                                                 │
                        └───────────────────────────────────────────────────┘
                                                        │
                                                        ▼
                        ┌───────────────────────────────────────────────────┐
                        │ Result: UI shows prometheus, not plan            │
                        │         build is hidden, OpenCode-Builder shown  │
                        │         (if enabled)                              │
                        └───────────────────────────────────────────────────┘
```

---

## Key Code Locations

| File | Lines | Purpose |
|------|-------|---------|
| `src/plugin-handlers/agent-config-handler.ts` | 118-123 | Read config flags |
| `src/plugin-handlers/agent-config-handler.ts` | 147-159 | Create OpenCode-Builder |
| `src/plugin-handlers/agent-config-handler.ts` | 161-172 | Create prometheus |
| `src/plugin-handlers/agent-config-handler.ts` | 174-188 | Filter build/plan from config |
| `src/plugin-handlers/agent-config-handler.ts` | 190-211 | Assemble final config with hidden flags |
| `src/plugin-handlers/plan-model-inheritance.ts` | 13-27 | Build plan demote config |
| `src/plugin-handlers/prometheus-agent-config-builder.ts` | 25-98 | Build prometheus config |

---

## Agent Mode Visibility Summary

| Agent | Mode | Hidden | Visible in UI |
|-------|------|--------|---------------|
| `sisyphus` | `all` | No | Yes (default agent) |
| `prometheus` | `all` | No | Yes (replaces plan) |
| `OpenCode-Builder` | `all` | No | Yes (if enabled) |
| `build` | `subagent` | Yes | No |
| `plan` | `subagent` | No* | No (subagent mode) |
| `oracle`, `librarian`, etc. | `subagent` | No | No (subagent mode) |

*Note: `plan` doesn't have explicit `hidden: true`, but `mode: "subagent"` prevents it from appearing as a selectable primary agent.
