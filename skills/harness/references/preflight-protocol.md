# Preflight Protocol

Run all checks below before entering the harness loop. On any mandatory failure, do NOT enter the loop.

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
3. Append a `baseline_recorded` event to `<harness_root>/.harness/tasks/<task_id>/state.jsonl`.
4. If baseline verification fails, the task config is broken. Return to plan mode immediately.

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
