# Tactical Planning

Discipline for generating high-precision approach steps. Loaded by Propose when the active approach needs tactical-level guidance. Complements `plan.md` (which defines the `steps[]` data model) with content standards.

## When to Generate Tactical Steps

Not every approach needs tactical steps. Generate them when:

- The approach involves creating or modifying 2+ files
- The approach requires a specific sequence of changes to stay green
- `task.protocol` is `tdd_required` or `tdd_preferred`

For single-file changes or exploratory spikes, short descriptive steps (as shown in `plan.md` examples) are sufficient.

## Step Generation Protocol

When generating tactical steps for an approach, each step must specify:

1. **Files touched** — exact repo-relative paths; distinguish create vs modify
2. **Content** — code blocks showing what to write or change; no prose-only descriptions for code steps
3. **Verification** — exact command to run and expected outcome (pass/fail pattern)

Step granularity: one action per step. If a step description needs "and", split it. Target: each step is one harness round.

### File Mapping

Before generating steps, list all files the approach will create or modify:

```
Files:
  Create: src/rate_limiter.py
  Modify: src/middleware.py:45-60
  Test:   tests/test_rate_limiter.py
```

This mapping locks in the approach scope. Steps reference these paths; no file appears in a step that wasn't listed here.

### TDD Rhythm

When `task.protocol` is `tdd_required` or `tdd_preferred`, steps follow this cycle:

1. Write failing test (code block with test)
2. Verify test fails (command + expected failure pattern)
3. Write minimal implementation (code block)
4. Verify test passes (command + expected pass)

Load `references/tdd-discipline.md` for anti-patterns and the pre-dispatch checklist. The discipline there governs test quality; this file governs step structure.

### Step Content Format

Each step in `steps[]` is a short string in plan.yaml, but during Propose the agent expands it into a round with full content. The tactical discipline governs that expansion:

```yaml
# plan.yaml (compact)
steps:
  - "write failing test for RateLimiter.allow()"
  - "implement RateLimiter.allow() with sliding window"
  - "integrate RateLimiter as middleware"

# Propose expansion for step 0 (not stored, used during round)
# Files: tests/test_rate_limiter.py (create)
# Code: [full test code block]
# Verify: pytest tests/test_rate_limiter.py -v → FAIL "RateLimiter not defined"
```

Steps stored in plan.yaml remain short strings. The tactical expansion happens at Propose time, informed by this reference.

## No-Placeholder Rules

These patterns in step expansions are failures — fix before committing:

- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" without actual test code
- "Similar to step N" (repeat the content; each round is self-contained)
- Steps that describe what to do without showing how
- References to types or functions not yet defined in any prior step or existing code

## Integration with plan.yaml

Tactical steps populate the `approach.steps[]` field during:

- **Bootstrap Strategy** (run initialization): for the first active milestone's active approach
- **Milestone advance**: when a new milestone activates and its approach needs tactical steps
- **Replan**: when new approaches are generated for affected milestones

Steps are advisory (as defined in `plan.md`): Propose may skip, merge, or reorder if evidence warrants, but must record deviations in `Decisions`.

When `current_step >= len(steps)` and `exit_criteria` not met, fall back to hypothesis-level Propose per `plan.md` rules.
