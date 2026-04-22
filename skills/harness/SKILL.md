---
name: harness
description: 'Use when the user wants an agent to autonomously iterate on a codebase with verified feedback loops.'
argument-hint: "[setup|run] [goal description]"
---

# Harness

## Task Surface

### Task-state carriers

```text
.harness/
  tasks/
    <task_id>/               # NNN-<task_slug>. NNN = smallest unused 3-digit number in .harness/tasks/
                             # task_slug: kebab-case short name derived from goal ([a-z0-9-], <= 36 chars)
      config.yaml            # harness contract (static)
      plan.yaml              # strategy + tactics controller state (mutable)
      context.md             # live working surface and next-step handoff (display)
      state.jsonl            # append-only event ledger (audit truth)
      artifacts/             # evidence written by run
        round-{N}/           # one directory per round, N starts at 1
```

Never write `.harness/` or task state inside a worktree.

## Route

First match wins:

1. `task_disposed` or `round: N / stopped` -> report final summary, do not resume loop
2. `harness_stopped` in `state.jsonl` without `task_disposed` -> report stop reason and last round summary, present disposition options, do not enter round lifecycle
3. No resolved task -> **setup:new** -> Load `references/setup.md`
4. Task files incomplete, contract not fully specified, or `plan.yaml` missing -> **setup:repair** -> Load `references/setup.md` (missing plan.yaml with recorded events: repair builds active plan from goal + ledger pointers)
5. `state.jsonl` empty but working directory shows progress beyond setup defaults -> **run:recovery** -> Load `references/run.md`
6. `state.jsonl` has no `baseline_recorded` -> **run:fresh** -> Load `references/run.md`
7. Has recorded events -> **run:resume** -> Load `references/run.md`

- `/harness setup`: route to setup, stop after writing task state.
- Bare `/harness`: continue into run after successful setup.

## Global Invariants

1. **Config is the contract.** Repo discovery during setup may suggest values; the written config wins.
2. **Boundary is law.** Never modify files outside `boundary.mutable`; never touch `boundary.immutable`. If boundary needs expanding after run starts, confirm with user first, then update config.
3. **Checks and evaluation stay distinct.** Checks produce safety and quality signals; evaluation decides `kept`, `reverted`, `escalated`, or `done`.
4. **State is append-only.** `state.jsonl` is append-only. Lines with subsequent events must not be edited or deleted; the tail (last entry) may be amended in-place before the next append.
5. **Truth hierarchy.** `state.jsonl` = audit truth (history). `plan.yaml` = control truth (current decisions). `context.md` = display layer (human-readable mirror, never participates in routing or recovery). On conflict: restore active pointers from ledger events; structural corruption (missing milestones/approaches/steps) that cannot be rebuilt from pointers → escalate. Rewrite `context.md` unconditionally.

## Integration Overlays

### Caffeine = silent

When `caffeine` wraps harness, intermediate rounds produce no chat output. Notify only on `done`, budget exhausted, stagnation, or hard blocker.
