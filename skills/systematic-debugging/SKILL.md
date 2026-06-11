---
name: systematic-debugging
description: Systematic root-cause analysis for bugs, test failures, or unexpected behavior. Apply before proposing fixes. Not a replacement for /harness iteration loops.
---

# Systematic Debugging

## Overview

Find root cause before attempting fixes; symptom fixes are failure. This holds under time pressure and for simple-looking issues. No fixes without completing Phase 1.

## The Four Phases

Complete each phase before the next.

### Phase 1: Root Cause Investigation

1. **Read error messages completely** — stack traces, line numbers, error codes. They often contain the solution.
2. **Reproduce consistently** — exact steps, reliable trigger. Not reproducible → gather more data, don't guess.
3. **Check recent changes** — git diff, recent commits, new dependencies, config and environment differences.
4. **Gather evidence in multi-component systems** (CI → build → signing, API → service → database):

   Before proposing fixes, add diagnostic instrumentation at each component boundary:

   ```
   For EACH component boundary:
     - Log what data enters component
     - Log what data exits component
     - Verify environment/config propagation
     - Check state at each layer
   ```

   Run once to gather evidence showing WHERE it breaks, then investigate that component.

   Example (multi-layer system):

   ```bash
   # Layer 1: Workflow
   echo "=== Secrets available in workflow: ==="
   echo "IDENTITY: ${IDENTITY:+SET}${IDENTITY:-UNSET}"

   # Layer 2: Build script
   echo "=== Env vars in build script: ==="
   env | grep IDENTITY || echo "IDENTITY not in environment"

   # Layer 3: Signing script
   echo "=== Keychain state: ==="
   security list-keychains
   security find-identity -v

   # Layer 4: Actual signing
   codesign --sign "$IDENTITY" --verbose=4 "$APP"
   ```

   This reveals which layer fails (secrets → workflow ✓, workflow → build ✗).

5. **Trace data flow** — when the error is deep in the call stack, trace the bad value backward to its origin; fix at source, not at symptom. Full technique: `root-cause-tracing.md` in this directory.

### Phase 2: Pattern Analysis

- Find similar working code in the same codebase.
- When implementing a pattern, read the reference implementation completely before applying it.
- List every difference between working and broken, however small.
- Identify dependencies: components, settings, config, environment, assumptions.

### Phase 3: Hypothesis and Testing

- Form a single specific hypothesis: "X is the root cause because Y."
- Test with the smallest possible change, one variable at a time.
- Confirmed → Phase 4. Not confirmed → new hypothesis; don't stack more fixes on top.
- When you don't understand something, say so and research; don't pretend to know.

### Phase 4: Implementation

1. **Create failing reproduction first** — simplest reproduction, automated test if possible, one-off script if no framework. It must fail for the expected reason before you change code. If the surrounding workflow keeps round evidence, record the failing reproduction, proof command, and failure evidence before patching.
2. **Implement single fix** — smallest change addressing the identified root cause. One change at a time; no "while I'm here" improvements or bundled refactoring.
3. **Verify** — failing reproduction passes, no other tests broken, issue actually resolved. Keep regression coverage when automation is possible.
4. **Fix didn't work** — return to Phase 1 and re-analyze with the new information. Don't attempt a fourth fix without the architecture discussion below.
5. **3+ failed fixes = architectural problem, not a failed hypothesis.** Signs: each fix reveals new coupling elsewhere, fixes require massive refactoring, each fix creates new symptoms. Stop fixing symptoms; question whether the pattern is fundamentally sound and discuss with the user before further attempts.

## Quick Reference

| Phase | Key Activities | Success Criteria |
|-------|---------------|------------------|
| **1. Root Cause** | Read errors, reproduce, check changes, gather evidence | Understand WHAT and WHY |
| **2. Pattern** | Find working examples, compare | Identify differences |
| **3. Hypothesis** | Form theory, test minimally | Confirmed or new hypothesis |
| **4. Implementation** | Create test, fix, verify | Bug resolved, tests pass |

## When Process Reveals "No Root Cause"

If systematic investigation shows the issue is truly environmental, timing-dependent, or external: document what you investigated, implement appropriate handling (retry, timeout, error message), and add monitoring/logging for future investigation. Most "no root cause" conclusions are incomplete investigation.

## Supporting Techniques

Available in this directory:

- **`root-cause-tracing.md`** - Trace bugs backward through call stack to find original trigger
- **`defense-in-depth.md`** - Add validation at multiple layers after finding root cause
- **`condition-based-waiting.md`** - Replace arbitrary timeouts with condition polling
