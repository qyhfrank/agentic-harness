---
name: fanout
description: Use when multiple perspectives on the same question improve quality, or when lightweight parallel dispatch to independent subtasks is needed. Triggers on /fanout, "fan out", "parallel agents", "split across agents", "get multiple opinions", "sample N agents".
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
| `-a` | 5 x `auto` | Number = count of inferred worker type; `type` = that type x5; `type:N` = specific. Supported types: `auto`, `thinker`, `doer`, `codex`, `opus` |
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
- Broad research -> breadth (split); narrow hard problems -> focused depth (`-a thinker` or explicit `-a codex`)
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

2. **Run the pre-flight gate**
   - If the task fits locally, keep it local.
   - If the task is a deterministic wait or scriptable monitor, use tool execution instead of child contexts.
   - Only continue when multiple child contexts materially improve the result.

3. **Choose worker type**
   - No `-a`: infer `thinker` for planning, review, root-cause analysis, and architecture decisions; infer `doer` for implementation, testing, and mechanical refactoring.
   - `-a thinker` / `-a doer`: use the current platform's Thinker / Doer mapping.
   - `-a codex` / `-a opus`: explicit runner override.
   - If a requested runner is unavailable on this platform, fail fast and say so instead of silently substituting.

4. **Launch agents**: issue all parallel calls in a single response
   - Default foreground (wait for all results); `-b` enables background (only when parent has independent work)
   - `auto`, `thinker`, `doer`: route through the current platform's archetype mapping
   - `-a codex`: load `/codex-exec`, invoke `codex exec` via Bash
   - `-a opus`: use the platform's Opus-class worker only when that runner exists
   - Mixed (`-a thinker:3 -a doer:2`, `-a codex:3 -a opus:2`): route by type, unify results

5. **Collect + failure handling**: partial failure -> retry (max 2) -> 2+ successes continue, 1 = degrade to single result, all fail = report error

6. **Post-processing** (mode-dependent)
   - split: return each agent's output separately. No aggregation.
   - sample: inject N outputs into prompt, synthesize final answer
     > "Identify common points and differences, compare whether evidence points to the same source, synthesize evidence-backed strengths, discard unsupported conclusions"

7. **Verify (mandatory for sample mode)**: go back to source and verify synthesized conclusions against actual code, docs, or command output. Thinking models (Codex/Thinker) are strong at finding problems but prone to overthinking -- they may invent issues or propose fixes that add unnecessary complexity. Always verify findings against source before surfacing to the user. Skip verification only for pure creative tasks with no codebase or executable environment.
