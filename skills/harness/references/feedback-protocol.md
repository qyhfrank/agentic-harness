# Feedback Protocol

Skill-level feedback: detect when harness protocols guide poorly, not when the target code is bad.

## Storage

```
~/.asb/state/harness-feedback/
  events.jsonl        # append-only, one JSON object per line
  candidates.json     # reducer output; rebuildable from events.jsonl
```

Create the directory on first append. Never store durable feedback in `.harness/` (task-scoped) or skill references (declarative).

## Structured Note (Agent Interface)

Agents do NOT write events directly. At each phase boundary, write a structured note in the current task directory:

```json
{
  "uncertainties": [],
  "workarounds": [],
  "needs_review": []
}
```

Rules:
- `uncertainties`: protocol was ambiguous or silent on a decision the agent had to make
- `workarounds`: agent had to deviate from protocol to succeed
- `needs_review`: agent followed protocol but suspects the outcome is suboptimal
- If all three arrays are empty, skip the note entirely (no noise)
- Each entry is a single sentence describing the specific situation

## Event Schema

The system (not the agent) expands notes into events. External facts also generate events automatically.

```json
{
  "ts": "ISO-8601",
  "run_id": "harness-run-YYYYMMDD-shortid",
  "round_ref": "scaffold | plan-finalize | run-round-N | session-end",
  "subject_ref": "harness.<phase>.<topic>",
  "subject_artifact": "references/<file>.md#<section>",
  "subject_version": "git:<7-char-sha>",
  "phase": "scaffold | plan | run",
  "source": "agent | grader | human",
  "fault_domain": "subject | repo | runtime | task | unknown",
  "kind": "<base-kind>",
  "summary": "one sentence",
  "evidence_refs": [],
  "payload": {}
}
```

## Base Kinds (MVP)

| Kind | Meaning | Example |
|---|---|---|
| `bad_default` | Protocol template or default led to wrong value | `objective: minimize` not in vocabulary |
| `protocol_gap` | Protocol was silent on a situation agent encountered | No guidance on monorepo sub-package detection |
| `boundary_error` | Mutable or immutable boundary was wrong | `payload.direction: underfit | overfit` |
| `repeat_loop` | Same failure signature appeared 2+ times in one run | `payload.signature: <normalized error>` |
| `workaround_needed` | Agent deviated from protocol to succeed | Skipped a step, invented a new approach |

Future kinds (not in MVP): `recovery_failure`, `premature_escalation`, `overhead_tax`.

## Auto-Detect Events

These events are generated from observable facts, not agent self-report:

1. **Out-of-vocab config value**: scaffold or plan produces a config value not in the defined vocabulary. Kind: `bad_default`, source: `grader`.
2. **Repeated failure signature**: same guard fails with same normalized error 2+ consecutive rounds. Kind: `repeat_loop`, source: `grader`.
3. **Boundary widen request**: agent requests `widen_boundary` doom_loop_action. Kind: `boundary_error`, source: `agent`, payload: `{direction: "underfit"}`.
4. **Human correction**: user overrides a protocol-suggested value during plan phase. Kind: `bad_default` or `protocol_gap`, source: `human`.

## Append Points

| When | Agent note path | Auto-detect |
|---|---|---|
| Scaffold complete | `.harness/tasks/<task_id>/feedback-note.json` | Check config values against vocabulary |
| Plan finalized | `.harness/tasks/<task_id>/feedback-note.json` | Check for human corrections vs draft |
| Each round end | `.harness/tasks/<task_id>/feedback-note.json` | Check repeated failure signatures |
| Session end | `.harness/tasks/<task_id>/feedback-note.json` | N/A |

## Source Weights (for future reducer)

When the same `subject_ref` accumulates events from multiple sources:
- `human`: weight 1.0 (ground truth)
- `grader`: weight 0.8 (deterministic or structured review)
- `agent`: weight 0.3 (self-report, known blind spots)

## Reducer (MVP)

The reducer groups events by `subject_ref + kind` and emits a candidate when:

- event count is at least 3
- distinct `run_id` count is at least 2

This is intentionally conservative. Single-run noise should not become protocol changes.

Current reducer entry point:

```bash
python ~/.asb/skills/harness/scripts/reduce_feedback.py \
  ~/.asb/state/harness-feedback/events.jsonl \
  --output ~/.asb/state/harness-feedback/candidates.json
```

Current candidate shape:

```json
{
  "candidate_count": 1,
  "candidates": [
    {
      "subject_ref": "harness.plan.boundary",
      "kind": "boundary_error",
      "event_count": 3,
      "distinct_runs": 3,
      "source_weight_total": 2.6,
      "sources": ["grader", "human"],
      "latest_summary": "boundary still too narrow",
      "sample_run_ids": ["run-a", "run-b", "run-c"]
    }
  ]
}
```

The reducer is a prioritization aid, not an automatic editor. Humans or later eval loops still decide whether a candidate becomes a real protocol change.

## MVP Workflow

1. Agent writes a structured note at the phase boundary
2. System appends events to `~/.asb/state/harness-feedback/events.jsonl`
3. Human reviews with `jq` when needed: `jq -s 'group_by(.kind) | map({kind: .[0].kind, count: length})' events.jsonl`
4. After 30-50 events, patterns emerge naturally. Then build a reducer.
