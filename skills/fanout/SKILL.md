---
name: fanout
description: 'Independent subtasks split across agents, or one question needing multiple perspectives (parallel agents, sample N agents, get multiple opinions). Review gates: /audit. Batch changes: /batch.'
argument-hint: <task> [-m split|sample] [-a <type>[:<count>]]... [-b|--fg]
---

Arguments: $ARGUMENTS

- `-m` (default: auto-infer) — `split` (data-parallel, no aggregation) / `sample` (multi-sample, synthesize)
- `-a` (default: `auto`x5) — `N` = N inferred workers; `type` = that type x5; `type:N` = exact. `/codex-exec` or `-a gpt` resolve to gpt.
- `-b` (default: on) — dispatch all workers to background, then block on quorum. `--fg` forces foreground (also when the platform lacks background child support).

## Ownership Boundary

`/fanout` dispatches and samples; not formal code review. Review gates/verdicts/scope adjudication: `/audit`. Use `/fanout` directly for exploration, diagnosis, design sampling, or caller-owned workflows.

## Step 1: Infer mode

- Broad research -> split; narrow/hard -> sample
- Scale to scope: narrow -> 3-5 agents; broad -> 10+. `sample`: 3 usually enough; 5+ for high-stakes architecture

## Step 2: Craft child prompt

- Self-contained prompts required. Common fixes: add context, bound scope, specify deliverable.
- Long/repeated prompts may use a prompt artifact when workers share filesystem access: full prompt in temp/project scratch; child message carries path, read contract, output contract. Keep until quorum; no secrets. Inline if file access is uncertain.
- Match shape to mode:
  - `sample`: same problem, independent stochastic exploration. Define the problem space, not solution paths: background/current state, goal, hard facts/constraints, non-goals, scoring criteria, output shape. Don't prescribe topology, steps, preferred hypotheses, file lists, or sequence unless the user made them hard constraints. Demote solution-shaped phrasing to questions/hypotheses first.
  - `split`: known decomposition. Assigned files, tasks, claims, or data slices.
  - verification/review: anchor on the specific claim, diff, artifact, acceptance criteria.

## Step 3: Dispatch

Dispatch through the active platform's approved agent surface. Use `/codex-exec` for gpt workers only when the platform requires that runner; otherwise use native agent spawn. `/fanout` owns wave shape, quorum, and failure handling, not model/tool defaults.

- Background is default when supported; foreground platforms dispatch one wave. `--fg` forces foreground.
- Recoverable errors (rate limit, network, timeout) -> max 2 retries. Non-recoverable -> no silent local substitution.

## Step 4: Post-processing

**Full quorum required.** While workers run, the main agent only waits — no own investigation, partial synthesis, verification, implementation, or downstream action until every agent returns (success or non-recoverable failure). Early work skews synthesis toward first returners, misses dissent, and contaminates a role that must stay neutral. Verify factual outputs against source.

- split: separate outputs, no aggregation.
- sample: after quorum, synthesize consensus/divergence, evidence alignment, dedup, and unsupported conclusions. Check whether consensus is evidence-backed or prompt-anchored; if shared assumptions collapsed diversity, say so and resample when option discovery matters.
- Partial quorum: only with explicit user approval. State deviation and missing agents first.
