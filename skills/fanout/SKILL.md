---
name: fanout
description: Use when multiple perspectives on the same question improve quality, or when lightweight parallel dispatch to independent subtasks is needed. Triggers on /fanout, /orchestrate, "fan out", "parallel agents", "split across agents", "get multiple opinions", "sample N agents".
user_invocable: true
argument-hint: <task> [-m split|sample] [-a <type>[:<count>]]... [-b]
---

Arguments: $ARGUMENTS

# Fanout

Parallel agent dispatch. For review workflows, prefer `/critique`. For large-scale changes with worktree isolation and PRs, use `/batch`.

## Arguments

| Flag | Default | Description |
| ---- | ------- | ----------- |
| `-m` | auto-infer | `split` (data-parallel, no aggregation) / `sample` (multi-sample, synthesize) |
| `-a` | 5 x default model | Number = count of default type; `type` = that type x5; `type:N` = specific |
| `-b` | off | Background child context |

```
/fanout analyze the root cause of this bug
/fanout fix these 5 test files -m split
/fanout analyze this bug -a codex:3 -a opus:2
/fanout analyze this bug -b
```

## Modes

| Mode | Each agent receives | Output | Infer when |
|------|-------------------|--------|------------|
| `split` | Different subtask | Results returned separately, no aggregation | Task decomposes into independent subtasks |
| `sample` | Same task | Synthesized answer from N perspectives | Not decomposable |

## Guards

- Applicable: 2+ independent subtasks, no shared state, agents don't compete for same files
- Not applicable: correlated failures, or full system state required to judge
- Broad research -> breadth (split); narrow hard problems -> focused depth (`-a codex` for Thinker)
- Minimum effective fan-out, but no artificial low caps on independent tasks

## Prompt Crafting

Each child prompt must be fully self-contained (Rule 8). Common mistakes:

| Mistake | Fix |
| --- | --- |
| "Fix all tests" (scope too wide) | "Fix agent-tool-abort.test.ts" (focused) |
| "Fix this race" (no context) | Include error message and test name |
| No constraints (agent refactors broadly) | "Don't touch production code" / "Only modify tests" |
| "Fix it" (vague output) | "Return root cause and change summary" |

Agent Context Card:

> Agent: <role> | depth <N>/<max> | budget <B> | parent: <parent_role>

Fill-in: role = `leaf` (default); depth = parent + 1; max = inherited (may tighten, never expand); budget = 0 for leaf.

## Execution

1. **Parse arguments + infer mode**

2. **Launch agents**: issue all parallel calls in a single response
   - Default foreground (wait for all results); `-b` enables background (only when parent has independent work)
   - `-a opus` (default): platform default child worker
   - `-a codex`: load `/codex-exec`, invoke `codex exec` via Bash
   - Mixed (`-a codex:3 -a opus:2`): route by type, unify results

3. **Collect + failure handling**: partial failure -> retry (max 2) -> 2+ successes continue, 1 = degrade to single result, all fail = report error

4. **Post-processing** (mode-dependent)
   - split: return each agent's output separately. No aggregation.
   - sample: inject N outputs into prompt, synthesize final answer
     > "Identify common points and differences, compare whether evidence points to the same source, synthesize evidence-backed strengths, discard unsupported conclusions"

5. **Verify (mandatory for sample mode)**: go back to source and verify synthesized conclusions against actual code, docs, or command output. Thinking models (Codex/Thinker) are strong at finding problems but prone to overthinking -- they may invent issues or propose fixes that add unnecessary complexity. Always verify findings against source before surfacing to the user. Skip verification only for pure creative tasks with no codebase or executable environment.
