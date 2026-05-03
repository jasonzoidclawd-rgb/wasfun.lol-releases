# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository shape
- The functional codebase is currently centered on a single script: `usage-hud.py`.
- `CLAUDE.md` contains behavioral constraints that should be treated as project rules for how changes are made.
- No dedicated app package layout, CI config, or formal test/lint/build tooling is currently defined at repo root.

## Commands used in this repo
### Environment / dependencies
- `python3 -m pip install rich`
- `npx ccusage weekly -j`

## Run
- `python3 usage-hud.py`

## Validation
- Syntax check: `python3 -m py_compile usage-hud.py`
- Tests: no test suite is currently configured in this repository.
- Run a single test: not available yet (no test framework or test files are present).

## High-level architecture
- `usage-hud.py` is a terminal UI loop built with Rich (`Live`, `Table`, `Panel`).
- Runtime flow:
  1. Collect usage metrics from three sources:
     - Codex session JSONL logs from `~/.codex/sessions/**/*.jsonl`
     - Claude context usage snapshot from `/tmp/claude_usage.json`
     - Claude weekly spend via `npx ccusage weekly -j`
  2. Transform raw usage into percentages/colors (`get_vibrant_color`, budget normalization).
  3. Render four 3-line “thick” progress bars in a single panel (`generate_hud` + `create_thick_bar`).
  4. Refresh continuously in a 2-second loop under `Live`.
- Error handling strategy is intentionally permissive: most data collection helpers swallow exceptions and fall back to `0.0` so the HUD keeps rendering even when one source is missing.
- The only configurable constant in code today is `CLAUDE_WEEKLY_BUDGET`; changing budget behavior currently requires editing this value directly.

## Important inherited rules (from `CLAUDE.md`)
- Think before coding: make assumptions explicit and ask when ambiguous.
- Keep solutions minimal; avoid speculative abstractions/features.
- Make surgical edits only; avoid unrelated refactors.
- Define verifiable success criteria for multi-step work before implementation.
