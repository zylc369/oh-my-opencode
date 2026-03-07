# build/plan 代理如何在 UI 中隐藏

## 结论

Oh-my-opencode 通过**两阶段过滤策略**在 UI 中隐藏 `build` 和 `plan` 代理:

1. **`build` 代理**: 从配置中完全过滤掉,然后以 `OpenCode-Builder` 重新添加,并设置 `hidden: true` 和 `mode: "subagent"`
2. **`plan` 代理**: 当启用 Sisyphus 时降级为 `prometheus`,原始 `plan` 配置接收 `mode: "subagent"` 和 `hidden: true`

隐藏机制在 6 阶段配置加载管道中的 `src/plugin-handlers/agent-config-handler.ts` 实现。

---

## 详细说明

### 1. 被隐藏的两个代理

#### `build` 代理
- OpenCode 的默认构建代理,用于自主任务执行
- Oh-my-opencode 用 `OpenCode-Builder` 替换它(可选,由 `default_builder_enabled` 控制)

#### `plan` 代理
- OpenCode 的默认规划代理
- Oh-my-opencode 用 `prometheus` 替换它(默认启用,由 `planner_enabled` 控制)

### 2. 配置标志

这些行为由 `sisyphus_agent` 配置部分控制:

```typescript
// .opencode/oh-my-opencode.jsonc
{
  "sisyphus_agent": {
    "disabled": false,              // 如果为 true,不进行隐藏
    "default_builder_enabled": false, // 如果为 true,创建 OpenCode-Builder
    "planner_enabled": true,         // 如果为 true,创建 prometheus
    "replace_plan": true             // 如果为 true,将 plan 降级为隐藏
  }
}
```

### 3. 阶段 1: 从配置代理中过滤掉

在 `applyAgentConfig()` 中,当启用 Sisyphus 时,代码从用户的配置代理中过滤掉 `build` 和 `plan`:

```typescript
// src/plugin-handlers/agent-config-handler.ts (第 174-188 行)

const filteredConfigAgents = configAgent
  ? Object.fromEntries(
      Object.entries(configAgent)
        .filter(([key]) => {
          // 完全过滤掉 "build"
          if (key === "build") return false
          
          // 如果我们要将其降级为 prometheus,则过滤掉 "plan"
          if (key === "plan" && shouldDemotePlan) return false
          
          // 不要覆盖内置代理
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

### 4. 阶段 2: 以隐藏标志重新添加

#### 4.1 `build` → `OpenCode-Builder`,带 `hidden: true`

```typescript
// src/plugin-handlers/agent-config-handler.ts (第 147-159 行)

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

随后,原始 `build` 键以隐藏标志重新添加:

```typescript
// src/plugin-handlers/agent-config-handler.ts (第 210 行)

params.config.agent = {
  ...agentConfig,
  ...Object.fromEntries(
    Object.entries(builtinAgents).filter(([key]) => key !== "sisyphus"),
  ),
  ...filterDisabledAgents(userAgents),
  ...filterDisabledAgents(projectAgents),
  ...filterDisabledAgents(pluginAgents),
  ...filteredConfigAgents,
  build: { ...migratedBuild, mode: "subagent", hidden: true },  // ← 隐藏
  ...(planDemoteConfig ? { plan: planDemoteConfig } : {}),
}
```

#### 4.2 `plan` → 以 `mode: "subagent", hidden: true` 降级

plan 降级配置由 `buildPlanDemoteConfig()` 构建:

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

  // 从 plan 覆盖或 prometheus 配置继承模型设置
  for (const key of MODEL_SETTINGS_KEYS) {
    const value = planOverride?.[key] ?? prometheusConfig?.[key]
    if (value !== undefined) {
      modelSettings[key] = value
    }
  }

  return { mode: "subagent" as const, ...modelSettings }
}
```

**重要**: 降级的 `plan` 配置没有显式设置 `hidden: true`。相反,隐藏发生的原因是:

1. `plan` 键从 `filteredConfigAgents` 中被过滤掉
2. 只有 `planDemoteConfig` 被重新添加(没有 `hidden` 键)
3. `mode: "subagent"` 阻止它作为主要代理选项出现

### 5. Prometheus 替换

当 `plan` 被隐藏时,`prometheus` 作为可见的替代品被创建:

```typescript
// src/plugin-handlers/agent-config-handler.ts (第 161-172 行)

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

Prometheus 以 `mode: "all"` 创建,使其在 UI 中可见:

```typescript
// src/plugin-handlers/prometheus-agent-config-builder.ts (第 68-87 行)

const base: Record<string, unknown> = {
  ...(resolvedModel ? { model: resolvedModel } : {}),
  ...(variantToUse ? { variant: variantToUse } : {}),
  mode: "all",  // ← 在 UI 中可见
  prompt: getPrometheusPrompt(resolvedModel),
  permission: PROMETHEUS_PERMISSION,
  description: `${(params.configAgentPlan?.description as string) ?? "Plan agent"} (Prometheus - OhMyOpenCode)`,
  color: (params.configAgentPlan?.color as string) ?? "#FF5722",
  // ... 其他设置
}
```

---

## 流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    applyAgentConfig() 入口                               │
│            (src/plugin-handlers/agent-config-handler.ts)                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              检查: isSisyphusEnabled && builtinAgents.sisyphus?          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │ NO                            │ YES
                    ▼                               ▼
┌──────────────────────────┐    ┌──────────────────────────────────────────┐
│ 不进行隐藏               │    │ 开始隐藏/降级过程                         │
│ 返回原始配置             │    └──────────────────────────────────────────┘
└──────────────────────────┘                            │
                                                        ▼
                        ┌───────────────────────────────────────────────────┐
                        │ 步骤 1: 过滤 configAgent                          │
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
            │ 创建             │ │ 创建             │ │ 构建             │
            │ OpenCode-Builder │ │ prometheus       │ │ planDemoteConfig │
            │ (mode: "all")    │ │ (mode: "all")    │ │ (mode: "subagent")│
            └──────────────────┘ └──────────────────┘ └──────────────────┘
                                                        │
                                                        ▼
                        ┌───────────────────────────────────────────────────┐
                        │ 步骤 2: 组装最终配置                               │
                        │                                                    │
                        │ params.config.agent = {                           │
                        │   ...agentConfig,           // sisyphus, prometheus│
                        │   ...builtinAgents,         // 其他代理            │
                        │   ...filteredConfigAgents,  // 用户代理(无         │
                        │                              // build/plan)        │
                        │   build: {                                        │
                        │     ...migratedBuild,                             │
                        │     mode: "subagent",                             │
                        │     hidden: true          // ← 隐藏               │
                        │   },                                              │
                        │   plan: planDemoteConfig  // ← mode: "subagent"   │
                        │ }                                                 │
                        └───────────────────────────────────────────────────┘
                                                        │
                                                        ▼
                        ┌───────────────────────────────────────────────────┐
                        │ 结果: UI 显示 prometheus,而不是 plan              │
                        │       build 被隐藏,OpenCode-Builder 显示         │
                        │       (如果启用)                                  │
                        └───────────────────────────────────────────────────┘
```

---

## 关键代码位置

| 文件 | 行号 | 用途 |
|------|-------|---------|
| `src/plugin-handlers/agent-config-handler.ts` | 118-123 | 读取配置标志 |
| `src/plugin-handlers/agent-config-handler.ts` | 147-159 | 创建 OpenCode-Builder |
| `src/plugin-handlers/agent-config-handler.ts` | 161-172 | 创建 prometheus |
| `src/plugin-handlers/agent-config-handler.ts` | 174-188 | 从配置中过滤 build/plan |
| `src/plugin-handlers/agent-config-handler.ts` | 190-211 | 组装带隐藏标志的最终配置 |
| `src/plugin-handlers/plan-model-inheritance.ts` | 13-27 | 构建 plan 降级配置 |
| `src/plugin-handlers/prometheus-agent-config-builder.ts` | 25-98 | 构建 prometheus 配置 |

---

## 代理模式可见性总结

| 代理 | 模式 | 隐藏 | UI 中可见 |
|-------|------|--------|---------------|
| `sisyphus` | `all` | 否 | 是 (默认代理) |
| `prometheus` | `all` | 否 | 是 (替换 plan) |
| `OpenCode-Builder` | `all` | 否 | 是 (如果启用) |
| `build` | `subagent` | 是 | 否 |
| `plan` | `subagent` | 否* | 否 (子代理模式) |
| `oracle`, `librarian` 等 | `subagent` | 否 | 否 (子代理模式) |

*注意: `plan` 没有显式的 `hidden: true`,但 `mode: "subagent"` 阻止它作为可选择的主要代理出现。