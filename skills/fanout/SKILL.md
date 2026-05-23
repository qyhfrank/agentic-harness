---
name: fanout
description: 'Use when independent subtasks can be split across agents, or when the same question benefits from several independent perspectives (/fanout, fan out, parallel agents, split this across agents, get multiple opinions, or "sample N agents"). Review gates: use /critique (calls /fanout). Worktree/PR batch changes: use /batch.'
argument-hint: <task> [-m split|sample] [-a <type>[:<count>]]... [-b]
---

Arguments: $ARGUMENTS

- `-m` (default: auto-infer) — `split` (data-parallel, no aggregation) / `sample` (multi-sample, synthesize)
- `-a` (default: 5 x `auto`) — number = count of inferred worker type; `type` = that type x5; `type:N` = specific. `/codex-exec` or `-a gpt` both resolve to gpt type.
- `-b` (default: off) — background child context

## Ownership Boundary

`/fanout` dispatches and samples; not formal code review. Review gates/verdicts/scope adjudication: use `/critique` (calls `/fanout -m sample`). `/fanout` directly for exploration, diagnosis, design sampling, or caller-owned workflows.

## Step 1: Infer mode

- Broad research -> split; narrow/hard -> sample
- Scale to scope: few files/narrow -> 3-5 agents; broad surface -> 10+. `sample`: 3 usually sufficient; 5+ for high-stakes architecture

## Step 2: Craft child prompt

- Self-contained prompts required
- Common mistakes: missing context -> include error + test name; unconstrained scope -> add boundaries; vague output -> specify deliverable
- `sample` thrives on divergent paths. Provide problem, symptoms, environment context; leave investigation strategy to each worker. Shared hypotheses and pre-assigned file lists collapse diversity into N copies of the same analysis. Ground with facts (config values, errors, paths); avoid steering the approach.

## Step 3: Dispatch

Platform-aware dispatch:

**Claude Code:** gpt workers -> chain-load `/codex-exec`, dispatch as `codex exec` Bash commands. Non-gpt -> Agent tool.

**Codex CLI:** native spawn, `gpt-5.5` `xhigh` defaults, 30-min timeout.

**Errors:** recoverable (rate limit, network, timeout) -> max 2 retries. Non-recoverable -> no silent local substitution.

## Step 4: Post-processing

**Full quorum required.** No post-processing, GSA, implementation, or downstream action until every dispatched agent returns (success or non-recoverable failure). Early synthesis skews toward first-returned perspectives and misses dissent. Wait.

Verify factual outputs against source code.

- split: separate outputs, no aggregation
- sample: GSA (consensus/divergence, evidence alignment, dedup, discard unsupported conclusions) after quorum
- Partial quorum exception: only with explicit user approval. State deviation and missing agents before acting.
