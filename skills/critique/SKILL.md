---
name: critique
description: Use when code changes need structured review with anchored findings, including spec compliance gates, code quality gates, plan/config review, or full multi-angle reviews. Triggers on /critique, "review this", "code review", "quality check", "spec compliance check", "review the plan", "review the config", "find blocking issues".
argument-hint: <what to review> [--spec|--quality|--plan] [-a thinker|doer|codex|opus] [--refactor]
---

Arguments: $ARGUMENTS

# Critique

Select profile + engine → structured findings with source anchors.

## Profiles and Engines

| Profile | Question | Default Engine |
|---|---|---|
| `spec` | Does implementation match requirements? | `single` |
| `quality` | Clean, tested, maintainable? | `single` |
| `plan` | Complete, coherent, ready to execute? | `single` |
| `full` | Multi-angle review with role diversity / cross-validation | `fanout` |

| Engine | How |
|---|---|
| `single` | One reviewer child context, no `/fanout`. Focused per-task gates. |
| `fanout` | Parallel reviewers via `/fanout`. Full reviews and merge gates. |

Arguments:

- `--spec`: profile=spec, engine=single
- `--quality`: profile=quality, engine=single
- `--plan`: profile=plan, engine=single
- No flag: profile=full, engine=fanout
- `-a thinker|doer|codex|opus`: worker override for fanout engine. No flag → `/fanout` infers.
- `--refactor`: append optional refactor proposals (full profile only)

## Verdict Envelope

All profiles return:

```markdown
**Verdict:** pass | fail | needs_escalation
**Findings:** F-001, F-002, ... (or "none")
**Assessment:** <one-line summary>
**Coverage:** requirements checked, files reviewed, risk dimensions assessed
**Unchecked:** areas not reviewed or not reviewable via static analysis
```

- `pass`: no blocking findings in checked scope. Bounded by coverage, not absolute correctness.
- `fail`: blocking findings exist, must address.
- `needs_escalation`: cannot determine, needs broader context or human judgment.

Coverage mandatory when verdict=pass. Single-reviewer verdicts are gate inputs, not completion oracles — inside `/harness`, a single-reviewer pass does not grant close authority.

## Profile: spec

Template: `references/spec-review-profile.md`

Input: task requirements + implementer's claimed changes. Checks missing requirements, extra unneeded work, misunderstandings. Verify by reading code, not trusting the report.

## Profile: quality

Templates: `references/quality-review-profile.md`, `references/code-reviewer.md`

Input: BASE_SHA, HEAD_SHA, implementation description, plan/requirements reference. Prerequisite: spec compliance must pass first when both profiles run in sequence.

## Profile: plan

Template: `references/plan-review-profile.md`

Input: planning artifact (plan doc, harness config.yaml, task strategy) + goal/spec reference. Use after plan mode, harness scaffold/plan, or any planning artifact is ready. Checks completeness, goal alignment, boundary coverage, verification readiness.

## Profile: full

Multi-angle review.

### When to Use

Required: after completing major features, before merging to main.
Recommended: when stuck (fresh perspective), before refactoring (establish baseline), after complex bug fixes.

### Strategy Selection

Main agent selects strategy:

- **Strategy R** (Role-Diverse): broad changes, architecture boundaries, multi-risk dimensions.
- **Strategy H** (Homogeneous Cross-Validation): local changes, single goal, reduce false-negative randomness.
- User-specified strategy takes precedence.

Over-engineering check: dedicated role under Strategy R, folded into unified prompt under Strategy H.

### Row and Prompt Generation

- Row count, naming, focus angles: dynamically generated per task.
- Reviewer count heuristic (user override takes precedence):
  - Single file / <50 lines diff → 3
  - Multi-file / 50–200 lines diff → 5 (fanout default)
  - Architectural change / >200 lines diff → 5 + specialized roles as needed

### Core Thinking

- Role diversity → coverage (different angles).
- Homogeneous sampling → stability (reduce randomness).
- Sample aggregation: main agent critically merges, filters unsupported suggestions.

## Invocation Boundary

- Invoked by the highest context owning final review judgment.
- Content-producing child workers (implementer, reviewer, researcher) should not recursively invoke `/critique`.
- `/planning` per-task gates use `--spec` / `--quality` (single engine). Final review uses bare `/critique` (full profile).

## Output Contract

Default: output `blocking` and `near-blocking` only. `non-blocking` only when user explicitly requests.

Finding format (all profiles):

```markdown
### F-001 · `blocking`

**Finding:** concrete factual description
**Impact:** short impact
**Trigger:** short trigger sentence

**Anchor:** path:line
**Minimal Fix:** shortest safe change
```

Full-profile main agent verification:

```markdown
### Main Verification · F-001

**Status:** verified
**Evidence Anchor:** path:line
**Note:** concise verification note
```

Rendering: markdown code block, bold field names, field order Finding → Impact → Trigger → Anchor → Minimal Fix. Pack short fields; separate long fields with blank lines. Blockquote format only when user explicitly requests.

With `--refactor`, append:

```markdown
### Optional Refactor · R-001

**Proposal:** ...
**Benefit:** ...
**Cost:** ...
**Risk:** ...
**Rollback:** ...
```

## Reviewer Policy

All reviewer findings must:

- Confirm actual alignment units, consumer boundaries, and semantic owners before judging where the problem lies.
- Suppress findings that stem only from wrong analysis granularity, generic-experience guessing, or paths not triggered by current implementation.
- Keep skill and reviewer prompts project-agnostic; project-specific terms only as evidence anchors from reviewed code.

Default to `near-blocking` for these signals; upgrade to `blocking` only with proof of real error or production risk:

- Legacy config key / lenient parsing with no real historical consumers
- Debug switches or observability structures exposed to external config in no-op stages
- Per-forward wrapper/hook or extra abstraction layer outside plan scope
- Shadow state variables threading through runtime (should be init assertions)
- Unplanned scope expansion
- Config bridge injection causing ownership confusion

These findings must also: state concrete facts (no vague "over-engineering" labels), describe the trigger path or consumer relationship in current implementation, and propose the minimal safe fix.

## Execution

### Single Engine (--spec, --quality, --plan)

1. Parse arguments, determine profile.
2. Spawn single reviewer child context with profile's prompt template.
3. Collect verdict envelope.
4. Main agent verifies anchors point to real source locations (lightweight check).
5. Return verdict.

### Fanout Engine (full, default)

1. Parse arguments. Pass `-a` to `/fanout` if present. If `codex` worker requested, load `/codex-exec` via Skill tool first.
2. Load `/fanout`.
3. Select Strategy R or H; dispatch reviewers with `-m sample` and parsed `-a` argument.
4. Collect outputs, run sample aggregation.
5. Main agent source-verifies all blocking findings; output `Main Verification` blocks.
6. Return verdict envelope.

## Reference Index

| File | Purpose | Used by |
|---|---|---|
| `references/spec-review-profile.md` | Spec compliance reviewer prompt | spec profile |
| `references/quality-review-profile.md` | Code quality reviewer prompt wrapper | quality profile |
| `references/plan-review-profile.md` | Plan/config reviewer prompt | plan profile |
| `references/code-reviewer.md` | Base code review agent prompt + policy | quality, full profiles |
