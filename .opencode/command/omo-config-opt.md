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

| Agent Type | Agents | Recommended Models | Key Traits |
|------------|--------|-------------------|------------|
| **Communicators** | Sisyphus, Metis | Claude Opus → Kimi K2.5 → GLM 5 | Complex instructions, orchestration |
| **Dual-Prompt** | Prometheus, Atlas | Claude Opus → GPT-5.4 → Gemini 3.1 Pro | Auto-switch Claude/GPT prompts |
| **Deep Specialists** | Hephaestus, Oracle, Momus | GPT-5.3 Codex, GPT-5.4, Gemini 3.1 Pro | Principle-driven, autonomous |
| **Utility Runners** | Explore, Librarian, Multimodal-Looker | Grok Code Fast, Gemini Flash, MiniMax | Speed > Intelligence, **NEVER upgrade to Opus** |

### Category Model Defaults

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

| Variant | Cost | Usage |
|---------|------|-------|
| `max` | Highest | Sisyphus ultrawork, complex architecture |
| `xhigh` | Very High | Momus (review), ultrabrain category |
| `high` | High | Oracle, Prometheus, visual-engineering, deep |
| `medium` | Medium | Prometheus, Atlas, unspecified-low, utility agents |
| `low` | Low | Explore, Librarian, quick category |

**Optimization Rules:**
- Critical agents: `max` or `xhigh` (Sisyphus ultrawork, Momus)
- Specialized agents: `high` or `medium` (Oracle, Prometheus)
- Utility agents: `low` or `medium` (Explore, Librarian) - **never upgrade**
- Categories: Match complexity (`ultrabrain`→xhigh, `quick`→low)
- Cost optimization: Downgrade `max`→`high`, `high`→`medium`
- Quality improvement: Upgrade `medium`→`high`, `high`→`xhigh`

**Special Case: Ultrawork Override**
```jsonc
{
  "agents": {
    "sisyphus": {
      "model": "kimi-for-coding/k2p5",
      "variant": "high",
      "ultrawork": { "model": "anthropic/claude-opus-4-6", "variant": "max" }
    }
  }
}
```

</optimization-principles>

---

## PHASE 1: GATHER — Collect User's Model Access

<user-model-survey>

**CRITICAL: Ask the user FIRST about their model access.**

```
Before optimizing, I need to know which models you have access to.

**Claude Family:**
- [ ] Claude Opus 4.6 / Sonnet 4.6 / Haiku 4.5
- [ ] Kimi K2.5 / GLM 5

**GPT Family:**
- [ ] GPT-5.3 Codex / GPT-5.4 / GPT-5-Nano

**Gemini Family:**
- [ ] Gemini 3.1 Pro / Gemini Flash

**Utility Models:**
- [ ] Grok Code Fast 1 / MiniMax M2.5 / Big Pickle

**Free Tier:**
- [ ] I'm okay using free-tier models (kimi-k2.5-free, minimax-m2.5-free, big-pickle)

Reply with your models or paste `opencode models` output.
```

Wait for user response before proceeding.

</user-model-survey>

---

## PHASE 2: ANALYZE — Read Current Configuration

<config-analysis>

```bash
# Read config (check both .json and .jsonc)
cat ~/.config/opencode/oh-my-opencode.jsonc 2>/dev/null || cat ~/.config/opencode/oh-my-opencode.json 2>/dev/null
```

**Analyze:**

1. **Model issues**
   - Suboptimal model-agent matching
   - Cost inefficiencies (Explore using Opus)
   - Missing user overrides (using defaults)

2. **Variant issues**
   - Critical agents with `low`/`medium` (should be `high`/`max`)
   - Utility agents with `high`/`max` (should be `low`/`medium`)
   - Missing ultrawork override for Sisyphus
   - Inappropriate category variants

3. **Cross-reference with user access**
   - Filter unavailable models
   - Adjust fallback chains
   - Suggest free-tier alternatives

</config-analysis>

---

## PHASE 3: RECOMMEND — Generate Optimization Report

<recommendation-template>

```markdown
## Configuration Optimization Report

### Model Access Summary
- Available: [list models]
- Missing: [list models]

### Agent Optimizations

| Agent | Current (Model/Variant) | Recommended (Model/Variant) | Reason |
|-------|------------------------|----------------------------|--------|
| sisyphus | claude-opus-4-6 / max | kimi-k2.5 / high | Cost savings |
| sisyphus.ultrawork | (none) | claude-opus-4-6 / max | Critical tasks |
| explore | claude-opus-4-6 / high | grok-code-fast-1 / low | Speed/cost |
| oracle | gpt-4-turbo / medium | gpt-5.4 / high | Better reasoning |

**Variant Changes:** Upgraded: oracle (medium→high) | Downgraded: explore (high→low) | Added: ultrawork override

### Category Optimizations

| Category | Current (Model/Variant) | Recommended (Model/Variant) | Reason |
|----------|------------------------|----------------------------|--------|
| quick | (default) | gpt-5-nano / low | Trivial tasks |
| ultrabrain | gpt-5.3-codex / high | gpt-5.3-codex / xhigh | Max reasoning |

**Cost Impact:** -XX% | **Performance Impact:** +Y%

**Apply these changes?** (yes/no/modify)
```

Wait for user confirmation.

</recommendation-template>

---

## PHASE 4: APPLY — Update Configuration

<apply-changes>

If user confirms:

### Step 1: Backup

```bash
mkdir -p ~/.config/opencode/oh-my-opencode-bak
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

if [ -f ~/.config/opencode/oh-my-opencode.json ]; then
  cp ~/.config/opencode/oh-my-opencode.json ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP
  CONFIG_FILE=~/.config/opencode/oh-my-opencode.json
elif [ -f ~/.config/opencode/oh-my-opencode.jsonc ]; then
  cp ~/.config/opencode/oh-my-opencode.jsonc ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP
  CONFIG_FILE=~/.config/opencode/oh-my-opencode.jsonc
else
  CONFIG_FILE=~/.config/opencode/oh-my-opencode.json
fi
```

### Step 2: Merge Configuration

```typescript
const currentConfig = fs.existsSync(configPath) ? parseJSONC(fs.readFileSync(configPath, 'utf-8')) : {};

const optimizedConfig = {
  ...currentConfig,
  agents: { ...currentConfig.agents, ...recommendedAgents },
  categories: { ...currentConfig.categories, ...recommendedCategories },
};
```

### Step 3: Write Configuration

```typescript
fs.writeFileSync(
  path.expandTilde('~/.config/opencode/oh-my-opencode.json'),
  JSON.stringify(optimizedConfig, null, 2),
  'utf-8'
);
```

**Important:** Always write to `.json`, preserve all existing fields, only update `agents` and `categories`.

### Step 4: Validate

```bash
bunx oh-my-opencode doctor || {
  echo "Validation failed! Restoring backup..."
  cp ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP ~/.config/opencode/oh-my-opencode.json
  exit 1
}
```

### Step 5: Report

```markdown
## ✓ Changes Applied

**Backup:** `~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP`

**Updates:**
- Agents: X configurations updated
- Categories: Y configurations updated

**Restore:** `cp ~/.config/opencode/oh-my-opencode-bak/oh-my-opencode.json.$TIMESTAMP ~/.config/opencode/oh-my-opencode.json`
```

</apply-changes>

---

## SCOPE CONTROL

If `$ARGUMENTS` provided:
- `agents` — only optimize agents section
- `categories` — only optimize categories section
- `check` — analyze only, don't apply
- Specific name — only optimize that item

## ABORT CONDITIONS

STOP if:
- User has no model access
- Current config is empty
- Cost increase without user approval

## REFERENCE

- [Agent-Model Matching Guide](docs/guide/agent-model-matching.md)
- [Configuration Reference](docs/reference/configuration.md)
- `opencode models` — list available models
- `opencode auth login` — authenticate providers

</command-instruction>

<user-request>
$ARGUMENTS
</user-request>
