---
name: harness
description: 'Agent autonomously iterates on a codebase with verified feedback loops.'
argument-hint: "[setup|run] [goal description]"
---

# Harness

## Task Surface

### Task-state carriers

```text
.harness/
  shared/                    # optional deduped immutable setup snapshots by digest
  tasks/
    <task_id>/               # NNN-<task_slug>; NNN = smallest unused unique 3-digit prefix
                             # task_slug: kebab-case from goal ([a-z0-9-], <= 36 chars)
      config.yaml            # harness contract (static)
      plan.yaml              # control state: intent anchors, source refs, strategy/tactics (mutable)
      context.md             # live working surface and next-step handoff (display)
      state.jsonl            # append-only event ledger (audit truth)
      artifacts/             # evidence written by run
        setup/               # setup source snapshots or pointers
        round-{N}/           # one directory per round, N starts at 1
```

## Route

Before routing: parse ledger, classify legacy aliases read-only, validate terminal invariants. Active non-canonical ledgers route to repair/recovery and may resume only after canonical repair. Archived legacy carriers may only report a warning.

First match wins:

1. Canonical `task_disposed` ledger tail -> report final summary, do not resume loop
2. `harness_stopped` in `state.jsonl` without `task_disposed` -> report stop reason and last round summary, present disposition via AskUserQuestion, do not enter round lifecycle
3. No resolved task -> **setup:new** -> Load `references/setup.md`
4. Task files incomplete, contract not fully specified, `plan.yaml` missing, `plan.yaml` has empty `milestones[]` without a recorded bootstrap, or an unstarted task lacks plan provenance -> **setup:repair** -> Load `references/setup.md`
5. `state.jsonl` empty but working directory shows progress beyond setup defaults -> **run:recovery** -> Load `references/run.md`
6. `state.jsonl` has no `baseline_recorded` -> **run:fresh** -> Load `references/run.md`
7. Has recorded events -> **run:resume** -> Load `references/run.md`

- `/harness setup`: route to setup, stop after writing task state.
- Bare `/harness`: continue into run after successful setup.
- `context.md` may mirror stopped/disposed state for humans, but route from ledger events, not display text.

## Global Invariants

1. **Config is the contract.** Repo discovery may suggest values; written config wins. `config.yaml` is static contract only; strategy, milestones, plan sources, and brainstorming summaries live in `plan.yaml` and `artifacts/setup/`.
2. **Boundary is law.** `boundary.mutable` governs implementation files, not harness bookkeeping. Current task carrier is implicitly mutable for `context.md`, `state.jsonl`, `plan.yaml`, and new round artifacts; sealed prior round artifacts are append-only evidence and corrections go in later errata/manifest. Sibling carriers are immutable unless being repaired. Never modify outside `boundary.mutable`; never touch `boundary.immutable`. Boundary expansion after run starts requires user confirmation plus config update.
3. **Worktree is mandatory.** Never modify repo files on the working branch. Create `.worktree/<task_slug>/` (branch `<task_slug>`) at Preflight; all code changes go there. `.harness/` stays outside the worktree.
4. **Checks and evaluation stay distinct.** Checks produce safety and quality signals; evaluation decides `kept`, `reverted`, `escalated`, or `done`.
5. **State is append-only.** Do not edit/delete lines with later events; tail may be amended before next append. Milestone advance uses `round_completed.controller.next_milestone_id`; `strategy_updated` is for bootstrap/replan only. Legacy `reason: reopen` is read-only recovery evidence. Post-stop scope expansion starts a new task unless user reopens a stopped-not-disposed task; disposed tasks never reopen.
6. **Truth hierarchy.** `state.jsonl` = audit history. `config.yaml` = contract. `plan.yaml` = current intent/sources/strategy/tactics. `context.md` = display. `artifacts/setup/` snapshots never control routing/recovery. On conflict: restore active pointers from ledger, contract fields from config; unrebuildable structural corruption -> escalate. Rewrite `context.md` unconditionally.

## Integration Overlays

### Native progress mirror

If native user-visible task or todo progress exists, mirror the current harness status into it. Mirror only progress display. Do not use native tasks for routing, recovery, verification, or control; `state.jsonl` and `plan.yaml` stay authoritative.

### Unattended = silent

When `unattended` wraps harness, intermediate rounds produce no chat output. Notify only on `done`, budget exhausted, stagnation, or hard blocker.
