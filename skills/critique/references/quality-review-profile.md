# Code Quality Review Profile

Single-reviewer profile for verifying implementation quality. Uses `code-reviewer.md` as the base review template.

**Prerequisite:** Only dispatch after spec compliance review passes.

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

**In addition to standard review policy, the reviewer should check:**

- Does each file have one clear responsibility with a well-defined interface?
- Are units decomposed so they can be understood and tested independently?
- Is the implementation following the file structure from the plan?
- Did this implementation create new files that are already large, or significantly grow existing files?

**Output:** Findings in F-NNN format with verdict envelope (see `code-reviewer.md`).
