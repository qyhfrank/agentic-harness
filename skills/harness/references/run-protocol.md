# Run Protocol

The autonomous propose-verify-evaluate-record cycle, plus verification gates and evaluation rules.

## Prerequisites

- Preflight passed (see `setup-protocol.md` Preflight)
- Baseline recorded as round 0 in `state.jsonl`
- config.yaml is finalized (no `draft: true`)

## Single Round Lifecycle

```
1. PROPOSE  -->  2. COMMIT  -->  3. VERIFY  -->  4. EVALUATE  -->  5. RECORD  -->  6. ENTROPY CHECK
     ^                                                                                             |
     |_____________________________________________________________________________________________|
                                                   continue
```

### 1. Propose

- Read context.md Next Steps for direction.
- Check context.md Durable Notes: do not revisit known dead ends, violate known constraints, or ignore known tool quirks.
- If the change is non-trivial, invoke `brainstorming` to explore alternatives first.
- Respect `implementation.protocol` from config:
  - `tdd_required`: create failing test first, confirm it fails, then write minimal code to pass.
  - `tdd_preferred`: follow TDD when practical, record why if skipped.
  - `direct`: no red-first required, but behavior changes need verification coverage.
- Make code changes within `boundary.mutable`. Never touch `boundary.immutable`.
- Apply `atomicity_test`: if the change description needs "and", split into separate rounds.

### 2. Commit

- Stage only files within `boundary.mutable`.
- Commit with message: `harness(round-{N}): <description>`
- If pre-commit hook blocks: record `hook_blocked`, move to Record.

### 3. Verify

Filter gates by frequency:
- `every_round`: always run (Tier 0 and Tier 1 gates).
- `milestone`: run every N rounds (default 10), plus always on final round.
- `final`: only during completion verification pass.

Execute eligible gates in order:
1. Run all mandatory `command` gates.
2. On all pass, run `agent_review` gates (if configured). Use `/critique` as the review engine.
3. On pass or escalation, queue `human_review` if configured.
4. Short-circuit on first mandatory deterministic `fail`.

Capture full stdout/stderr to `artifacts/round-{N}/`.

### 4. Evaluate

| Verification + evaluation outcome | Result |
|---|---|
| Deterministic gate failed | `discard` |
| Review produced unresolved escalation | `needs_escalation` |
| Verification command crashed | `crash` |
| Verified, but improvement below `min_delta` | `no_op` |
| Verified and frontier improved or held | `keep` |
| Objective met and close authority satisfied | `complete` |

After revert (`discard` or `crash`): verify the revert itself does not break baseline.

### 5. Record

Append one event to `state.jsonl`. Required fields: `event`, `task_id`, `ts`, `round`, `commit`, `verification.status`, `evaluation.result`, `summary`.

Before writing context.md, route information correctly:
- Contract/threshold -> `config.yaml`
- Current status/next action -> `context.md` Current State / Working Memory / Next Steps
- Durable reusable fact -> `context.md` Durable Notes
- Raw output/proof -> `artifacts/`

Update context.md:
- Overwrite Current State
- Curate Working Memory
- Rewrite Next Steps
- Append to Decisions if a new decision was made
- Update Durable Notes if this round produced cross-round knowledge

### 6. Entropy Check

#### Stop Conditions (check in order)

1. **Objective met** (`satisfy` mode): all acceptance_criteria pass and close_authority satisfied.
2. **Budget exhausted**: current round >= `max_rounds`. (`-1` disables budget stop.)
3. **Stagnation**: last N rounds all `no_op` or `discard`.
4. **Escalation**: result is `needs_escalation`. Pause for human input.
5. **Complete**: latest round result is `complete`.

#### Doom Loop Detection

If the same guard fails with the same error signature N times (N = `doom_loop_threshold`):

First invoke `systematic-debugging` to diagnose root cause. Then:

| Action | Behavior |
|---|---|
| `reread_and_pivot` | Re-read mutable files, invoke `brainstorming` for alternatives, record failed approach in Durable Notes. |
| `codex_rescue` | Delegate diagnosis to Codex via `/codex-exec`. If viable fix found, continue. Otherwise fall back to `ask_human`. |
| `widen_boundary` | Suggest expanding mutable boundary. Requires user approval. |
| `ask_human` | Pause, present failure pattern, ask for guidance. |

#### Continue

If no stop condition fires, return to step 1 (Propose).

Do not stop on a successful round while non-blocked work remains.

---

## Verification Gates

### Gate Types

| Type | Executor | Deterministic | Trust |
|---|---|---|---|
| `command` | script / test / lint / build | Yes | Medium |
| `agent_review` | LLM reviewer via `/critique` | No | Low-Medium |
| `human_review` | Human | N/A | Highest |

### Verification Tiers

| Tier | Question | Frequency | Examples |
|---|---|---|---|
| 0 | Did I break anything? | `every_round` | typecheck, lint, unit tests |
| 1 | Did this specific change work? | `every_round` | isolated benchmark, profiling probe, smoke metric |
| 2 | Does the whole system work? | `milestone` or `final` | full e2e suite, integration tests |

**Tier 1 is the key insight.** Most workflows default to unit tests (Tier 0) or full e2e (Tier 2), skipping the middle. A Tier 1 gate is a task-specific isolated probe that answers "did this optimization actually help?" without running the full pipeline.

### Verdict Schema

Every gate produces: `verdict: { status: pass|fail|needs_escalation, evidence: "..." }`

### agent_review Rules

1. Single reviewer cannot close a task. Produces `needs_escalation` or a candidate opinion only.
2. Evidence must contain mechanically checkable anchors (file:line, test name). No anchors -> `needs_escalation`.
3. When the repo has runnable deterministic checks, `agent_review` cannot be the sole mandatory gate.
4. `/critique` is a good fit for structured review. It is not a completion oracle.

### Oracle Lifting

When `/critique` repeatedly confirms the same class of finding, invest in converting it to a `command` gate. Over time, the system should migrate from review-dependent to command-backed verification.

---

## Evaluation Rules

### Role Split

| Role | Question |
|---|---|
| Verification | Is this candidate safe and acceptable enough to consider? |
| Evaluation | Does this candidate improve the frontier, satisfy the objective, or require termination? |

### Evaluation Results

`baseline`, `keep`, `discard`, `crash`, `no_op`, `hook_blocked`, `needs_escalation`, `complete`.

### Decision Rules (apply in order)

1. Verification failed -> `discard` or `needs_escalation`
2. Verification crashed -> `crash`
3. Verified but improvement below `min_delta` -> `no_op`
4. Verified and frontier improved or held -> `keep`
5. Objective met and close authority satisfied -> `complete`

### Close Authority

| Type | Meaning |
|---|---|
| `command_backed` | Completion requires deterministic command evidence. |
| `human_review` | Human explicitly approves closure. |
| `confirmed_review` | Review-based completion only after confirmation rules succeed. |

`agent_review` alone never closes a task.

### Metric Interpretation

- `increase`: higher is better
- `decrease`: lower is better
- `none`: informational only; rely on acceptance criteria

### Volatile Metrics

When `volatile: true`: run measurement N times (default 3), take median. Improvement must exceed `min_delta` for `confirmation_runs` consecutive rounds to be trusted.

---

## Rollback Safety

- `revert_commit` (default): `git revert HEAD --no-edit`
- `reset_to_last_pass`: `git reset --hard <last_keep_commit>` (destructive)
- `preserve_failed_experiments: true`: save diff to `artifacts/round-{N}/attempted.patch` before reverting

## Session Boundary

When ending a session mid-loop:
1. Complete the current round (do not leave a half-committed state).
2. Update context.md with full current state.
3. If mid-verify, record as `crash` and revert.
4. Report: current round, best result, how to resume (`/harness run`).
