# Ruflo Full Stack — Token Savings + Swarm Design

Date: 2026-05-08
Status: Approved

## Goal

Install and configure the full Ruflo CLI stack (`npx ruflo init`) in `~/` to maximize token savings and unlock agentic swarm coordination. Complements the existing RTK Bash-hook (already saving 60–90% on shell output) and the LiteLLM proxy running at `:4000` (Groq/Cerebras).

---

## Token Savings Stack (layered)

| Layer | Tool | Mechanism | Estimated Savings |
|-------|------|-----------|-------------------|
| 0 | RTK hook (active) | Bash output compression | 60–90% on shell |
| 1 | ruflo-intelligence | 3-tier model routing (WASM/Haiku/Sonnet) | 30–50% on tasks |
| 2 | ruflo-ruvllm → LiteLLM `:4000` | Route tier 1–2 to Groq/Cerebras | Replaces Anthropic cost |
| 3 | ReasoningBank + HNSW | Pattern retrieval, no re-reasoning | ~32% |
| 4 | ruflo-rag-memory | Session persistence, no re-explaining context | Qualitative |
| 5 | ruflo-cost-tracker | Per-task budget caps and alerts | Visibility |

---

## Agentic Structure

```
Swarm topology:  hierarchical (anti-drift)
Max agents:      8
Strategy:        specialized
Consensus:       raft (leader maintains authoritative state)
Memory backend:  hybrid (SQLite + AgentDB HNSW)

Pipeline (SendMessage-coordinated, not polling):
  researcher ──► architect ──► coder ──► tester ──► reviewer
                                                        │
                     AgentDB HNSW shared namespace ◄───┘
```

Memory bridge: existing `~/.claude/projects/*/memory/` files imported into AgentDB with 384-dim ONNX embeddings for semantic retrieval across swarms.

Daemon: **on-demand** (not always-running). 12 background workers (audit, optimize, testgaps, etc.) available, triggered when a swarm is spawned.

---

## Installation Plan

### Phase 1 — CLI Init (safe merge)
```bash
npx ruflo@latest init
```
- Run non-interactively or with wizard, using `--skip-claude-md` / manual merge to preserve existing `~/CLAUDE.md`
- Creates `~/.claude-flow/` config and supporting dirs
- Registers Ruflo MCP server: `claude mcp add ruflo -- npx ruflo@latest mcp start`

### Phase 2 — Plugin Installation (6 plugins)
```
/plugin install ruflo-intelligence@ruflo    # 3-tier routing + ReasoningBank
/plugin install ruflo-cost-tracker@ruflo   # per-task budget visibility
/plugin install ruflo-ruvllm@ruflo         # LiteLLM :4000 routing
/plugin install ruflo-rag-memory@ruflo     # session persistence
/plugin install ruflo-swarm@ruflo          # multi-agent coordination
/plugin install ruflo-autopilot@ruflo      # 270s heartbeat, cache-aware
```

### Phase 3 — Configuration
- Configure `ruflo-ruvllm` to use LiteLLM endpoint at `http://localhost:4000`
- Wire hooks in `~/.claude/settings.json`:
  - `session-start` → memory import + HNSW init
  - `route` → 3-tier model selection
  - `post-task` → ReasoningBank pattern learning
  - `post-edit` → train patterns
- Preserve existing RTK `PreToolUse/Bash` hook

### Phase 4 — Smoke Test
```bash
npx ruflo@latest doctor
# Expected: all green
```
- Verify MCP server responds with 300+ tools
- Verify cost-tracker baseline established
- Verify memory bridge imports existing Claude memories

---

## Key Constraints

- **CLAUDE.md must not be overwritten** — merge manually if init tries to replace it
- **RTK hook must be preserved** — it lives in `PreToolUse/Bash`, ruflo hooks go in other positions
- **Daemon: on-demand only** — do not auto-start the background daemon
- **LiteLLM target**: `http://localhost:4000` (already running with Groq/Cerebras)

---

## Agentic Tools Worth Activating

Beyond token savings, these Ruflo tools add agentic power:

| Tool | Value |
|------|-------|
| `ruflo-swarm` + SendMessage | Named agents coordinate via message passing, no polling |
| `ruflo-rag-memory` + HNSW | Sub-ms retrieval of prior project context |
| `ruflo-intelligence` ReasoningBank | Learns successful patterns, improves routing over time |
| `ruflo-autopilot` 270s heartbeat | Stays within Anthropic cache TTL (avoids cold-start cost) |
| Memory bridge | Makes all auto-memory files searchable across sessions |

---

## Success Criteria

1. `npx ruflo@latest doctor` passes all checks
2. MCP server exposes 300+ tools in-session
3. 3-tier routing active: `[TASK_MODEL_RECOMMENDATION]` appears in hook output
4. ruflo-ruvllm routes to `:4000` for tier 1–2 tasks
5. Cost-tracker shows per-task token baselines
6. Existing RTK and LiteLLM setup undisturbed
