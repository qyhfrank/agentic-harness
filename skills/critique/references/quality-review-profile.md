# Code Quality Review Profile

Single-reviewer profile for verifying implementation quality and spec compliance. Uses `code-reviewer.md` as the base review template.

## Reviewer Prompt Template

```
Foreground child context:
  role: leaf code quality reviewer
  Use template at ./code-reviewer.md

  WHAT_WAS_IMPLEMENTED: {IMPLEMENTATION_SUMMARY}
  PLAN_OR_REQUIREMENTS: {REQUIREMENTS_REFERENCE}
  BASE_SHA: {BASE_SHA}
  HEAD_SHA: {HEAD_SHA}
  DESCRIPTION: {TASK_DESCRIPTION}
```

**Phase 1 — Spec compliance.** Do NOT trust the implementer's report. Read the actual code and verify:

- Did they implement everything that was requested? Are there requirements skipped or missed?
- Did they build things that weren't requested? Over-engineering or unnecessary features?
- Did they interpret requirements differently than intended? Solve the wrong problem?

If spec compliance fails, stop and return `fail` — do not proceed to quality checks.

**Phase 2 — Code quality.** In addition to the standard review policy in `code-reviewer.md`:

- Does each file have one clear responsibility with a well-defined interface?
- Are units decomposed so they can be understood and tested independently?
- Is the implementation following the file structure from the plan?
- Did this implementation create new files that are already large, or significantly grow existing files?
- Is the implementation appropriately simple for its scope, or does it introduce structure (abstractions, indirection, extension points) that no current consumer needs?
- Can newly added logic reuse an existing helper or utility instead of duplicating behavior?

**Output:** Findings in F-NNN format with verdict envelope.
