# Plan

## `plan.yaml` Schema

```yaml
version: 1

plan_sources:
  - id: S1
    kind: setup_synthesis        # brainstorming_embedded|external_file|setup_synthesis|legacy_repair|replan
    original_path: null          # user-provided source path, if any
    artifact_path: null          # artifacts/setup/<source snapshot>, if any
    sha256: null
    imported_at: null
    status: accepted             # accepted|superseded|rejected
    summary: "..."

planning_context:
  done_when_hint: null
  non_goals: []
  constraints: []
  assumptions: []
  decisions: []
  risks: []
  open_questions: []

strategy:
  version: 1
  status: pending              # pending|active|done|blocked
  active_milestone_id: null
  last_replan_round: 0
  source_refs: []

milestones:
  - id: M1
    title: "..."
    objective: "..."
    exit_criteria: "verifiable condition"
    source_refs: []
    status: pending            # pending|active|done|blocked|dropped
    approaches:
      - id: A1
        hypothesis: "..."
        rationale: "..."
        risk_notes: []
        score: 70              # 0-100; initial spread >= 15 between ranks
        source_refs: []
        status: candidate      # candidate|active|failed|done|blocked
        steps: []              # ordered sub-steps; optional, populated when approach has multi-step structure
        current_step: 0        # index into steps[]; 0 when steps is empty or first step
        attempts: 0
        evidence_for: []
        evidence_against: []
        # revert_streak, last_failure_family: omitted at bootstrap; created on first revert
```

Field notes: `version` bumps only on structural replan. `plan_sources[]` records seed origin; `artifact_path` points to the full external/embedded plan snapshot. Snapshots are evidence, not control truth. `planning_context` holds compact intent anchors from the plan; contract fields stay in `config.yaml`; contract-blocking `open_questions` keep setup open. `source_refs[]` cite source ids. `exit_criteria` must reference a check, artifact, or observable code-state claim. `score` ranks approaches; it is not probability. `evidence_for/against` are one-line summaries; full evidence stays in `artifacts/`.

Setup writes populated pending `milestones[]`; run Bootstrap activates/enriches after baseline. Empty `milestones[]` is legacy/repair fallback only.

### Approach steps

`steps[]` is an optional ordered list of sub-objectives within an approach. Each entry is a short string describing one verifiable sub-goal. When populated, Propose consumes `steps[current_step]` instead of reasoning from the hypothesis alone. On `kept`, advance `current_step` if the sub-goal is met. On `reverted`, retry the same step (not the same patch). When all steps are consumed, evaluate the milestone `exit_criteria`.

Steps are generated alongside the approach during bootstrap or replan. They may be empty for simple approaches where the hypothesis is self-explanatory. Steps are advisory, not binding: Propose may skip ahead or merge steps if evidence warrants, but must note the deviation in `Decisions`.

When `current_step >= len(steps)` but milestone `exit_criteria` is not met: fall back to hypothesis-level Propose (ignore steps, scope round to the approach hypothesis directly). If the next round also fails to advance toward exit_criteria, treat as normal adapt logic (demote/failed based on failure_scope).

### Step content standards

Not every approach needs high-precision steps. Generate tactical steps when:

- The approach creates or modifies 2+ files
- Changes require a specific sequence to stay green
- `task.protocol` is `tdd_required` or `tdd_preferred`

For single-file changes or exploratory spikes, short descriptive steps suffice.

**Per-step requirements** (when tactical):

1. **Files touched** — exact repo-relative paths; distinguish create vs modify
2. **Content** — precise intended change; include exact code/text only when required to keep the round executable
3. **Verification** — command + expected outcome (pass/fail pattern)

Granularity: one action per step. If a description needs "and", split it. Target: each step = one harness round.

**File mapping:** Before generating steps, list all files the approach will touch. This locks scope; no file appears in a step that wasn't listed.

**TDD rhythm** (when `tdd_required` or `tdd_preferred`): capture failing reproduction → verify failure → write minimal implementation → verify pass. See `references/tdd-discipline.md` for fail-first rules and test quality anti-patterns.

- `tdd_required`: if the step changes production behavior, the step expansion must name the failing reproduction, the proof command, and where the RED evidence will be recorded before implementation.
- `tdd_preferred`: use the same rhythm by default; if a step is pure docs, config, or other non-behavior work, note why the TDD overlay does not apply in `Decisions`.

**Propose expansion:** Steps stored in plan.yaml stay as short strings. During Propose, expand the current step into one executable round with file paths, intended change, and verification command.

**No-placeholder rules** — step expansions fail on TBD/TODO/later, vague edge-case wording, tests without assertion intent, "similar to step N", undefined types/functions, or prose-only implementation where exact content is required.

## Approach Lifecycle

Status transitions:

```
candidate -> active -> done
                    -> failed
                    -> blocked
```

Only one approach per milestone has `status: active`. On failure, promote highest-score `candidate`. No approach is ever resurrected after `failed` unless new evidence explicitly removes the original blocker.

## Score Mechanics

| Round outcome | Condition | Score delta | Approach decision |
|---|---|---|---|
| `kept` | milestone not done | 0 | `continue` |
| `kept` | exit_criteria met | 0 | `complete` -> advance milestone |
| `done` | final task objective met | 0 | `task_done` -> strategy.status=done, all active milestones/approaches → done |
| `escalated` | any | 0 | `blocked` -> stop |

**On revert (conditional):**

| Round outcome | Condition | Score delta | Approach decision |
|---|---|---|---|
| `reverted` | first occurrence of this failure family | -10 | `demote` (retry same approach) |
| `reverted` | same failure family repeated OR `[dead-end]` evidence | -30 | `failed` -> switch approach |
| `reverted` | `[constraint]` invalidates hypothesis | -30 | `failed` -> switch approach |

Hysteresis: active approach keeps priority unless a candidate exceeds it by >= 15 points. Since kept rounds do not increase score, the active approach's lead is set at bootstrap; only reverts can erode it.

## Failure Scope

`last_failure_family` classifies revert cause:

- `execution`: implementation bug, typo, wrong file. Retry same approach.
- `hypothesis`: evidence shows this approach direction is unviable. Switch approach.
- `environment`: tool quirk, flaky infra. Retry; repeated → escalate.

Default to `execution` when ambiguous. Only upgrade to `hypothesis` on repeated same-family failure or explicit `[constraint]`/`[dead-end]` evidence.

## Adapt Decision Table

The `adapt` step runs after Evaluate, before Record. It translates round verdicts into plan actions:

**Always:**
1. Read `evaluation.result` and current `plan.yaml` state
2. Update approach counters (`attempts`)

**If kept:**
3. If approach has `steps[]`: advance `current_step` when sub-goal is met
4. Determine `approach_decision`: `continue|complete|task_done`
5. If `complete`: mark milestone done, activate next pending milestone, set `controller.next_milestone_id`

**If done:**
3. Set `strategy.status = done`, mark current milestone and approach as `done`, set `approach_decision = task_done`

**If reverted:**
3. Classify `failure_scope` (`execution|hypothesis|environment`)
4. Update `revert_streak` and `score` per delta table
5. Determine `approach_decision`: `demote|failed|blocked`
6. If `failed`: mark approach, activate next candidate by score
7. If no candidates remain: set `strategy_signal = all_approaches_exhausted`

**If escalated:**
3. Set `approach_decision = blocked`, `approach.status = blocked`

**Finally (all outcomes):**
Write updated `plan.yaml`; emit `strategy_updated` only for non-linear transitions (replan, reopen). Milestone advance uses `controller.next_milestone_id`, not `strategy_updated`.
