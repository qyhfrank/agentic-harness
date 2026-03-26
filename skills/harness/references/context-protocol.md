# Context Protocol

Maintenance rules for `.harness/tasks/<task_id>/context.md` and cross-session recovery.

## context.md Structure

```markdown
# Harness Context

## Current State
- phase: B (run)
- round: 12 / 50
- current_objective: "reduce inference latency below 200ms"
- best_result: "76.5 latency score (round 9, commit abc1234)"
- last_action: "round 11 attempted X, verifier failed (lint regression), reverted"

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
| Current State | Phase, round counter, best result, last action | Every round (overwrite) |
| Working Memory | Codebase observations, patterns, dead ends | Every round (curate) |
| Decisions | Committed choices with rationale | When a decision is made (append) |
| Next Steps | Immediate priorities for upcoming rounds | Every round (full rewrite) |

## Update Discipline

### Current State
Overwrite all fields every round. Fields must reflect the state after the round completes, not before.

### Working Memory
- Add new observations discovered during the round.
- Remove or compress entries that are no longer relevant.
- Move settled observations to Decisions when they become committed choices.
- Target: 5-15 bullet points. If it exceeds 15, prune aggressively.

### Decisions
- Append only. Do not edit or remove past decisions.
- Each entry: what was decided, why, and which round.
- Move here from Working Memory once a hypothesis becomes a committed approach.

### Next Steps
- Full rewrite every round. This is not a backlog.
- 1-3 concrete, actionable items for the next round(s).
- If blocked, state the blocker and what is needed to unblock.

## Artifact Storage

Every round's verification output goes to `.harness/tasks/<task_id>/artifacts/round-{N}/`. Store:
- Full verification command stdout/stderr
- Diff of changes attempted
- Review outputs referenced by the evaluator
- Any diagnostic output referenced in context.md

Do not store artifacts inline in context.md. Reference by path.

## Division of Labor

| Question | Answer from |
|---|---|
| What happened in round N? | `state.jsonl` |
| What was the metric trend? | `state.jsonl` |
| Why did we choose approach X? | `context.md` Decisions |
| What patterns have we noticed? | `context.md` Working Memory |
| What should we do next? | `context.md` Next Steps |
| What was the exact output? | `artifacts/round-{N}/` |

`state.jsonl` records facts. `context.md` records meaning. Never duplicate structured event payloads from `state.jsonl` into context.md; summarize them.

## Recovery Protocol

On session start (fresh agent, no prior context in conversation), execute this sequence:

```
1. Resolve current task                  -> explicit id, branch, .harness/current-task, or sole task
2. Read task config.yaml                 -> task definition, boundaries, verification, evaluation, stop guards
3. Read task context.md                  -> phase, progress, learnings, blockers
4. Read task state.jsonl                 -> tail recent events, full scan if needed
5. Read AGENTS.md                        -> protocol constraints, repo conventions
```

After reading, verify recovery by confirming these four answers before proceeding:

1. Current phase and round number
2. Best result so far (metric value or acceptance state, round, commit)
3. Outstanding issues or blockers
4. What to do next (from Next Steps)

If any answer is missing or contradictory between context.md and `state.jsonl`, resolve from `state.jsonl` (structured source of truth for facts) and update context.md before continuing.

## Recovery Smoke Test

A fresh agent reading only `AGENTS.md` + task-scoped `.harness/` state must accurately answer:

1. Current phase and round
2. Best result so far and its commit
3. Outstanding issues and blockers
4. What to do next

If it cannot, context.md is stale or incomplete. Fix before running the next round.

## Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| context.md round != latest round event | Missed update | Set context.md round from `state.jsonl` |
| Next Steps references a completed item | Stale rewrite | Rewrite Next Steps from current state |
| Working Memory > 15 items | Insufficient pruning | Compress or move to Decisions |
| best_result in context.md disagrees with `state.jsonl` frontier | Missed update | Recompute from `state.jsonl` |
| Decisions contradict Working Memory | Hypothesis promoted without cleanup | Remove from Working Memory |
