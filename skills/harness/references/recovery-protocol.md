# Recovery Protocol

Rules for context.md maintenance, state.jsonl schema, and cross-session recovery.

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
- last_action: "round 11 attempted X, verifier failed, reverted"

## Working Memory
- Observations about the codebase relevant to the task

## Durable Notes
- [dead-end] Parser reorder only masks import bug; fix must target resolver (round 5)
- [constraint] Public API must preserve v1 response shape (round 5)
- [tool-quirk] npm ci broken on node 18 lockfile, use npm install instead (round 7)

## Decisions
- Key decisions with rationale and round number

## Next Steps
- 1-3 concrete actionable items for the next round(s)
```

## Section Semantics

| Section | Contains | Update frequency |
|---|---|---|
| Current State | Phase, round, best result, last action, harness root, worktree | Every round (overwrite) |
| Working Memory | Current-round operative observations | Every round (curate, max 15) |
| Durable Notes | Dead ends, constraints, tool quirks persisting across rounds | When discovered (append, max 10) |
| Decisions | Committed choices with rationale | When made (append only) |
| Next Steps | Immediate priorities | Every round (full rewrite) |

### Durable Notes Rules

- Max 10 items. Each is a one-line claim with evidence reference.
- Categories: `dead-end`, `constraint`, `tool-quirk`, `code-map`.
- Add when knowledge will matter in 3+ future rounds.
- Remove when proven wrong or underlying code/config changed.
- Before proposing, scan for dead ends. Do not re-attempt without new evidence.

## Information Routing

| Question | Answer from |
|---|---|
| What happened in round N? | `state.jsonl` |
| Why did we choose approach X? | `context.md` Decisions |
| What patterns have we noticed? | `context.md` Working Memory |
| What should we do next? | `context.md` Next Steps |
| What constraints/dead-ends persist? | `context.md` Durable Notes |
| Evidence manifest for a round? | `artifacts/round-{N}/manifest.json` |
| Exact output? | `artifacts/round-{N}/` |

`state.jsonl` records facts. `context.md` records meaning, reusable knowledge, and next steps.

---

## State Ledger (state.jsonl)

One UTF-8 JSON object per line, LF line endings, append-only.

### Event Types

**Baseline** — written once after preflight:

```json
{
  "event": "baseline_recorded",
  "task_id": "000-fix-auth-timeout",
  "ts": "2026-04-02T09:01:12Z",
  "round": 0,
  "commit": "a1b2c3d",
  "verification": { "status": "pass" },
  "evaluation": { "result": "baseline" },
  "summary": "initial measurement on task branch"
}
```

**Round** — exactly once per completed round:

```json
{
  "event": "round_completed",
  "task_id": "000-fix-auth-timeout",
  "ts": "2026-04-02T09:18:40Z",
  "round": 4,
  "commit": "m0n1o2p",
  "verification": { "status": "pass" },
  "evaluation": { "result": "keep", "frontier": "improved" },
  "metric": { "value": 76.5, "delta": 4.4 },
  "summary": "batched inference, reduced memory footprint"
}
```

**Disposition** — once when Completion flow resolves the worktree:

```json
{
  "event": "task_disposed",
  "task_id": "000-fix-auth-timeout",
  "ts": "2026-04-02T10:25:00Z",
  "round": 12,
  "disposition": "merged",
  "summary": "squash-merged 12 rounds into main"
}
```

### Required Fields

| Field | Events | Rules |
|---|---|---|
| `event` | all | Event type identifier. |
| `task_id` | all | Stable task identifier. |
| `ts` | all | ISO-8601 UTC. |
| `summary` | all | Short description. |
| `round` | all | Baseline = 0, monotonically increasing. |
| `commit` | baseline, round | Short SHA or `null`. |
| `verification.status` | baseline, round | `pass`, `fail`, `needs_escalation`. |
| `evaluation.result` | baseline, round | `baseline`, `keep`, `discard`, `crash`, `no_op`, `hook_blocked`, `needs_escalation`, `complete`. |
| `disposition` | task_disposed | `merged`, `kept`, `discarded`. |

### Append-Only Rule

Never edit or delete events. One event per round. Corrections go in the next round's `summary`.

### Invariants

1. Exactly one `baseline_recorded` at round `0`.
2. Round numbers strictly increasing by 1.
3. `keep` or `complete` always has `verification.status: pass`.
4. `state.jsonl` is source of truth for facts; `context.md` for meaning.

---

## Recovery Protocol

On session start (fresh agent), execute this sequence:

```
0. Resolve harness root      -> walk up for .harness/, git worktree list, git rev-parse --show-toplevel
1. Resolve current task       -> explicit task_id, .harness-task, sole task, .harness/current-task
2. Verify worktree state      -> confirm in task's worktree, harness root accessible
3. Read config.yaml           -> boundaries, verification, evaluation, stop guards
4. Read context.md            -> phase, progress, durable notes, blockers
5. Read state.jsonl           -> tail recent events
6. Read AGENTS.md             -> protocol constraints
```

After reading, verify these six answers before proceeding:

1. Harness root location
2. Current phase and round
3. Best result so far (value, round, commit)
4. Outstanding issues or blockers
5. Worktree state (correct branch and path)
6. What to do next

If context.md and state.jsonl disagree, resolve from state.jsonl and update context.md.

## Legacy Archive Recovery

When state.jsonl is empty but context.md/artifacts show prior work:

1. Resolve task and worktree state normally.
2. Read config.yaml, context.md, AGENTS.md.
3. Read the smallest artifact set that answers the recovery questions.
4. Rewrite context.md labeling recovered facts.
5. Leave state.jsonl empty. Do not fabricate historical events.
6. Next run records a fresh baseline against current branch state.

## Failure Modes

| Symptom | Fix |
|---|---|
| context.md round != latest state.jsonl round | Set from state.jsonl |
| Next Steps references completed item | Rewrite from current state |
| Working Memory > 15 items | Prune |
| best_result disagrees with state.jsonl | Recompute from state.jsonl |
| Durable Notes > 10 items | Remove least relevant |
