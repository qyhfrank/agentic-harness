---
name: critique
description: 'Use when code, plan, or config changes need structured review with source-anchored findings, such as spec checks, quality gates, plan review, or final multi-angle review. Triggers on /critique, "review this", "code review", "review the plan", "review the config", or "find blocking issues"; do not use to respond to existing review feedback after comments arrive.'
argument-hint: <what to review> [--quality|--plan]
---

Arguments: $ARGUMENTS

# Critique

Structured review with source-anchored findings. Select a profile, get a verdict.

## Profiles

| Profile | Question | Reviewers |
|---|---|---|
| `quality` | Does implementation match requirements and is it clean, tested, maintainable? | 1 |
| `plan` | Is this plan complete, coherent, and ready to execute? | 1 |
| `full` | Multi-angle review with role diversity | N via `/fanout` |

Arguments:

- `--quality`: single reviewer, quality + spec compliance
- `--plan`: single reviewer, plan/config artifact
- No flag: full multi-reviewer review via `/fanout`

## Verdict Envelope

All profiles return:

```markdown
**Verdict:** pass | fail | needs_escalation
**Findings:** F-001, F-002, ... (or "none")
**Assessment:** <one-line summary>
```

- `pass`: no blocking findings.
- `fail`: blocking findings exist, must address.
- `needs_escalation`: cannot determine, needs broader context or human judgment.

Single-reviewer verdicts are gate inputs, not completion oracles — inside `/harness`, a single-reviewer pass contributes to `review_streak` but does not satisfy `close_rule` on its own.

## Profile: quality

Templates: `references/quality-review-profile.md`, `references/code-reviewer.md`

Input: BASE_SHA, HEAD_SHA, implementation description, plan/requirements reference. First verifies spec compliance (did you build what was asked?), then checks code quality, architecture, and testing.

## Profile: plan

Template: `references/plan-review-profile.md`

Input: planning artifact (plan doc, harness config.yaml, task strategy) + goal/spec reference. Use after plan mode, harness scaffold/plan, or any planning artifact is ready. Checks completeness, goal alignment, boundary coverage, verification readiness.

## Profile: full

Multi-angle review via `/fanout`.

### When to Use

Required: after completing major features, before merging to main.
Recommended: when stuck (fresh perspective), before refactoring (establish baseline), after complex bug fixes.

### Reviewer Allocation

Reviewer count based on diff size (user override takes precedence):

- Single file / <50 lines diff: 3
- Multi-file / 50-200 lines diff: 5 (fanout default)
- Architectural change / >200 lines diff: 5 + specialized roles as needed

Row count, naming, and focus angles are dynamically generated per task. One reviewer lane should take a skeptical stance — actively look for failure modes (auth boundaries, data loss, race conditions, rollback safety, stale state, schema drift) rather than validating the change.

Main agent critically merges reviewer outputs and filters unsupported suggestions.

## Invocation Boundary

- Invoked by the highest context owning final review judgment.
- Content-producing child workers (implementer, reviewer, researcher) should not recursively invoke `/critique`.
- `/planning` per-task gates use `--quality` (single reviewer). Final review uses bare `/critique` (full profile).

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

Full-profile: main agent verifies all blocking finding anchors point to real source locations before returning the verdict.

## Reviewer Policy

Three core rules for all reviewer findings:

1. **Ground in source.** Every finding must cite a real code path, file, and line. Do not report issues based on generic experience or hypothetical scenarios.
2. **Suppress speculative findings.** If the issue stems from a path not triggered by the current implementation, or requires material inference about future usage, do not report it.
3. **Minimize false positives.** Default to `near-blocking`; upgrade to `blocking` only with evidence of real error or production risk. Do not recommend adding abstractions or safety mechanisms beyond what current code paths require.

## Execution

### Single Reviewer (--quality, --plan)

1. Parse arguments, determine profile.
2. Spawn single reviewer child context with profile's prompt template.
3. Collect verdict envelope.
4. Verify finding anchors point to real source locations.
5. Return verdict.

### Full Review (default)

1. Load `/fanout`.
2. Dispatch reviewers with `-m sample`. Dynamically generate reviewer roles and focus angles.
3. Collect outputs, critically merge findings.
4. Source-verify all blocking findings.
5. Return verdict envelope.

## Reference Index

| File | Purpose | Used by |
|---|---|---|
| `references/quality-review-profile.md` | Quality + spec compliance reviewer prompt | quality, full profiles |
| `references/plan-review-profile.md` | Plan/config reviewer prompt | plan profile |
| `references/code-reviewer.md` | Base code review agent prompt + policy | quality, full profiles |
