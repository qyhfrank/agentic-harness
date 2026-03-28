---
name: harness
description: >
  Use when the user wants an agent to autonomously iterate on a codebase with
  verified feedback loops. Triggers: /harness, "set up a harness", "run harness",
  "scaffold harness", "autonomous iteration", "verified code loop", "harness plan",
  "harness run", or when the user describes a task that needs repeated
  propose-verify-keep/discard cycles against a test suite or metric.
user_invocable: true
argument-hint: "[scaffold|plan|run] <goal description>"
---

# Harness

Autonomous iteration engine: propose code changes, verify against gates, evaluate progress, keep or discard, repeat.

Three phases: **scaffold** (diagnose repo, create task-scoped `.harness/` state), **plan** (interactive config finalization), **run** (autonomous loop).

## Canonical Task Layout

Harness always uses task-scoped state:

```
.harness/
  current-task
  tasks/
    <task_id>/
      config.yaml
      context.md
      state.jsonl
      artifacts/
```

Single-task usage is just the case where only one task exists. It is not a second storage layout.

## Task Resolution

Resolve the current task in this order:

1. Explicit task identifier supplied by the user or command wrapper
2. Current git branch name matching a unique task slug from an existing `<task_id>`
3. `.harness/current-task`
4. Sole task under `.harness/tasks/`

If no task can be resolved, scaffold derives a task slug and task ID from the user's goal (see Task ID Generation) and sets `.harness/current-task` to the task ID.

## Task ID Generation

Generate two related names from the user's goal:

- `task_slug`: a meaningful kebab-case slug derived from the goal. Use this for branch names, worktree names, and other human-facing labels.
- `task_id`: the durable harness storage ID of the form `NNN-<task_slug>`. Use this only for `.harness/tasks/<task_id>/` and `.harness/current-task`.

The slug must be deterministic enough that a human can recognize the task from the ID alone.

Rules:

1. Extract the primary verb and object from the goal, drop filler words (the, a, an, this, that, for, with, and, or, to, in, on, of). Keep domain-specific nouns and verbs intact. Do not rephrase, summarize, or use synonyms.
2. Build `task_slug` with lowercase `[a-z0-9-]` only, no leading, trailing, or consecutive hyphens. Keep the human-readable slug within 36 characters so the full `task_id` fits the existing 40-character budget once `NNN-` is prepended.
3. For non-ASCII goals: translate the verb and object literally to English (not paraphrase). Preserve ASCII proper nouns, API names, and technical terms as-is. If a stable translation is not possible, ask the user for a short English label.
4. If the derived slug is empty after sanitization (e.g., goal was all emoji or punctuation), ask the user for a task name instead of proceeding.
5. Allocate `task_id` by scanning `.harness/tasks/` for numeric prefixes and picking the smallest unused zero-padded three-digit prefix, then append `-<task_slug>`. Example: `000-fix-auth-timeout-bug`.
6. The numeric prefix belongs only to harness storage IDs. Do not prepend it to branch names, worktree names, or other human-facing labels unless the user explicitly asks.
7. If branch or worktree naming needs collision handling, keep that logic on `task_slug` only (for example, `fix-auth-timeout-bug-2`). Do not rewrite the already allocated `task_id`.
8. There is no `default` fallback. Missing Goal Gate guarantees a goal exists before scaffold runs. If somehow no slug can be derived, ask the user — never silently create a task.

Examples:

| Goal | Task Slug | Example Task ID |
|---|---|---|
| "fix the auth timeout bug" | `fix-auth-timeout-bug` | `000-fix-auth-timeout-bug` |
| "add dark mode support to the settings page" | `add-dark-mode-settings-page` | `001-add-dark-mode-settings-page` |
| "raise test coverage above 80%" | `raise-test-coverage-above-80` | `002-raise-test-coverage-above-80` |
| "修复登录超时问题" | `fix-login-timeout` | `003-fix-login-timeout` |
| "优化首页加载速度" | `optimize-homepage-load-speed` | `004-optimize-homepage-load-speed` |

## Mode Detection

Detect mode from explicit argument first, then auto-infer from repo state and the resolved task:

```
Explicit `/harness scaffold|plan|run` --> use that mode

Otherwise auto-infer:
  .harness/ does not exist                          --> scaffold
  current task missing                              --> scaffold
  current task config has `draft: true`            --> plan
  current task config finalized, state.jsonl empty --> run (fresh)
  current task config finalized, state.jsonl has events --> run (resume)
```

If `state.jsonl` is empty but `context.md` or `artifacts/` clearly show the
task predates ledger discipline, treat run entry as legacy archive recovery
instead of a blank-slate fresh start. Follow
`references/context-protocol.md` before preflight, and do not backfill
historical events into `state.jsonl`.

If auto-inferred mode is `run` but the user's message implies planning intent (asking questions about config, boundaries, verification, evaluation, or strategy), override to `plan`.

## Worktree Isolation

Harness defaults to running inside a git worktree. This keeps iteration work isolated from the main branch until the user explicitly merges.

### Enforcement

After mode detection, before executing any mode:

1. Check if the session is already inside a worktree (e.g., CWD is under `.claude/worktrees/`).
2. If already in a worktree, proceed normally.
3. If not in a worktree, create one before writing any files or making changes.

### Worktree Creation

Call `EnterWorktree` with name `<task_slug>`. The worktree branch is `<task_slug>`.

- **Task ID known** (plan, run, resume, or scaffold with explicit goal): derive or resolve the task ID first, then use its slug portion for worktree and branch naming.
- **Task ID unknown** (scaffold without goal): trigger the Missing Goal Gate to obtain the goal, derive the task slug, allocate the task ID, then create the worktree from the slug. Do not create a worktree before the goal is known.

After entering the worktree, proceed with the detected mode.

### Opting Out

If the user explicitly requests in-place execution (e.g., "no worktree", "in-place", "on this branch"), skip worktree creation and proceed directly. Do not prompt a second time.

## Missing Goal Gate

Scaffold requires a concrete task goal, not just a request to initialize harness.

- Treat `/harness`, `run harness`, `set up a harness`, and similar phrases as routing signals, not as the task objective.
- If the repo needs scaffold but the user has not said what the harness should iterate on, ask exactly one targeted question for the goal before creating task files.
- If the user gives only a broad intent such as `optimize this repo` or `make it better`, ask exactly one focused narrowing question before creating task files.
- Recommended form: `What task should this harness pursue in this repo? For example: fix a bug, add a feature, raise test coverage, or optimize a metric.`
- Recommended narrowing form: `What specific outcome should this harness optimize here? For example: reduce test failures, add smoke coverage for one area, or improve a named metric.`
- Do not write `task.description` or `current_objective` from generic setup language.

## Mode: scaffold

Read `references/scaffold-protocol.md` and follow it.

Also read `references/feedback-protocol.md` for the feedback note step at scaffold completion.

Produces: `.harness/current-task`, `.harness/tasks/<task_id>/config.yaml`, `.harness/tasks/<task_id>/context.md`, `.harness/tasks/<task_id>/state.jsonl`, `.harness/tasks/<task_id>/artifacts/`, and an `AGENTS.md` update.

Idempotent: re-running scaffold only fills gaps, never overwrites existing files.

Before loading `references/scaffold-protocol.md`, ensure the goal is known. If the goal is missing, ask for it first and stop there.

## Mode: plan

Read these references before proceeding:
- `references/plan-protocol.md` -- interactive config finalization flow
- `references/state-ledger.md` -- `state.jsonl` schema and reading conventions
- `references/verification-gate.md` -- gate types and verifier semantics
- `references/evaluation-protocol.md` -- evaluation roles and completion rules
- `references/feedback-protocol.md` -- feedback note at plan finalization

Produces: finalized config.yaml (removes `draft: true`), updated context.md.

Exit criteria: mutable surface, verification commands, evaluation policy, and stop guards are all explicitly defined. Mandatory gate dry-run passes.

## Mode: run

Read these references before proceeding:
- `references/loop-protocol.md` -- core iteration lifecycle
- `references/preflight-protocol.md` -- pre-loop checks
- `references/verification-gate.md` -- gate execution and verifier verdict schema
- `references/evaluation-protocol.md` -- evaluation decisions and close authority
- `references/state-ledger.md` -- state recording
- `references/context-protocol.md` -- context.md update and session recovery
- `references/feedback-protocol.md` -- feedback notes at round/session boundaries

### Recovery (resume)

When `state.jsonl` has existing events, this is a resumed session. Follow the recovery protocol in `references/context-protocol.md` before entering the loop.

### Recovery (legacy archive)

When `state.jsonl` is empty but the task directory clearly preserves earlier
work from before ledger discipline, do not treat the task as a blank fresh run.
Follow the legacy archive recovery path in `references/context-protocol.md`:

- recover the current state from `context.md` plus the smallest archived
  artifact set that answers the recovery questions
- rewrite `context.md` so recovered facts are clearly labeled as recovered from
  artifacts
- keep `state.jsonl` append-only and empty until the next real preflight records
  a fresh baseline
- do not synthesize or backfill historical `baseline_recorded` or
  `round_completed` events

### Fresh Start

Run preflight checks first. On pass, enter the loop defined in `references/loop-protocol.md`.

## Completion

When the harness loop terminates (evaluation result `complete`, budget exhausted, stagnation, or user-initiated stop), and the session is running inside a worktree created by Worktree Isolation:

1. Report the final result summary (rounds completed, best frontier, acceptance criteria status).
2. Proactively ask the user how to handle the worktree changes using `AskUserQuestion`:
   - **Merge to main** -- squash-merge the worktree branch into main and remove the worktree.
   - **Keep worktree** -- leave the worktree and branch intact for manual review or continued work.
   - **Discard** -- remove the worktree and discard all changes.

### Merge to Main

If the user chooses to merge:

1. Ensure all harness rounds are committed (no uncommitted state).
2. Switch back to the original directory with `ExitWorktree` action `keep`.
3. Merge the worktree branch into the original branch (the one active before `EnterWorktree`, typically `main`): `git merge --squash <worktree-branch>`, then commit with message `harness(<task_slug>): <one-line summary of what the harness achieved>`.
4. Clean up: `git branch -D <worktree-branch>` and remove the worktree directory.

### Keep or Discard

- **Keep**: call `ExitWorktree` with action `keep`. Report the worktree path and branch name so the user can return later.
- **Discard**: ensure feedback notes have been expanded to durable events in `~/.asb/state/harness-feedback/events.jsonl` before removal. Then call `ExitWorktree` with action `remove` (set `discard_changes: true` if needed). Confirm with the user before discarding uncommitted work.

If the session is not in a harness-created worktree (user opted out or was already in-place), skip this section entirely.

## Skill Integration

Harness operates within the ASB ecosystem. At specific points, prefer invoking a specialized skill over ad-hoc reasoning:

| Trigger | Skill | When |
|---|---|---|
| Plan phase: exploring task scope, boundary, verification, or evaluation strategy | `brainstorming` | Before filling config fields that require creative or architectural judgment |
| Doom loop: same error repeats N times | `systematic-debugging` | Before attempting pivot -- diagnose root cause first |
| Doom loop: pivot needed after diagnosis | `brainstorming` | Generate alternative strategies instead of guessing |
| `agent_review` or discovery review verification gate | `critique` | Use as the structured review engine for agent-review gates |
| Run phase: complex multi-file change in Propose step | `brainstorming` | When the change is non-trivial and benefits from exploring alternatives |

These are not mandatory for every round. Invoke when the trigger condition matches. If the change is simple and the path is clear, skip straight to implementation.

## Cross-Cutting Rules

1. **Boundary is law.** Never modify files outside `boundary.mutable`. Never touch `boundary.immutable`.
2. **Verification is mandatory.** Every proposed change must pass through the verification gate before being kept.
3. **Evaluation is distinct.** Verifiers produce safety and quality signals; evaluators decide frontier updates, completion, and termination.
4. **State is append-only.** Never edit or delete existing events in `state.jsonl`. Only append.
5. **Context survives sessions.** Update context.md every round so a fresh agent can resume.
6. **Config is the contract.** All runtime behavior derives from config.yaml. No implicit defaults.
7. **Reviewer context is bounded.** Reviewers get the diff, relevant files, AGENTS.md, and config excerpts they need. They do not get proposer reasoning as authority.

## Reference Index

| File | Purpose | Loaded by |
|---|---|---|
| `scaffold-protocol.md` | Phase 0: repo diagnosis, task directory creation | scaffold |
| `plan-protocol.md` | Phase A: interactive config finalization | plan |
| `loop-protocol.md` | Phase B: propose-verify-evaluate-record cycle | run |
| `preflight-protocol.md` | Pre-loop validation checks | run |
| `verification-gate.md` | Gate types, verifier verdict schema, volatile metrics | plan, run |
| `evaluation-protocol.md` | Evaluator role, metric interpretation, close authority | plan, run |
| `state-ledger.md` | JSONL event schema and reading conventions | plan, run |
| `context-protocol.md` | context.md updates, session recovery | run |
| `feedback-protocol.md` | Skill-level feedback notes and event schema | scaffold, plan, run |
