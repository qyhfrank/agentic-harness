# State Ledger

Schema and conventions for the canonical task ledger: `.harness/tasks/<task_id>/state.jsonl`.

## Canonical Format

One UTF-8 JSON object per line, LF line endings, append-only.

The canonical ledger is `state.jsonl`. A TSV summary may be generated for human inspection, but it is never the source of truth.

## Event Types

Use the smallest set of event shapes that covers current runtime behavior:

### Baseline Event

Written once after preflight succeeds.

```json
{
  "schema_version": 1,
  "event": "baseline_recorded",
  "task_id": "fix-auth-timeout",
  "round": 0,
  "commit": "a1b2c3d",
  "metric": {
    "value": 72.3,
    "delta": null,
    "direction": "increase"
  },
  "verification": {
    "status": "pass",
    "guards": ["pass"]
  },
  "evaluation": {
    "result": "baseline",
    "frontier": "initial"
  },
  "artifacts": [],
  "summary": "initial measurement on main"
}
```

### Round Event

Written exactly once per completed round.

```json
{
  "schema_version": 1,
  "event": "round_completed",
  "task_id": "fix-auth-timeout",
  "round": 4,
  "commit": "m0n1o2p",
  "verification": {
    "status": "pass",
    "guards": ["pass"],
    "findings": []
  },
  "evaluation": {
    "result": "keep",
    "objective_met": false,
    "frontier": "improved"
  },
  "metric": {
    "value": 76.5,
    "delta": 4.4,
    "direction": "increase",
    "confidence": 1.8
  },
  "artifacts": [
    ".harness/tasks/fix-auth-timeout/artifacts/round-4/verify.txt"
  ],
  "summary": "batched inference, reduced memory footprint"
}
```

## Required Fields

| Field | Type | Rules |
|---|---|---|
| `schema_version` | integer | Start at `1`. Increment only for breaking ledger changes. |
| `event` | string | `baseline_recorded` or `round_completed` for MVP. |
| `task_id` | string | Stable task identifier. |
| `round` | integer | Baseline is `0`. Monotonically increasing by 1 for round events. |
| `commit` | string or null | Short SHA for kept/discarded work, `null` when no commit exists. |
| `verification.status` | enum | `pass`, `fail`, or `needs_escalation`. |
| `evaluation.result` | enum | `baseline`, `keep`, `discard`, `crash`, `no_op`, `hook_blocked`, `needs_escalation`, `complete`. |
| `summary` | string | Short human-readable description of what happened. |

## Optional Fields

Use optional nested fields instead of inventing new top-level columns:

- `metric.value`, `metric.delta`, `metric.direction`, `metric.confidence`
- `verification.guards`, `verification.findings`, `verification.provider`
- `evaluation.objective_met`, `evaluation.frontier`, `evaluation.close_authority`
- `artifacts`
- `error`
- `branch`
- `environment_fingerprint`

## Append-Only Rule

Never edit or delete existing events. Each completed baseline or round appends exactly one event. If a round must be corrected, append a new event for the next round that records the correction in `summary`.

## Reading Conventions

| Need | Command |
|---|---|
| Latest event | `tail -1 .harness/tasks/<task_id>/state.jsonl` |
| Recent N events | `tail -n N .harness/tasks/<task_id>/state.jsonl` |
| All non-keep results | `jq -c 'select(.evaluation.result != "keep" and .evaluation.result != "baseline")' .harness/tasks/<task_id>/state.jsonl` |
| Best metric | `jq -s 'map(select(.metric.value != null)) | max_by(.metric.value)' .harness/tasks/<task_id>/state.jsonl` |
| Specific round | `jq -c 'select(.round == N)' .harness/tasks/<task_id>/state.jsonl` |
| Escalations | `jq -c 'select(.verification.status == "needs_escalation" or .evaluation.result == "needs_escalation")' .harness/tasks/<task_id>/state.jsonl` |

## Derived Summary View

If a human-friendly TSV is helpful, generate it from `state.jsonl`. Do not read it during recovery or use it to drive runtime decisions.

## Invariants

1. Exactly one `baseline_recorded` event exists and it is always round `0`.
2. Round numbers are strictly increasing by 1 across `round_completed` events.
3. Every `keep` or `complete` result has `verification.status: pass`.
4. Every `discard` result has either `verification.status: fail` or an evaluation reason recorded in `summary`.
5. `state.jsonl` is the structured source of truth for facts; `context.md` records meaning and next steps.
