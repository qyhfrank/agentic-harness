# State Ledger

Schema and conventions for the canonical task ledger: `<harness_root>/.harness/tasks/<task_id>/state.jsonl`.

## Canonical Format

One UTF-8 JSON object per line, LF line endings, append-only.

The canonical ledger is `state.jsonl`. A TSV summary may be generated for human inspection, but it is never the source of truth.

## Schema Version

New events use `schema_version: 2`. Older events without `ts` remain at version 1. A single `state.jsonl` file may contain both v1 and v2 events; readers must parse each line by its own `schema_version`.

## Event Types

### Session Started

Written at the start of each harness session, before preflight or loop entry.

```json
{
  "schema_version": 2,
  "event": "session_started",
  "task_id": "fix-auth-timeout",
  "session_id": "harness-run-20260402-a1b2",
  "ts": "2026-04-02T09:00:00Z",
  "reason": "initial",
  "summary": "fresh session on task branch"
}
```

On resume, include recovery context:

```json
{
  "schema_version": 2,
  "event": "session_started",
  "task_id": "fix-auth-timeout",
  "session_id": "harness-run-20260402-c3d4",
  "ts": "2026-04-02T10:00:00Z",
  "reason": "resume_after_recovery",
  "prev_session_id": "harness-run-20260402-a1b2",
  "resume_round": 5,
  "summary": "resumed after agent restart"
}
```

`reason` values: `initial`, `resume_after_pause`, `resume_after_handoff`, `resume_after_recovery`.

### Session Ended

Written on clean shutdown when the task is not terminal. Optional -- crash means it will not be written; the next `session_started` with a recovery reason covers the gap.

```json
{
  "schema_version": 2,
  "event": "session_ended",
  "task_id": "fix-auth-timeout",
  "session_id": "harness-run-20260402-a1b2",
  "ts": "2026-04-02T09:19:05Z",
  "round": 4,
  "reason": "paused",
  "summary": "user-initiated pause after round 4"
}
```

`reason` values: `paused`, `handoff`, `budget_exhausted`, `stagnation`.

### Baseline Event

Written once after preflight succeeds.

```json
{
  "schema_version": 2,
  "event": "baseline_recorded",
  "task_id": "fix-auth-timeout",
  "session_id": "harness-run-20260402-a1b2",
  "ts": "2026-04-02T09:01:12Z",
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
  "summary": "initial measurement on task branch"
}
```

### Round Event

Written exactly once per completed round.

```json
{
  "schema_version": 2,
  "event": "round_completed",
  "task_id": "fix-auth-timeout",
  "session_id": "harness-run-20260402-a1b2",
  "ts": "2026-04-02T09:18:40Z",
  "round": 4,
  "round_started_at": "2026-04-02T09:01:30Z",
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

`round_started_at` records when this round's work began. If the round started in a previous session that crashed and the start time is unknown, omit the field rather than guessing.

### Disposition Event

Written once when the Completion flow resolves the task's worktree.

```json
{
  "schema_version": 2,
  "event": "task_disposed",
  "task_id": "fix-auth-timeout",
  "session_id": "harness-run-20260402-c3d4",
  "ts": "2026-04-02T10:25:00Z",
  "round": 12,
  "disposition": "merged",
  "target_branch": "main",
  "summary": "squash-merged 12 rounds into main"
}
```

`disposition` values: `merged`, `kept`, `discarded`.

## Required Fields

### v2 Common Envelope (all events)

| Field | Type | Rules |
|---|---|---|
| `schema_version` | integer | `2` for new events. |
| `event` | string | Event type identifier. |
| `task_id` | string | Stable task identifier. |
| `session_id` | string | Format `harness-run-YYYYMMDD-shortid`, aligned with feedback log. |
| `ts` | string | ISO-8601 UTC, e.g. `2026-04-02T09:14:33Z`. Time the event was written. |
| `summary` | string | Short human-readable description. |

### Event-Specific Required Fields

| Field | Events | Rules |
|---|---|---|
| `round` | `baseline_recorded`, `round_completed`, `session_ended`, `task_disposed` | Baseline is `0`. Monotonically increasing by 1 for round events. |
| `commit` | `baseline_recorded`, `round_completed` | Short SHA, or `null` when no commit exists. |
| `verification.status` | `baseline_recorded`, `round_completed` | `pass`, `fail`, or `needs_escalation`. |
| `evaluation.result` | `baseline_recorded`, `round_completed` | `baseline`, `keep`, `discard`, `crash`, `no_op`, `hook_blocked`, `needs_escalation`, `complete`. |
| `reason` | `session_started`, `session_ended` | Why the session started or ended. |
| `disposition` | `task_disposed` | `merged`, `kept`, `discarded`. |

## Optional Fields

| Field | Events | Rules |
|---|---|---|
| `round_started_at` | `round_completed` | ISO-8601 UTC. When this round's work began. Omit if unknown. |
| `prev_session_id` | `session_started` | Previous session for recovery linking. |
| `resume_round` | `session_started` | Round number the session will resume from. |
| `metric.*` | `baseline_recorded`, `round_completed` | `value`, `delta`, `direction`, `confidence`. |
| `implementation.*` | `round_completed` | `protocol`, `red`, `green`. |
| `verification.*` | `baseline_recorded`, `round_completed` | `guards`, `findings`, `provider`. |
| `evaluation.*` | `baseline_recorded`, `round_completed` | `objective_met`, `frontier`, `close_authority`. |
| `artifacts` | any | List of artifact paths. |
| `error` | any | Error details. |
| `branch` | any | Git branch name. |
| `environment_fingerprint` | any | Environment identifier. |

## Session ID Format

Use `harness-run-YYYYMMDD-shortid` to align with the feedback event log. The date is the session start date; `shortid` is a 4-character random hex suffix. Example: `harness-run-20260402-a1b2`.

## Timing Semantics

- `ts` records when the event was **written** to `state.jsonl`. It is wall-clock metadata, not a causal ordering mechanism.
- File append order remains the causal ordering authority.
- `round_started_at` on `round_completed` enables round duration calculation: `ts - round_started_at`.
- Total task wall-clock time is derived: `task_disposed.ts - first(session_started.ts)`.
- Session gaps are derived: compare `session_started.ts` against the previous event's `ts`.
- If timing data is unavailable (crash, v1 legacy), omit the field. Never fabricate timestamps.

## Append-Only Rule

Never edit or delete existing events. Each completed baseline or round appends exactly one event. If a round must be corrected, append a new event for the next round that records the correction in `summary`.

## Reading Conventions

All paths below use `<harness_root>` as the resolved harness root path (see SKILL.md Harness Root).

| Need | Command |
|---|---|
| Latest event | `tail -1 <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Recent N events | `tail -n N <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| All non-keep results | `jq -c 'select(.evaluation.result != "keep" and .evaluation.result != "baseline")' <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Best metric | `jq -s 'map(select(.metric.value != null)) | max_by(.metric.value)' <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Specific round | `jq -c 'select(.round == N)' <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Escalations | `jq -c 'select(.verification.status == "needs_escalation" or .evaluation.result == "needs_escalation")' <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Session events | `jq -c 'select(.event == "session_started" or .event == "session_ended")' <harness_root>/.harness/tasks/<task_id>/state.jsonl` |
| Round timing | `jq -c 'select(.event == "round_completed") | {round, ts, round_started_at}' <harness_root>/.harness/tasks/<task_id>/state.jsonl` |

## Backward Compatibility

- v1 events (no `ts`, no `session_id`) remain valid. Do not backfill or rewrite them.
- Readers must handle mixed v1/v2 files by checking `schema_version` per line.
- For v1 events, timing is `unknown`. Do not infer timestamps from file mtime or line position.
- When a v2 writer takes over an existing v1 task, it emits `session_started` with a recovery reason before proceeding. Timing coverage begins from that point.

## Derived Summary View

If a human-friendly TSV is helpful, generate it from `state.jsonl`. Do not read it during recovery or use it to drive runtime decisions.

## Invariants

1. Exactly one `baseline_recorded` event exists and it is always round `0`.
2. Round numbers are strictly increasing by 1 across `round_completed` events.
3. Every `keep` or `complete` result has `verification.status: pass`.
4. Every `discard` result has either `verification.status: fail` or an evaluation reason recorded in `summary`.
5. `state.jsonl` is the structured source of truth for facts; `context.md` records meaning and next steps.
6. Every v2 event has `ts` and `session_id`. Missing `ts` means v1.
7. A `session_started` without a later `session_ended` in the same `session_id` indicates an unclean exit. The next `session_started` clarifies the gap via its `reason`.
