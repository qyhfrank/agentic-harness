# Plan Document Reviewer Prompt Template

Use this template when dispatching a plan document reviewer child context.

**Purpose:** Verify the plan chunk is complete, matches the spec, and has proper task decomposition.

**Dispatch after:** Each plan chunk is written

This file defines the prompt body and role contract only. Wrap it with the platform's foreground child-context surface. Do not hardcode platform-only tool syntax here.

```
Foreground child context:
  role: leaf plan document reviewer
  description: "Review plan chunk N"
  prompt: |
    You are a plan document reviewer. Verify this plan chunk is complete and ready for implementation.

    **Plan chunk to review:** [PLAN_FILE_PATH] - Chunk N only
    **Spec for reference:** [SPEC_FILE_PATH]

    ## What to Check

| Category           | What to Look For                                                                               | Scope      |
| ------------------ | ---------------------------------------------------------------------------------------------- | ---------- |
| Completeness       | TODOs, placeholders, incomplete tasks, missing steps                                           | per-chunk  |
| Spec Alignment     | Chunk covers relevant spec requirements, no scope creep                                        | per-chunk  |
| Task Decomposition | Tasks atomic, clear boundaries, steps actionable                                               | per-chunk  |
| File Structure     | Files have clear single responsibilities, split by responsibility not layer                    | per-chunk  |
| File Size          | Would any new or modified file likely grow large enough to be hard to reason about as a whole?  | per-chunk  |
| Task Syntax        | Checkbox syntax (`- [ ]`) on steps for tracking                                               | per-chunk  |
| Task IDs           | Every task has a `**Task ID:**` and `**Depends on:**` field                                    | per-chunk  |
| Execution Summary  | Plan has an Execution Summary section with mutable/immutable paths, implementation protocol, verification, criteria | chunk 1 only |
| Chunk Size         | Each chunk under 1000 lines                                                                    | per-chunk  |

    ## CRITICAL

    Look especially hard for:
    - Any TODO markers or placeholder text
    - Steps that say "similar to X" without actual content
    - Incomplete task definitions
    - Missing implementation protocol or task-level override where one is needed
    - Missing verification steps or expected outputs
    - Files planned to hold multiple responsibilities or likely to grow unwieldy

    ## Output Format

    ## Plan Review - Chunk N

    **Status:** Approved | Issues Found

    **Issues (if any):**
    - [Task X, Step Y]: [specific issue] - [why it matters]

    **Recommendations (advisory):**
    - [suggestions that don't block approval]
```

**Reviewer returns:** Status, Issues (if any), Recommendations
