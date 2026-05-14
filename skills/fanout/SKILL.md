---
name: fanout
description: 'Use when independent subtasks can be split across agents, or when the same question benefits from several independent perspectives (/fanout, fan out, parallel agents, split this across agents, get multiple opinions, or "sample N agents"). Review gates: use /critique (calls /fanout). Worktree/PR batch changes: use /batch.'
argument-hint: <task> [-m split|sample] [-a <type>[:<count>]]... [-b]
---

Arguments: $ARGUMENTS

- `-m` (default: auto-infer) — `split` (data-parallel, no aggregation) / `sample` (multi-sample, synthesize)
- `-a` (default: 5 x `auto`) — number = count of inferred worker type; `type` = that type x5; `type:N` = specific. Valid types include `gpt`. `/codex-exec` in the task argument or `-a gpt` both resolve worker type to `gpt`.
- `-b` (default: off) — background child context

## Ownership Boundary

`/fanout` dispatches and samples; it does not own formal code review. Code-review gates, verdicts, scope adjudication → `/critique` (calls `/fanout -m sample`). Use `/fanout` directly for exploration, diagnosis, design sampling, or caller-owned workflows.

## Step 1: Infer mode

- Broad research → split; narrow/hard → sample.
- Scale to scope: narrow question → 3-5 agents; broad surface → 10+. `sample`: 3 perspectives default; 5+ only for high-stakes architecture.

## Step 2: Craft child prompt

Self-contained. Include: error context, scope boundaries, output format. Vague prompts ("fix this race", "fix it") → add error message, test name, deliverable spec.

## Step 3: Dispatch

Platform-aware dispatch by worker type. Default timeout: 30 minutes per agent (all platforms).

**Claude Code:** gpt workers → chain-load `/codex-exec`, dispatch as Bash `codex exec` (not Agent tool). Other workers → Agent tool.

**Codex CLI:** model `gpt-5.5` reasoning `xhigh` by default.

**Errors:** recoverable (rate limit, transient, spawn) → max 2 retries. Non-recoverable → surface; never silently substitute with local inline work.

## Step 4: Post-processing

**Full quorum required.** No post-processing, GSA, implementation, or downstream action until every dispatched agent returns (success or non-recoverable failure). Early synthesis skews toward first-returned perspectives and misses dissent. Wait.

Verify factual outputs against source code.

- **split:** return outputs separately, no aggregation.
- **sample:** run GSA (consensus/divergence, evidence alignment, dedup, discard unsupported).
- **storage:** compact `fanout-gsa.md`; raw outputs kept only for audit.
- **partial quorum exception:** only with explicit user approval. State deviation and missing agents before acting.
