---
name: fanout
description: 'Use when independent subtasks can be split across agents, or when the same question benefits from several independent perspectives (/fanout, fan out, parallel agents, split this across agents, get multiple opinions, or "sample N agents"). Review gates: use /critique (calls /fanout). Worktree/PR batch changes: use /batch.'
argument-hint: <task> [-m split|sample] [-a <type>[:<count>]]... [-b]
---

Arguments: $ARGUMENTS

- `-m` (default: auto-infer) — `split` (data-parallel, no aggregation) / `sample` (multi-sample, synthesize)
- `-a` (default: 5 x `auto`) — number = count of inferred worker type; `type` = that type x5; `type:N` = specific. Valid types include `gpt`. `/codex-exec` in the task argument or `-a gpt` both resolve worker type to `gpt`.
- `-b` (default: off) — background child context

## Step 1: Infer mode

- Broad research -> split; narrow and hard -> sample
- Scale dispatch to scope: few files / narrow question -> 3-5 agents; many files / broad surface -> 10+. `sample`: 3 perspectives usually sufficient; 5+ only for high-stakes architectural decisions

## Step 2: Craft child prompt

- Prompt must be self-contained
- Common mistakes:
  - Missing context ("Fix this race") -> include error message + test name
  - No constraints causing agent to refactor broadly -> add boundaries ("Only modify tests")
  - Vague output ("Fix it") -> specify deliverable ("Return root cause and change summary")

## Step 3: Dispatch

Platform-aware dispatch. Choose the correct path based on current platform and worker type.

**Claude Code:**

- worker type = gpt → chain-load `/codex-exec` via Skill tool first, then dispatch each worker as a Bash `codex exec` command following the `/codex-exec` contract. Do NOT use Agent tool for gpt workers.
- worker type != gpt → dispatch via Agent tool.

**Codex CLI:**

- Spawn agents natively with model `gpt-5.4` reasoning `xhigh` (defaults, unless user explicitly overrides model or effort).

**Error handling (all platforms):**

- Recoverable errors (rate limit, transient network, launch timeout, spawn error with no semantic result): max 2 retries per agent.
- Non-recoverable errors: do not silently substitute with local inline work.

## Step 4: Post-processing

Factual outputs must be verified against source code.

- split: each agent returns output separately, no aggregation
- sample: after all N outputs arrive, run GSA = Generative Self-Aggregation (consensus/divergence, evidence alignment, dedup, discard unsupported conclusions)
