---
description: Optimize oh-my-opencode configuration for better model-agent matching and cost efficiency
---

<command-instruction>

Configuration optimization via analysis of model availability, agent requirements, and category defaults. You are the OPTIMIZER — analyze, recommend, verify, then apply.

<rules>
- **User's models are law.** Only recommend models the user has access to.
- **No adding/removing agents/categories.** Only optimize existing `agents` and `categories` fields.
- **Cost matters.** Balance performance with cost efficiency.
- **Model-agent matching matters.** Follow the agent-model matching guide.
</rules>

<optimization-principles>

### Agent-Model Matching

Different agents need different model personalities:

**Communicators (Claude/Kimi/GLM)** → Sisyphus, Metis
- Excel at following complex, multi-step instructions
- Good for orchestration and coordination tasks
- Sisyphus: Claude Opus → Kimi K2.5 → GLM 5

**Dual-Prompt Agents** → Prometheus, Atlas
- Auto-switch between Claude and GPT prompts at runtime
- Prometheus: Claude Opus → GPT-5.4 → Gemini 3.1 Pro
- Atlas: Claude Sonnet 4.6 → GPT-5.4

**Deep Specialists (GPT)** → Hephaestus, Oracle, Momus
- Principle-driven, autonomous execution
- Hephaestus: GPT-5.3 Codex ONLY (no fallback)
- Oracle: GPT-5.4 → Gemini 3.1 Pro → Claude Opus
- Momus: GPT-5.4 → Claude Opus → Gemini 3.1 Pro

**Utility Runners (Speed > Intelligence)** → Explore, Librarian, Multimodal-Looker
- Fast, cheap models for grep/search/retrieval
- Explore: Grok Code Fast → MiniMax → Haiku → GPT-5-Nano
- Librarian: Gemini Flash → MiniMax → Big Pickle
- **NEVER upgrade to Opus** — massive cost waste

### Category Model Defaults

Categories map task types to models:

| Category | Purpose | Fallback Chain |
|----------|---------|----------------|
| `visual-engineering` | Frontend/UI | Gemini 3.1 Pro → GLM 5 → Claude Opus |
| `ultrabrain` | Max reasoning | GPT-5.3 Codex → Gemini 3.1 Pro → Claude Opus |
| `deep` | Deep coding | GPT-5.3 Codex → Claude Opus → Gemini 3.1 Pro |
| `artistry` | Creative work | Gemini 3.1 Pro → Claude Opus → GPT-5.4 |
| `quick` | Simple tasks | Claude Haiku → Gemini Flash → GPT-5-Nano |
| `unspecified-high` | Complex work | GPT-5.4 → Claude Opus → GLM 5 → K2P5 |
| `unspecified-low` | Standard work | Claude Sonnet → GPT-5.3 Codex → Gemini Flash |
| `writing` | Text/docs | Gemini Flash → Claude Sonnet |

### Variant Optimization

Variant controls model quality/intelligence level. Choose based on task criticality and cost:

| Variant | Intelligence | Cost | When to Use |
|---------|-------------|------|-------------|
| `max` | Maximum | Highest | Critical orchestration (Sisyphus ultrawork), complex architecture |
| `xhigh` | Extra High | Very High | Hard reasoning tasks, plan review (Momus), ultrabrain category |
| `high` | High | High | Strategic tasks (Oracle, Prometheus), visual-engineering, deep coding |
| `medium` | Medium | Medium | Standard tasks, unspecified-low, utility agents (Explore, Librarian) |
| `low` | Low | Low | Simple tasks, quick category, doc search |

**Variant Optimization Rules:**

1. **Critical Agents → `max` or `xhigh`**
   - Sisyphus (orchestrator): `max` for ultrawork, `high` for normal tasks
   - Momus (ruthless reviewer): `xhigh` for thorough verification
   - Oracle (architecture): `high` for consultation quality

2. **Specialized Agents → `high` or `medium`**
   - Prometheus (planner): `high` for strategic planning
   - Hephaestus (deep worker): `medium` balances cost and capability
   - Atlas (todo orchestrator): `medium` for efficient coordination

3. **Utility Agents → `low` or `medium`**
   - Explore (grep): `low` (speed > intelligence)
   - Librarian (docs): `low` (retrieval doesn't need deep reasoning)
   - Multimodal-Looker: `medium` (vision tasks need some intelligence)

4. **Categories by Complexity:**
   - `ultrabrain`: `xhigh` (maximum reasoning)
   - `deep`, `artistry`, `visual-engineering`: `high` (quality matters)
   - `unspecified-high`: `high` (complex work)
   - `unspecified-low`: `medium` (standard work)
   - `quick`, `writing`: `low` or `medium` (simple/fast tasks)

5. **Cost-Performance Balance:**
   - If budget is tight: downgrade `max` → `high`, `high` → `medium`
   - If quality is critical: upgrade `medium` → `high`, `high` → `xhigh`
   - **Never upgrade utility agents** (Explore, Librarian) — speed is their value

**Special Case: Ultrawork Override**

Sisyphus supports per-message ultrawork variant override:
```jsonc
{
  "agents": {
    "sisyphus": {
      "model": "kimi-for-coding/k2p5",
      "variant": "high",  // Normal tasks
      "ultrawork": {
        "model": "anthropic/claude-opus-4-6",
        "variant": "max"  // Ultrawork mode: maximum intelligence
      }
    }
  }
}
```


---

## PHASE 1: GATHER — Collect User's Model Access

<user-model-survey>

**CRITICAL: Ask the user FIRST about their model access.**

Present this survey to the user:

```
Before optimizing your configuration, I need to know which models you have access to.

**Claude Family:**
- [ ] Claude Opus 4.6 (Anthropic subscription)
- [ ] Claude Sonnet 4.6 (Anthropic subscription)
- [ ] Claude Haiku 4.5 (Anthropic subscription)
- [ ] Kimi K2.5 (Kimi subscription or pay-per-token)
- [ ] GLM 5 (GLM subscription or pay-per-token)

**GPT Family:**
- [ ] GPT-5.3 Codex (OpenAI subscription)
- [ ] GPT-5.4 (OpenAI subscription)
- [ ] GPT-5-Nano (OpenAI or OpenCode Zen free tier)

**Gemini Family:**
- [ ] Gemini 3.1 Pro (Google AI subscription)
- [ ] Gemini Flash (Google AI or pay-per-token)

**Utility Models:**
- [ ] Grok Code Fast 1 (Copilot or OpenCode Zen)
- [ ] MiniMax M2.5 (OpenCode Zen free tier)
- [ ] Big Pickle / GLM 4.6 (OpenCode Zen free tier)

**Free Tier (OpenCode Zen):**
- [ ] I'm okay using free-tier models where appropriate (kimi-k2.5-free, minimax-m2.5-free, big-pickle)

Please reply with the models you have access to, or paste your `opencode models` output.
```

Wait for user response before proceeding.

</user-model-survey>

---

## PHASE 2: ANALYZE — Read Current Configuration

<config-analysis>

Read the user's configuration file:

```bash
# Check user config first, then project config
cat ~/.config/opencode/oh-my-opencode.jsonc 2>/dev/null || cat ~/.config/opencode/oh-my-opencode.json 2>/dev/null
cat .opencode/oh-my-opencode.jsonc 2>/dev/null || cat .opencode/oh-my-opencode.json 2>/dev/null
```

Analyze:

1. **Current agent configurations**
   - For each agent: model, variant, fallback_models
   - Check if model matches agent's personality type
   - Identify cost inefficiencies (e.g., Explore using Opus)

2. **Current category configurations**
   - For each category: model, variant, description
   - Check if model matches category's purpose
   - Verify fallback chains are reasonable

3. **Identify optimization opportunities**
   - **Model issues:**
     - Agents using suboptimal models for their type
     - Categories with no user override (using defaults)
     - Cost-saving opportunities (switching to cheaper models)
     - Performance improvements (switching to better models)
   
   - **Variant issues:**
     - Critical agents using `low` or `medium` (should be `high` or `max`)
     - Utility agents using `high` or `max` (should be `low` or `medium`)
     - Categories using inappropriate variants for their complexity
     - Missing ultrawork override for Sisyphus (should have `max` variant)
     - Cost optimization: downgrade variants where quality isn't critical
     - Quality improvement: upgrade variants for critical tasks

4. **Cross-reference with user's model access**
   - Filter out recommendations requiring unavailable models
   - Adjust fallback chains based on available providers
   - Suggest free-tier alternatives when appropriate
</config-analysis>

---

## PHASE 3: RECOMMEND — Generate Optimization Report

<recommendation-template>

Present a structured recommendation:

```markdown
## Configuration Optimization Report

### Model Access Summary
- Available: [list user's models]
- Missing: [list unavailable models]

### Agent Optimizations

| Agent | Current | Recommended | Reason |
|-------|---------|-------------|--------|
| sisyphus | claude-opus-4-6 | kimi-k2.5 | Cost savings with similar performance |
| explore | claude-opus-4-6 | grok-code-fast-1 | 10x faster, 100x cheaper for grep tasks |
| oracle | gpt-4-turbo | gpt-5.4 | Better reasoning for architecture consultation |

### Category Optimizations

| Category | Current | Recommended | Reason |
|----------|---------|-------------|--------|
| quick | (default) | gpt-5-nano | Ultra-cheap for trivial tasks |
| visual-engineering | claude-opus-4-6 | gemini-3.1-pro | Gemini excels at visual tasks |

### Cost Impact

- Estimated cost change: -XX% (savings from cheaper models)
- Performance impact: +Y% (improvements from better model-agent matching)

### Recommended Configuration

```jsonc
{
  "agents": {
    "sisyphus": {
      "model": "kimi-for-coding/k2p5",
      "ultrawork": { "model": "anthropic/claude-opus-4-6", "variant": "max" }
    },
    "explore": { "model": "github-copilot/grok-code-fast-1" },
    // ... other agents
  },
  "categories": {
    "quick": { "model": "opencode/gpt-5-nano" },
    "visual-engineering": { "model": "google/gemini-3.1-pro", "variant": "high" },
    // ... other categories
  }
}
```

**Apply these changes?** (yes/no/modify)
```

Wait for user confirmation.

</recommendation-template>

---

## PHASE 4: APPLY — Update Configuration

<apply-changes>

If user confirms, apply changes with backup and validation:

### Step 1: Create Backup Directory and Backup

```bash
# Create backup directory if not exists
mkdir -p ~/.config/opencode/oh-my-opencode-bak

# Generate timestamp (format: YYYYMMDD_HHMMSS)
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Backup current config (check both .json and .jsonc)
if [ -f ~/.config/opencode/oh-my-opencode.json ]; then
  cp ~/.config/opencode/oh-my-opencode.json ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP
  CONFIG_FILE=~/.config/opencode/oh-my-opencode.json
elif [ -f ~/.config/opencode/oh-my-opencode.jsonc ]; then
  cp ~/.config/opencode/oh-my-opencode.jsonc ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP
  CONFIG_FILE=~/.config/opencode/oh-my-opencode.jsonc
else
  # No existing config, will create new one
  CONFIG_FILE=~/.config/opencode/oh-my-opencode.json
fi

echo "Backup saved to: ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP"
```

### Step 2: Read and Merge Configuration

Read the current configuration file:

```typescript
// Read existing config
const configPath = CONFIG_FILE;
let currentConfig = {};

if (fs.existsSync(configPath)) {
  const content = fs.readFileSync(configPath, 'utf-8');
  // Parse JSONC (remove comments and trailing commas)
  currentConfig = parseJSONC(content);
}
```

Merge optimized settings into current configuration:

```typescript
// Deep merge - preserve all existing settings
const optimizedConfig = {
  ...currentConfig,  // Keep all existing fields
  
  // Update agents section (deep merge)
  agents: {
    ...currentConfig.agents,
    ...recommendedAgents,  // Override with optimized settings
  },
  
  // Update categories section (deep merge)
  categories: {
    ...currentConfig.categories,
    ...recommendedCategories,  // Override with optimized settings
  },
};
```

### Step 3: Write Updated Configuration

Write to `~/.config/opencode/oh-my-opencode.json` (always use .json for output):

```typescript
import fs from 'fs';

const outputPath = path.expandTilde('~/.config/opencode/oh-my-opencode.json');

// Format with 2-space indentation
const outputContent = JSON.stringify(optimizedConfig, null, 2);

fs.writeFileSync(outputPath, outputContent, 'utf-8');

console.log(`✓ Configuration written to: ${outputPath}`);
```

**Important**: 
- Always write to `~/.config/opencode/oh-my-opencode.json` (not .jsonc)
- Preserve all existing configuration fields (disabled_hooks, background_task, etc.)
- Only update `agents` and `categories` sections
- Maintain JSON formatting with proper indentation

### Step 4: Validate Configuration

```bash
# Run doctor to verify config is valid
bunx oh-my-opencode doctor

# If validation fails, restore from backup
if [ $? -ne 0 ]; then
  echo "Validation failed! Restoring backup..."
  cp ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP ~/.config/opencode/oh-my-opencode.json
  echo "Backup restored. Please check the configuration."
  exit 1
fi
```

### Step 5: Report Changes

```markdown
## ✓ Changes Applied Successfully

### Backup
- Location: `~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP`
- Original file preserved

### Updates Applied
- **Agents**: Updated X agent configurations
  - sisyphus: claude-opus-4-6 → kimi-k2.5
  - explore: claude-opus-4-6 → grok-code-fast-1
  - oracle: gpt-4-turbo → gpt-5.4
- **Categories**: Updated Y category configurations
  - quick: (default) → gpt-5-nano
  - visual-engineering: claude-opus-4-6 → gemini-3.1-pro

### Configuration File
- Updated: `~/.config/opencode/oh-my-opencode.json`
- All other settings preserved (hooks, skills, MCPs, etc.)

### Validation
- ✓ Configuration validated successfully
- ✓ No errors detected

### Next Steps
1. Test with `ultrawork` command
2. Monitor costs and performance
3. To restore backup:
   ```bash
   cp ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP ~/.config/opencode/oh-my-opencode.json
   ```
```

</apply-changes>

---

## SCOPE CONTROL

If `$ARGUMENTS` is provided:
- `agents` — only optimize agents section
- `categories` — only optimize categories section
- `check` — analyze only, don't apply changes
- Specific agent/category name — only optimize that item

## ABORT CONDITIONS

STOP and ask user if:
- User has no model access (can't proceed without models)
- Current config is empty (need initial setup first)
- Recommended changes would increase cost significantly without user approval

## REFERENCE

- [Agent-Model Matching Guide](docs/guide/agent-model-matching.md)
- [Configuration Reference](docs/reference/configuration.md)
- Run `opencode models` to see available models
- Run `opencode auth login` to authenticate providers

</command-instruction>

<user-request>
$ARGUMENTS
</user-request>