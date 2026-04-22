# Using Agents

## Agent Types

- **Thinker**: depth-first reasoning — planning, review, RCA, architecture. Treat findings as hypotheses; prefer simpler fixes. Thinkers tend to over-infer: may invent problems or propose fixes that add unnecessary complexity. Always verify against source before surfacing.
- **Doer**: execution-first — code changes, tests, mechanical refactors.

Per-platform mapping:

| Type | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Thinker | `/codex-exec` (MUST LOAD THIS SKILL FIRST!) | `@agent-gpt-5.4-xhigh` | native `gpt-5.4 xhigh` |
| Doer | self / Agent / Task | `@agent-opus` | `/claude-exec` (MUST LOAD THIS SKILL FIRST!) |

## Execution Surfaces

Choose execution surface separately from agent type.

| Capability | Claude Code | OpenCode | Codex CLI |
|---|---|---|---|
| Foreground tool execution (Bash etc.) | R/W | R/W | R/W |
| Background tool execution | R/W | Not supported | R/W |
| Foreground child context (blocking agent) | R/W | R/W | R/W |
| Background child context | R/W | Read-only | R/W |
| Team orchestration (shared task list / messaging) | Supported | Not supported | Not supported |

Prefer simplest surface for current step. Deterministic waits and scriptable monitors stay with tools unless ongoing reasoning needed.

## Hard Rules

1. **max_depth = 1** by default. Children are `leaf` (`child_budget = 0`) unless caller grants deeper budget.
2. **Leaf agents cannot spawn.** `role = leaf`, `child_budget = 0`, or `depth >= max_depth` prohibits Agent, Task, or orchestration skill calls.
3. **Budget is hard.** Every new agent context (incl. retries) consumes `child_budget`. Child may tighten but never expand inherited budget.
4. **No cycles.** Never invoke an orchestrator or runner already in call stack.
5. **Child prompts must be self-contained.** Include scope, goal, constraints, expected output format. No implied context.
6. **Background delegation is read-only or advisory by default.** Concurrent code edits require partitioned file/resource ownership per child before dispatch; execution surface must support safe writes.
7. **Parent owns verification.** Subagent findings are hypotheses until parent verifies against source, code, or tool output.
8. **No silent substitution.** If child dispatch fails or execution shape changes, state it explicitly. Never fallback to local inline work automatically without users' approval.

## Dispatch Protocol

### Thinker Routing

Route to Thinker when any apply:

- Problem without clear solution path.
- Multiple valid approaches, best not obvious.
- Architecture, tradeoffs, or quality judgment involved.
- Reasoning spans multiple files or systems.

Skip: single-file mechanical changes, direct lookups, tasks with specific implementation plan.

### Execution Shape

Before implementation with 3+ independent subtasks or 3+ unrelated files, choose execution shape:

1. **Local / Foreground child** — work fits single context. Delegate to one blocking child when parent context is insufficient.
2. **Tool execution** — deterministic wait or scriptable monitor.
3. **Same-wave fanout** — subtasks independent, no shared-write conflicts.
4. **Background child** — read-only or advisory work while parent has independent work.
5. **Team orchestration** — task decomposition will evolve mid-flight and platform supports it.

Quick test:

- All child prompts writable before any starts → fanout.
- Workers need repartitioning after findings → team orchestration or stay local.

## Agent Context Card

Every child prompt opens with plain-text Agent Context Card. Replace every placeholder with concrete value.

Template:

> Agent: <role> | depth <current_depth>/<max_depth> | budget <child_budget> | parent: <parent_role>

Meaning:

- `role`: child role for current workflow
- `current_depth`: child's depth in delegation tree
- `max_depth`: max delegation depth for run
- `child_budget`: remaining child-agent spawns (not token or time budget)
- `parent_role`: immediate parent workflow or agent role
