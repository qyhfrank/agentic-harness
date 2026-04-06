# Context Protocol

Maintenance rules for `.harness/tasks/<task_id>/context.md` and cross-session recovery.

## context.md Structure

```markdown
# Harness Context

## Current State
- phase: B (run)
- round: 12 / 50
- harness_root: /absolute/path/to/repo
- worktree: <task_slug> branch, path /absolute/path/.claude/worktrees/<task_slug>/
- current_objective: "reduce inference latency below 200ms"
- best_result: "76.5 latency score (round 9, commit abc1234)"
- last_action: "round 11 attempted X, verifier failed (lint regression), reverted"
- active_discoveries: 6 active / 0 needs_recheck
- session_id: harness-run-20260402-a1b2
- session_started: 2026-04-02T09:00:00Z
- task_started: 2026-04-02T08:30:00Z
- last_round_started: 2026-04-02T09:45:00Z
- timing_coverage: full since round 3 (rounds 0-2 predate v2)

## Working Memory
- Observations about the codebase relevant to the task
- Patterns noticed across rounds
- Dead ends to avoid

## Decisions
- Key decisions made during plan phase and run phase
- Rationale for choices

## Next Steps
- Immediate priorities for the next round(s)
```

## Section Semantics

| Section | Contains | Update frequency |
|---|---|---|
| Current State | Phase, round counter, best result, last action, harness root, worktree info, session and timing anchors, discovery summary | Every round (overwrite) |
| Working Memory | Current-round operative observations, discovery ID references | Every round (curate) |
| Decisions | Committed choices with rationale | When a decision is made (append) |
| Next Steps | Immediate priorities for upcoming rounds | Every round (full rewrite) |

## Update Discipline

### Current State
Overwrite all fields every round. Fields must reflect the state after the round completes, not before. When `discovery.md` has active entries, include `active_discoveries` (see `discovery-protocol.md`).

### Working Memory
- Add new observations discovered during the round.
- Remove or compress entries that are no longer relevant.
- Move settled observations to Decisions when they become committed choices.
- Promote durable cross-round knowledge to `discovery.md` per `discovery-protocol.md`. Replace promoted items with an ID reference (e.g., `→ See env-003`).
- Target: 5-15 bullet points. If it exceeds 15, prune aggressively. Discovery ID references count as items but are shorter, freeing space for fresh observations.
- Do not repeat `config.yaml` contract values, detailed evidence tables, or full durable discovery content here. Point to the canonical file instead.

### Decisions
- Append only. Do not edit or remove past decisions.
- Each entry: what was decided, why, and which round.
- Move here from Working Memory once a hypothesis becomes a committed approach.

### Next Steps
- Full rewrite every round. This is not a backlog.
- 1-3 concrete, actionable items for the next round(s).
- If blocked, state the blocker and what is needed to unblock.

## Artifact Storage

Every round's verification output goes to `<harness_root>/.harness/tasks/<task_id>/artifacts/round-{N}/`. Store:
- Full verification command stdout/stderr
- Diff of changes attempted
- Review outputs referenced by the evaluator
- Any diagnostic output referenced in context.md

Do not store artifacts inline in context.md. Reference by path.

Non-canonical helper files such as task `README.md`, `plan.md`, `design.md`, or
extra reference notes are optional. They must not become required for normal
recovery. Create them only when the canonical files cannot answer the recovery
or execution question without repeated costly reconstruction.

## Division of Labor

| Question | Answer from |
|---|---|
| What happened in round N? | `state.jsonl` |
| What was the metric trend? | `state.jsonl` |
| Why did we choose approach X? | `context.md` Decisions |
| What patterns have we noticed? | `context.md` Working Memory |
| What should we do next? | `context.md` Next Steps |
| What was the exact output? | `artifacts/round-{N}/` |
| When did round N happen? | `state.jsonl` (`ts`, `round_started_at`) |
| How long did a session/task take? | `state.jsonl` (derived from `session_started`/`session_ended`/`task_disposed` timestamps) |
| What is the current session and timing coverage? | `context.md` Current State (timing anchors for quick recovery orientation) |
| What did the user redirect or correct mid-run? | `state.jsonl` (`user_directive` events) |
| What constraints, dead ends, quirks, and patterns persist across rounds? | `discovery.md` |

`state.jsonl` records facts. `context.md` records meaning. `discovery.md` records reusable knowledge. Never duplicate across them; reference by ID or round number.

Practical routing test:

- contract, thresholds, boundaries, execution policy -> `config.yaml`
- current state, blocker, next move -> `context.md`
- durable reusable fact -> `discovery.md`
- raw output, logs, patches, proof -> `artifacts/`

## Recovery Protocol

On session start (fresh agent, no prior context in conversation), execute this sequence:

```
0. Resolve harness root                  -> walk up for .harness/, git worktree list, or git rev-parse --show-toplevel
1. Resolve current task (local-first)    -> explicit task_id, <worktree_root>/.harness-task, unique branch/task_slug, sole task, <harness_root>/.harness/current-task
2. Verify worktree state                 -> confirm session is in the task's worktree; re-enter if needed. Confirm harness root is accessible.
3. Read task config.yaml                 -> task definition, boundaries, verification, evaluation, stop guards
4. Read task context.md                  -> phase, progress, learnings, blockers
4a. If discovery.md exists:
    -> Read Snapshot and Active Index
    -> Expand entries listed in read_first or referenced by current objective/blocker
    -> Do NOT read full file or Archived section unless specifically needed
5. Read task state.jsonl                 -> tail recent events, full scan if needed
                                            -> scan for user_directive events; note any active redirections that may still apply to the current approach
6. Read AGENTS.md                        -> protocol constraints, repo conventions
```

Do not read `README.md`, `plan.md`, `design.md`, or extra reference notes during
normal recovery unless steps 3-6 still leave one of the required recovery
answers missing.

Conflict rules for step 1:
- If `.harness-task` and branch-derived match disagree, this is a repair condition — stop and ask for an explicit task identifier. Do not proceed with either value. After the user confirms, rewrite `.harness-task` before continuing.
- If inside a worktree but local signals (`.harness-task`, branch match) both fail, check whether tasks exist. If no tasks exist yet, proceed to scaffold normally. If tasks exist, this is a repair condition — stop and ask for an explicit task identifier. Do not fall through to `sole task` or `current-task` from inside a worktree when tasks exist but local binding failed.
- If `.harness-task` does not exist but branch matching succeeds, opportunistically write `.harness-task` for future resilience (only in harness-managed worktrees, not in the main checkout when running in-place).

After reading, verify recovery by confirming these six answers before proceeding:

1. Harness root location and accessibility
2. Current phase and round number
3. Best result so far (metric value or acceptance state, round, commit)
4. Outstanding issues or blockers
5. Worktree state (in expected worktree and branch, or needs re-entry)
6. What to do next (from Next Steps)

If any answer is missing or contradictory between context.md and `state.jsonl`, resolve from `state.jsonl` (structured source of truth for facts) and update context.md before continuing.

## Legacy Archive Recovery

Use this path when all of the following are true:

- `state.jsonl` is empty
- the task config is already finalized
- `context.md`, `design.md`, `plan.md`, or `artifacts/` clearly show that the
  task has prior work from before ledger discipline

Interpretation: the task is resumable, but its durable history lives in archived
docs rather than structured ledger events. Recover enough state to continue
safely without rewriting history.

### Legacy Recovery Order

1. Resolve the task and verify worktree state as in normal recovery.
2. Read `config.yaml`, `context.md`, and `AGENTS.md`.
3. Read the smallest archived artifact set that can answer the five recovery
   questions. Prefer an artifact index or compressed history note over raw logs.
4. Rewrite `context.md` so recovered facts are explicitly labeled, for example
   `Recovered from artifacts: ...`.
5. Leave `state.jsonl` untouched. Do not fabricate historical
   `baseline_recorded` or `round_completed` events.
6. When the next real run begins, record a fresh baseline event against the
   current branch state. Mention archival recovery in the new baseline summary
   only if that context helps explain the starting point.

## Recovery Smoke Test

A fresh agent reading only `AGENTS.md` + task-scoped `.harness/` state must accurately answer:

1. Harness root location
2. Current phase and round
3. Best result so far and its commit
4. Outstanding issues and blockers
5. Worktree state (correct branch and path)
6. What to do next

When `discovery.md` exists, also answer:

7. What hard constraints limit proposals?
8. What approaches are known dead ends?
9. What tool/verification quirks affect execution?
10. What is the key module structure for this task?

If questions 1-6 cannot be answered, context.md is stale or incomplete. If questions 7-10 cannot be answered but discovery.md has active entries, the Snapshot or read_first is wrong. Fix before running the next round.

## Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| context.md round != latest round event | Missed update | Set context.md round from `state.jsonl` |
| Next Steps references a completed item | Stale rewrite | Rewrite Next Steps from current state |
| Working Memory > 15 items | Insufficient pruning | Compress or move to Decisions |
| best_result in context.md disagrees with `state.jsonl` frontier | Missed update | Recompute from `state.jsonl` |
| Decisions contradict Working Memory | Hypothesis promoted without cleanup | Remove from Working Memory |
| `state.jsonl` is empty but the task clearly has prior history | Pre-ledger task archive or missed ledger adoption | Recover from archived artifacts, label recovered facts in `context.md`, and let the next real run record a fresh baseline |
