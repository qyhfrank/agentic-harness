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
2. Current git branch name `harness/<task_id>`
3. `.harness/current-task`
4. Sole task under `.harness/tasks/`

If no task can be resolved, scaffold derives a task ID from the user's goal (see Task ID Generation) and sets `.harness/current-task` to that value.

## Task ID Generation

Generate a meaningful kebab-case slug from the user's goal. The slug must be deterministic enough that a human can recognize the task from the ID alone.

Rules:

1. Extract the primary verb and object from the goal, drop filler words (the, a, an, this, that, for, with, and, or, to, in, on, of). Keep domain-specific nouns and verbs intact. Do not rephrase, summarize, or use synonyms.
2. Constraints: lowercase `[a-z0-9-]` only, no leading/trailing/consecutive hyphens. Max 40 characters for the final task ID including any collision suffix.
3. For non-ASCII goals: translate the verb and object literally to English (not paraphrase). Preserve ASCII proper nouns, API names, and technical terms as-is. If a stable translation is not possible, ask the user for a short English label.
4. If the derived slug is empty after sanitization (e.g., goal was all emoji or punctuation), ask the user for a task name instead of proceeding.
5. Collision handling: check `.harness/tasks/` for existing directories. If the slug collides, append `-2`, `-3`, etc. (smallest unused). Reserve suffix space from the 40-char budget: truncate the base slug to `40 - len(suffix)` before appending.
6. There is no `default` fallback. Missing Goal Gate guarantees a goal exists before scaffold runs. If somehow no slug can be derived, ask the user — never silently create a task.

Examples:

| Goal | Slug |
|---|---|
| "fix the auth timeout bug" | `fix-auth-timeout-bug` |
| "add dark mode support to the settings page" | `add-dark-mode-settings-page` |
| "raise test coverage above 80%" | `raise-test-coverage-above-80` |
| "修复登录超时问题" | `fix-login-timeout` |
| "优化首页加载速度" | `optimize-homepage-load-speed` |

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

If auto-inferred mode is `run` but the user's message implies planning intent (asking questions about config, boundaries, verification, evaluation, or strategy), override to `plan`.

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

### Fresh Start

Run preflight checks first. On pass, enter the loop defined in `references/loop-protocol.md`.

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
