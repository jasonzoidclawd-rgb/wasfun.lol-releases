# Ruflo Full Stack Install Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Install and configure the full Ruflo CLI stack in `~/` to activate layered token savings (3-tier routing, LiteLLM :4000 routing, ReasoningBank memory, cost tracking) and unlock swarm-coordinated agentic pipelines.

**Architecture:** Ruflo CLI init writes `~/.claude-flow/` config and helpers; the MCP server registers 300+ coordination tools; six plugins add intelligence, memory, cost tracking, swarm, and local LLM routing. All hooks layer on top of the existing RTK Bash hook without conflict.

**Tech Stack:** ruflo@latest CLI, @claude-flow/cli v3.6, Claude Code plugin system, LiteLLM at localhost:4000, HNSW/AgentDB vector memory, ONNX embeddings (all-MiniLM-L6-v2 384-dim)

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `~/CLAUDE.md` | Preserve (merge only) | Existing behavioral guidelines — must not be overwritten |
| `~/.claude/settings.json` | Modify | Add PostToolUse + Stop hooks for ruflo; keep RTK PreToolUse/Bash hook |
| `~/.claude-flow/` | Create (via init) | Ruflo CLI config, memory DB, helpers, daemon config |
| `~/.claude-flow/config.json` | Modify after init | Set ruflo-ruvllm base URL to http://localhost:4000 |
| `~/.claude/plugins/cache/ruflo/` | Populate (via plugin installs) | Plugin skill/agent/command files for 6 new plugins |

---

## Task 1: Back up CLAUDE.md

**Files:** Read `~/CLAUDE.md`, write backup to `/tmp/CLAUDE.md.bak`

- [ ] **Step 1: Capture current content**

```bash
cp ~/CLAUDE.md /tmp/CLAUDE.md.bak && echo "Backed up: $(wc -l < /tmp/CLAUDE.md.bak) lines"
```

Expected: `Backed up: 66 lines`

- [ ] **Step 2: Verify backup is intact**

```bash
diff ~/CLAUDE.md /tmp/CLAUDE.md.bak && echo "Backup matches"
```

Expected: `Backup matches` (no diff output)

---

## Task 2: Run Ruflo CLI Init

**Files:** Creates `~/.claude-flow/`, may touch `~/CLAUDE.md`

- [ ] **Step 1: Try init from home directory**

```bash
cd ~ && npx ruflo@latest init 2>&1
```

Expected output includes lines like:
```
✓ Created .claude-flow/
✓ Initialized AgentDB
✓ Memory backend: hybrid
```

If you see `npm error: Could not read package.json`, proceed to Step 1b.

- [ ] **Step 1b (only if Step 1 fails with package.json error): Create stub and retry**

```bash
echo '{}' > ~/package.json && npx ruflo@latest init 2>&1; rm -f ~/package.json
```

- [ ] **Step 2: Restore CLAUDE.md if overwritten**

```bash
diff ~/CLAUDE.md /tmp/CLAUDE.md.bak
```

If diff shows ruflo replaced the file, restore the original:

```bash
cp /tmp/CLAUDE.md.bak ~/CLAUDE.md
```

Then append the ruflo behavioral rules that are genuinely useful (agent routing table, swarm config defaults) from `/tmp/CLAUDE.md.bak`'s new content into `~/CLAUDE.md` under a new `## Ruflo` section. Keep it to <20 lines.

- [ ] **Step 3: Verify .claude-flow created**

```bash
ls ~/.claude-flow/ 2>/dev/null && echo "OK" || echo "MISSING - init may have failed"
```

Expected: directory listing with `config.json` (or `claude-flow.config.json`) present.

- [ ] **Step 4: Commit**

```bash
git -C ~ add CLAUDE.md && git -C ~ diff --cached --stat
git -C ~ commit -m "chore: add ruflo init config (CLAUDE.md merge)"
```

---

## Task 3: Register Ruflo MCP Server

**Files:** Claude Code MCP config (managed by `claude mcp` command)

- [ ] **Step 1: Register the server**

```bash
claude mcp add ruflo -- npx ruflo@latest mcp start
```

Expected: `Added MCP server "ruflo"` (no error)

- [ ] **Step 2: Verify registration**

```bash
claude mcp list
```

Expected output includes:
```
ruflo: npx ruflo@latest mcp start
```

- [ ] **Step 3: Verify server starts and exposes tools**

```bash
npx ruflo@latest mcp start &
sleep 3
npx ruflo@latest mcp tools 2>/dev/null | wc -l || echo "check mcp status manually"
kill %1 2>/dev/null
```

Expected: 100+ lines (300+ tools). If this command syntax doesn't exist, skip — tool count is verified in Task 7 smoke test.

---

## Task 4: Enable Six Plugins in Claude Code

**Files:** `~/.claude/settings.json` (enabledPlugins), `~/.claude/plugins/cache/ruflo/` (populated by slash commands)

The plugin *cache* can only be populated by Claude Code's plugin loader — modify settings.json first to declare intent, then run the slash commands in Claude Code to fetch and cache.

- [ ] **Step 1: Read current settings.json**

```bash
cat ~/.claude/settings.json
```

Confirm `enabledPlugins` currently contains `"ruflo-core@ruflo": true`.

- [ ] **Step 2: Add six plugins to enabledPlugins in settings.json**

In `~/.claude/settings.json`, add these six entries inside `enabledPlugins`:

```json
"ruflo-intelligence@ruflo": true,
"ruflo-cost-tracker@ruflo": true,
"ruflo-ruvllm@ruflo": true,
"ruflo-rag-memory@ruflo": true,
"ruflo-swarm@ruflo": true,
"ruflo-autopilot@ruflo": true
```

The full `enabledPlugins` block becomes:

```json
"enabledPlugins": {
  "vercel@claude-plugins-official": true,
  "rust-analyzer-lsp@claude-plugins-official": true,
  "codex@openai-codex": true,
  "frontend-design@claude-plugins-official": true,
  "superpowers@claude-plugins-official": true,
  "code-simplifier@claude-plugins-official": true,
  "github@claude-plugins-official": true,
  "playwright@claude-plugins-official": true,
  "claude-md-management@claude-plugins-official": true,
  "ralph-loop@claude-plugins-official": true,
  "security-guidance@claude-plugins-official": true,
  "agent-sdk-dev@claude-plugins-official": true,
  "ruflo-core@ruflo": true,
  "ruflo-intelligence@ruflo": true,
  "ruflo-cost-tracker@ruflo": true,
  "ruflo-ruvllm@ruflo": true,
  "ruflo-rag-memory@ruflo": true,
  "ruflo-swarm@ruflo": true,
  "ruflo-autopilot@ruflo": true
}
```

- [ ] **Step 3: Validate JSON is well-formed**

```bash
python3 -c "import json,sys; json.load(open(sys.argv[1])); print('valid')" ~/.claude/settings.json
```

Expected: `valid`

- [ ] **Step 4: Populate plugin cache via Claude Code slash commands**

Run these six commands in a Claude Code session (one at a time — each fetches and caches the plugin files):

```
/plugin install ruflo-intelligence@ruflo
/plugin install ruflo-cost-tracker@ruflo
/plugin install ruflo-ruvllm@ruflo
/plugin install ruflo-rag-memory@ruflo
/plugin install ruflo-swarm@ruflo
/plugin install ruflo-autopilot@ruflo
```

Expected per command: `Installed ruflo-<name>` confirmation.

- [ ] **Step 5: Verify cache populated**

```bash
ls ~/.claude/plugins/cache/ruflo/
```

Expected: directories for each of the 7 plugins (ruflo-core + 6 new ones).

- [ ] **Step 6: Commit settings change**

```bash
git -C ~ add .claude/settings.json
git -C ~ commit -m "chore: enable ruflo intelligence/cost/ruvllm/rag/swarm/autopilot plugins"
```

---

## Task 5: Configure ruflo-ruvllm for LiteLLM :4000

**Files:** `~/.claude-flow/config.json` (or equivalent generated by init)

- [ ] **Step 1: Find the ruflo config file**

```bash
ls ~/.claude-flow/
cat ~/.claude-flow/config.json 2>/dev/null || cat ~/.claude-flow/claude-flow.config.json 2>/dev/null | head -60
```

Note the exact filename for subsequent steps.

- [ ] **Step 2: Check current providers section**

Look for a `providers` or `ruvllm` section in the config. If it exists, locate the local/custom provider entry. If it doesn't exist, it will be added.

- [ ] **Step 3: Set LiteLLM as the local provider**

In `~/.claude-flow/config.json`, add or update the providers block:

```json
{
  "providers": {
    "local": {
      "baseUrl": "http://localhost:4000",
      "apiKey": "ignored",
      "models": ["groq-fast", "cerebras-batch"],
      "type": "openai-compatible"
    }
  },
  "routing": {
    "tier1Provider": "local",
    "tier2Provider": "local",
    "tier3Provider": "anthropic"
  }
}
```

Merge this into the existing config (do not replace the whole file — preserve whatever init created).

- [ ] **Step 4: Verify LiteLLM is reachable**

```bash
curl -s http://localhost:4000/health | python3 -m json.tool | head -10
```

Expected: JSON response showing LiteLLM is running with available models.

- [ ] **Step 5: Commit**

```bash
git -C ~ add .claude-flow/config.json
git -C ~ commit -m "chore: configure ruflo-ruvllm to route tier 1-2 to LiteLLM :4000"
```

---

## Task 6: Wire Hooks in settings.json

**Files:** `~/.claude/settings.json`

Adds two new hook positions (`PostToolUse` for pattern training on file edits, `Stop` for session-end metrics). The existing `PreToolUse/Bash` RTK hook is preserved untouched.

- [ ] **Step 1: Read current hooks section**

```bash
python3 -c "import json; d=json.load(open('$HOME/.claude/settings.json')); print(json.dumps(d.get('hooks',{}), indent=2))"
```

Confirm only `PreToolUse` exists with the RTK Bash hook.

- [ ] **Step 2: Update hooks section in settings.json**

Replace the `hooks` block in `~/.claude/settings.json` with:

```json
"hooks": {
  "PreToolUse": [
    {
      "matcher": "Bash",
      "hooks": [
        {
          "type": "command",
          "command": "rtk hook claude"
        }
      ]
    }
  ],
  "PostToolUse": [
    {
      "matcher": "Edit|Write|MultiEdit",
      "hooks": [
        {
          "type": "command",
          "command": "npx ruflo@latest hooks post-edit --train-patterns 2>/dev/null || true"
        }
      ]
    }
  ],
  "Stop": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "npx ruflo@latest hooks session-end --export-metrics true 2>/dev/null || true"
        }
      ]
    }
  ]
}
```

- [ ] **Step 3: Validate JSON**

```bash
python3 -c "import json,sys; json.load(open(sys.argv[1])); print('valid')" ~/.claude/settings.json
```

Expected: `valid`

- [ ] **Step 4: Spot-check RTK hook is still present**

```bash
python3 -c "
import json
d = json.load(open('$HOME/.claude/settings.json'))
hooks = d['hooks']['PreToolUse']
rtk = any('rtk hook claude' in str(h) for h in hooks)
print('RTK hook present:', rtk)
"
```

Expected: `RTK hook present: True`

- [ ] **Step 5: Commit**

```bash
git -C ~ add .claude/settings.json
git -C ~ commit -m "chore: wire ruflo PostToolUse pattern-training and Stop session-end hooks"
```

---

## Task 7: Import Memory Bridge + Smoke Test

**Files:** AgentDB at `~/.claude-flow/data/` (populated by bridge import)

- [ ] **Step 1: Run ruflo doctor**

```bash
npx ruflo@latest doctor
```

Expected: all checks green. Note any failures — common ones:
- `Daemon: not running` → OK, we want on-demand only
- `API key: missing` → set `ANTHROPIC_API_KEY` if not in env

- [ ] **Step 2: Auto-fix any reported issues**

```bash
npx ruflo@latest doctor --fix
```

Inspect output. Do NOT let it auto-start the daemon (`daemon start` step) — if it tries, cancel and set daemon to manual mode:

```bash
npx ruflo@latest daemon disable --auto-start || true
```

- [ ] **Step 3: Import existing Claude memories into AgentDB**

```bash
npx ruflo@latest memory import-claude --all-projects 2>&1 | tail -10
```

Expected: lines like `Imported N memories from ~/.claude/projects/*/memory/`

If the CLI subcommand name differs, try:

```bash
npx ruflo@latest memory search -q "mayhem oracle" 2>&1 | head -10
```

If that returns results, the bridge is already wired by init.

- [ ] **Step 4: Verify cost-tracker baseline**

```bash
npx ruflo@latest status 2>&1 | head -30
```

Expected: shows token usage stats, provider routing status, and memory backend health.

- [ ] **Step 5: Verify 3-tier routing signal**

```bash
npx ruflo@latest hooks route --task "rename a variable from foo to bar" 2>&1
```

Expected output includes `[TASK_MODEL_RECOMMENDATION]` or tier assignment like `Tier 1 → WASM/Haiku`.

- [ ] **Step 6: Verify LiteLLM routing for tier 1-2**

```bash
npx ruflo@latest providers list 2>&1
```

Expected: `local (http://localhost:4000)` listed as active tier 1/2 provider.

- [ ] **Step 7: Final commit**

```bash
git -C ~ add -A && git -C ~ status
git -C ~ commit -m "chore: ruflo full-stack install complete — memory bridge + smoke test passed"
```

---

## Verification Checklist

All six success criteria from the spec, in order:

| # | Check | Command | Expected |
|---|-------|---------|----------|
| 1 | ruflo doctor clean | `npx ruflo@latest doctor` | All green (daemon OK to be off) |
| 2 | MCP 300+ tools | `claude mcp list` + in-session tool count | ruflo server listed |
| 3 | 3-tier routing active | `npx ruflo@latest hooks route --task "rename var"` | Tier recommendation output |
| 4 | LiteLLM routing | `npx ruflo@latest providers list` | local @ :4000 as tier 1-2 |
| 5 | Cost-tracker baseline | `npx ruflo@latest status` | Token stats visible |
| 6 | RTK undisturbed | `python3 -c "import json; ..."` (Step 6.4) | RTK hook present: True |

---

## Swarm Quick-Start Reference (post-install)

Once installed, spawn the anti-drift coding swarm in any session:

```javascript
// 1. Initialize swarm (MCP tool — one call)
mcp__ruflo__swarm_init({ topology: "hierarchical", maxAgents: 8, strategy: "specialized" })

// 2. Spawn named pipeline agents (all in ONE message, background)
Task({ prompt: "Research requirements. SendMessage findings to 'architect' when done.", subagent_type: "ruflo-core:researcher", name: "researcher", run_in_background: true })
Task({ prompt: "Wait for research. Design solution. SendMessage design to 'coder'.", subagent_type: "ruflo-core:coder", name: "architect", run_in_background: true })
Task({ prompt: "Wait for design. Implement. SendMessage to 'reviewer'.", subagent_type: "ruflo-core:coder", name: "coder", run_in_background: true })
Task({ prompt: "Wait for code. Review quality and security. Report findings.", subagent_type: "ruflo-core:reviewer", name: "reviewer", run_in_background: true })

// 3. Kick off
SendMessage({ to: "researcher", summary: "Start", message: "[task description]" })
```
