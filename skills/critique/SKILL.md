---
name: critique
description: 'Use when code, plan, or config changes need structured review with source-anchored findings. Triggers on /critique, "review this", "code review", "review the plan", "review the config", or "find blocking issues"; do not use to respond to existing review feedback after comments arrive.'
argument-hint: <what to review> [--quality|--plan]
---

Arguments: $ARGUMENTS

# Critique

Structured review with source-anchored findings. One reviewer, one verdict.

## Profiles

- `--quality` (default): Does implementation match requirements, and is it clean, tested, maintainable?
- `--plan`: Is this plan complete, coherent, and ready to execute?

For multi-angle review, use `/fanout` with `/critique`.

## Verdict Envelope

```markdown
**Verdict:** pass | fail | needs_escalation
**Findings:** F-001, F-002, ... (or "none")
**Assessment:** <one-line summary>
```

- `pass`: no blocking findings.
- `fail`: blocking findings exist, must address.
- `needs_escalation`: cannot determine, needs broader context or human judgment.

Single-reviewer verdicts are gate inputs, not completion oracles — inside `/harness`, a pass contributes to `review_streak` but does not satisfy `close_rule` on its own.

## Finding Format

```markdown
### F-001 · `blocking`

**Finding:** concrete factual description
**Impact:** short impact
**Trigger:** short trigger sentence

**Anchor:** path:line
**Minimal Fix:** shortest safe change
```

Severity: `blocking` (must fix), `near-blocking` (should fix). `non-blocking` only when user explicitly requests.

## Quality Review Checklist

Spawn a single reviewer child context with the git range (`BASE_SHA..HEAD_SHA`), implementation description, and plan/requirements reference.

**Phase 1 — Spec compliance.** Do NOT trust the implementer's report. Read the actual code and verify:

- Did they implement everything requested? Requirements skipped or missed?
- Did they build things that weren't requested?
- Did they interpret requirements differently than intended?

If spec compliance fails, stop and return `fail`.

**Phase 2 — Code quality.**

- Does each file have one clear responsibility?
- Are units decomposed for independent understanding and testing?
- Is the implementation following the file structure from the plan?
- Is it appropriately simple, or does it introduce structure no current consumer needs?
- Can newly added logic reuse an existing helper instead of duplicating behavior?

## Plan Review Checklist

Spawn a single reviewer child context with the planning artifact and goal/spec reference.

**For plan documents:** completeness (no TODOs/placeholders), goal alignment (no scope creep), task decomposition (atomic, clear boundaries), dependency ordering, file scope (no overlapping writes), verification (each task has testable acceptance criteria).

**For harness config.yaml:** boundary completeness (mutable/immutable paths cover scope), checks have runnable commands, acceptance criteria are concrete, budget/stagnation limits are reasonable.

## Reviewer Policy

1. **Ground in source.** Every finding must cite a real code path, file, and line.
2. **Suppress speculative findings.** If the issue stems from a path not triggered by current implementation, do not report it.
3. **Minimize false positives.** Default to `near-blocking`; upgrade to `blocking` only with evidence of real error or production risk.

## Execution

1. Parse arguments, determine profile.
2. Spawn single reviewer child context with the profile's checklist.
3. Collect verdict envelope.
4. Verify finding anchors point to real source locations.
5. Return verdict.
