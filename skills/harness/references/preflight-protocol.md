# Preflight Protocol

Run all checks below before entering the harness loop. On any mandatory failure, do NOT enter the loop.

## Session Initialization

Before running checks, establish session and controller identity, then emit a `session_started` event to `state.jsonl`:

- Generate a `session_id` in format `harness-run-YYYYMMDD-XXXX` (4-char random hex).
- Generate an `agent_id` in format `harness-controller-XXXX` (4-char random hex). This identifies the controller for provenance. If resuming and the previous session's `agent_id` is known, reuse it for continuity; otherwise generate a fresh one.
- If this is a fresh run, set `reason: initial`.
- If resuming (previous events exist in `state.jsonl`), check whether the last session ended cleanly (has a `session_ended` event). Set `reason` to `resume_after_pause` or `resume_after_recovery` accordingly. Include `prev_session_id` and `resume_round`.
- Record `ts` as the current UTC time.
- Include `agent_id` in the event.

Store the `session_id`, `agent_id`, and `session_started_at` in memory for use in all subsequent events this session.

Embedded `/planning` implementer agents do not replace the parent controller's ledger identity. The parent controller remains the sole `agent_id` writing to `state.jsonl`.

## Mandatory Checks

| # | Check | Rule |
| --- | --- | --- |
| 1 | Repo integrity | Git repo exists, not in detached HEAD, no stale `index.lock`. |
| 2 | Clean working tree | No uncommitted changes (`git status --porcelain` is empty). |
| 3 | Baseline verification | All mandatory verification gates pass on current HEAD. |
| 4 | Scope resolution | Every path in `boundary.mutable` and `boundary.immutable` exists on disk. |

## Optional Checks

- Pre-commit / pre-push hooks existence.
- CI config consistency with local verification commands.

Optional failures produce `[SKIP]` with reason, not a block.

## On Failure

Report which check failed and suggest a concrete fix. Example:

```
[FAIL] Clean tree: 2 uncommitted files (src/foo.ts, src/bar.ts)
  Fix: commit or stash changes before running harness.
```

Do not attempt auto-remediation. Return control to the caller.

## Baseline Recording

After all mandatory checks pass:

1. Run all mandatory verification gates against current HEAD (in the code worktree CWD).
2. Evaluate the baseline objective state and metric, if any.
3. Append a `baseline_recorded` event to `<harness_root>/.harness/tasks/<task_id>/state.jsonl`. Include `ts`, `session_id`, `agent_id`, and all v2 common envelope fields.
4. If baseline verification fails, the task config is broken. Return to plan mode immediately.
5. Update context.md Current State with timing anchors (`session_id`, `session_started`, `task_started`).

## Preflight Output Format

```
Preflight Check Results:
[PASS] Repo integrity: clean git repo on branch <task_id>
[PASS] Clean tree: no uncommitted changes
[PASS] Baseline verify: make test passed (14/14 tests, 72.3% coverage)
[PASS] Scope resolve: all mutable/immutable paths exist
[SKIP] Hook check: no pre-commit hooks configured

Baseline recorded: round 0, commit abc1234, metric 72.3%
Ready to enter loop.
```
