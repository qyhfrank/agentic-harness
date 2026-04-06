# State Ledger

Schema for the canonical task ledger: `<harness_root>/.harness/tasks/<task_id>/state.jsonl`.

## Format

One UTF-8 JSON object per line, LF line endings, append-only.

## Event Types

Three event types. Each is written exactly once at its lifecycle point.

### Baseline Event

Written once after preflight succeeds.

```json
{
  "event": "baseline_recorded",
  "task_id": "000-fix-auth-timeout",
  "ts": "2026-04-02T09:01:12Z",
  "round": 0,
  "commit": "a1b2c3d",
  "verification": { "status": "pass" },
  "evaluation": { "result": "baseline" },
  "metric": { "value": 72.3, "direction": "increase" },
  "summary": "initial measurement on task branch"
}
```

### Round Event

Written exactly once per completed round.

```json
{
  "event": "round_completed",
  "task_id": "000-fix-auth-timeout",
  "ts": "2026-04-02T09:18:40Z",
  "round": 4,
  "commit": "m0n1o2p",
  "verification": { "status": "pass" },
  "evaluation": { "result": "keep", "frontier": "improved" },
  "metric": { "value": 76.5, "delta": 4.4, "direction": "increase" },
  "summary": "batched inference, reduced memory footprint"
}
```

### Disposition Event

Written once when the Completion flow resolves the task's worktree.

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

`disposition` values: `merged`, `kept`, `discarded`.

## Required Fields

| Field | Events | Rules |
|---|---|---|
| `event` | all | Event type identifier. |
| `task_id` | all | Stable task identifier. |
| `ts` | all | ISO-8601 UTC. Time the event was written. |
| `summary` | all | Short human-readable description. |
| `round` | all | Baseline is `0`. Monotonically increasing by 1 for round events. |
| `commit` | `baseline_recorded`, `round_completed` | Short SHA, or `null` when no commit exists. |
| `verification.status` | `baseline_recorded`, `round_completed` | `pass`, `fail`, or `needs_escalation`. |
| `evaluation.result` | `baseline_recorded`, `round_completed` | `baseline`, `keep`, `discard`, `crash`, `no_op`, `hook_blocked`, `needs_escalation`, `complete`. |
| `disposition` | `task_disposed` | `merged`, `kept`, `discarded`. |

## Optional Fields

| Field | Events | Rules |
|---|---|---|
| `metric.*` | `baseline_recorded`, `round_completed` | `value`, `delta`, `direction`. |
| `evaluation.*` | `baseline_recorded`, `round_completed` | `objective_met`, `frontier`. |
| `artifacts` | any | List of artifact paths. |
| `error` | any | Error details. |

## Append-Only Rule

Never edit or delete existing events. Each completed baseline or round appends exactly one event. If a round must be corrected, append a new event for the next round that records the correction in `summary`.

## Reading Conventions

| Need | Command |
|---|---|
| Latest event | `tail -1 <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Recent N events | `tail -n N <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| All non-keep results | `jq -c 'select(.evaluation.result != "keep" and .evaluation.result != "baseline")' state.jsonl` |
| Best metric | `jq -s 'map(select(.metric.value != null)) \| max_by(.metric.value)' state.jsonl` |
| Specific round | `jq -c 'select(.round == N)' state.jsonl` |

## Invariants

1. Exactly one `baseline_recorded` event exists and it is always round `0`.
2. Round numbers are strictly increasing by 1 across `round_completed` events.
3. Every `keep` or `complete` result has `verification.status: pass`.
4. Every `discard` result has either `verification.status: fail` or an evaluation reason in `summary`.
5. `state.jsonl` is the structured source of truth for facts; `context.md` records meaning and next steps.
