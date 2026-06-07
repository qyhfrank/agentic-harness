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
- Common fixes: add missing context; bound scope; specify deliverable
- Match prompt shape to mode:
  - `sample`: same problem, independent stochastic exploration. Define problem space, not solution paths. Provide background/current state, goal, hard facts/constraints, non-goals, scoring criteria, and output shape. Do not prescribe topology, steps, preferred hypotheses, file lists, or sequence unless the user made them hard constraints.
  - `split`: known decomposition. Assigned files, tasks, claims, or data slices are appropriate.
  - verification/review: anchor on the specific claim, diff, artifact, and acceptance criteria.
- Before dispatching `sample`, demote solution-shaped phrasing to questions or hypotheses. Bound exploration with facts and criteria, not preferred architecture.

## Step 3: Dispatch

Platform-aware dispatch:

**Claude Code:** gpt workers -> chain-load `/codex-exec`, dispatch as `codex exec` Bash commands. Non-gpt -> Agent tool.

**Codex CLI:** native spawn, `gpt-5.5` `xhigh` defaults, 30-min timeout.

**Errors:** recoverable (rate limit, network, timeout) -> max 2 retries. Non-recoverable -> no silent local substitution.

## Step 4: Post-processing

**Full quorum required.** No post-processing, GSA, implementation, or downstream action until every dispatched agent returns (success or non-recoverable failure). Early synthesis skews toward first-returned perspectives and misses dissent. Wait.

Verify factual outputs against source code.

- split: separate outputs, no aggregation
- sample: after quorum, GSA consensus/divergence, evidence alignment, dedup, and unsupported conclusions. Check whether consensus is evidence-backed or prompt-anchored; if shared assumptions collapsed diversity, state it and resample when option discovery matters.
- Partial quorum exception: only with explicit user approval. State deviation and missing agents before acting.
