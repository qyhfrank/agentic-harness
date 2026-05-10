---
name: critique
description: Use when the user requests a structured review or needs to surface blocking issues.
reviewer_model: gpt
---

# Critique

Critique dispatches GPT reviewers, verifies source, owner-adjudicates, and returns a verdict. Do not route critique findings through `receiving-code-review`; that skill is for human-supplied or externally authored feedback only.

## Hard Rules

- MUST use GPT reviewers. `reviewer_model: gpt` is binding.
- MUST dispatch via `/fanout -m sample -a gpt:2` per selected goal.
- MUST confirm reviewer agent type = `gpt` before dispatch. If not, STOP.
- MUST collect 2 independent GPT reviewer outputs per goal before GSA.
- MUST adjudicate in the critique owner during Step 5. DO NOT dispatch a separate adjudicator agent.
- DO NOT run inline review. Inline review is not allowed.
- DO NOT substitute non-GPT reviewers after launch failure. Let `/fanout` retry; if cardinality remains unmet, return `needs_escalation`.
- DO NOT pressure an artifact to implement explicitly deferred or out-of-scope work.
- Only `required_fix` justifies new guards, state, schema, validation infrastructure, or broad tests by default.

## Step 1: Load skill `/fanout`

Claude Code: load `/codex-exec` first. Codex CLI: use native gpt-5.5 xhigh.

## Step 2: Plan the review goals

### Case 1: Code

#### Goal 1: Spec and intent
- Are explicit requirements, acceptance criteria, and called-out scenarios covered?
- Are existing behavior, interfaces, data contracts, and compatibility preserved unless requested to change?
- Does the implementation address the root problem or user goal, not only a surface symptom?
- Are incomplete, conflicting, or unknowable specs called out?
- If a surface is explicitly deferred or out of scope, are findings limited to current risks instead of pressuring the artifact to implement deferred work?

#### Goal 2: Functional correctness
- Within agreed behavior, are happy paths, edge cases, nulls, invalid inputs, state transitions, return values, and runtime behavior correct?
- Do side effects happen only under the right conditions, time, and scope?

#### Goal 3: Validation and evidence
- Are tests, assertions, logs, metrics, probes, or manual checks sufficient for key conclusions?
- Does validation cover critical paths, boundaries, and major failure modes?
- Are conclusions evidence-backed rather than "the code looks right"?
- If closing a gap requires new heavy browser/DOM harnesses, test frameworks, or validation infrastructure not already used by the repo, prefer `near-blocking` and recommend a lighter deterministic probe unless there is concrete bug evidence.

#### Goal 4 (optional): Risk and failure safety
Add when the change touches permissions/trust boundaries, durable state/data writes, concurrency, retry, rollback, or idempotency.
- Could partial failure, retry, rollback, or interruption leave bad state, corruption, or dangerous side effects?
- Could stale state, races, ordering dependencies, idempotency gaps, or cross-boundary interactions break behavior?
- Are permissions, trust boundaries, and dangerous capabilities constrained?

#### Goal 5 (optional): Structure, simplification, and maintainability
Add for large refactors, new abstractions, high coupling, unclear ownership, or hot-path/perf-sensitive code.
- Prefer the smallest structure preserving clarity, correctness, and testability.
- Reuse local helpers instead of duplicate or hand-rolled string/path/env/type checks.
- Prefer fewer helpers, wrappers, branches, return values, aliases, and state; remove unused returns/params, thin wrappers, dead generalization, and structures with no current consumer.
- Keep a helper only when it removes real duplication, captures a stable semantic boundary, or creates a materially better test seam.
- Remove unnecessary work, missed concurrency, hot-path bloat, pre-check-before-operate, missing cleanup, and overly broad reads/loads.

For structure findings, prefer the smallest safe diff: delete, inline, narrow, or reuse before adding layers. Skip stylistic, speculative, or scope-expanding suggestions.

#### Additional optional goals
Compatibility and migration, concurrency and timing, data/state consistency, observability/operability, performance/resource usage, and similar angles.

### Other cases

For non-code reviews (plans/task lists), derive the smallest goal set from provided material: completeness, boundaries, dependencies, acceptance, risk. Launcher must supply artifact and enough context. Anchor goals/findings/verification to provided material only; do not inspect unrelated workspace or adjacent files. If insufficient, return `needs_escalation` and name missing context.

### Review capsule

Before dispatch, define target/revision, goal, selected goals, non-goals, and boundaries. Smallest goal set for the decision: simplification = Goal 1 + Goal 5; durable-state/permission/concurrency/retry/rollback risk adds Goal 4. Missing material context => `needs_escalation`.

For harness tasks, scope reviewers to `boundary.mutable` plus the current round's touched files by default. Treat `boundary.immutable`, unrelated dirty files, and pre-existing workspace drift as out of scope unless explicitly included. Include this boundary in the reviewer prompt.

For git branches/worktrees, include base, candidate, and diff range. For harness tasks, derive base from task baseline or round base, not current `main`. If `main` or base branch advanced since setup, do not use `main..HEAD` unless explicit. Unknown base => `needs_escalation`; request it before dispatch.

## Step 3: Dispatch reviewers via `/fanout -m sample`

Gate check: Confirm reviewer agent type = gpt. If not, STOP.

For each selected goal, dispatch 2 independent GPT reviewers with the same prompt:

```bash
/fanout -m sample -a gpt:2
```

Total reviewers required: `2 * selected goals`.
If dispatch is blocked, let `/fanout` retry. Return `needs_escalation` only after retry exhausts with unmet cardinality; no reviewer substitution or inline review.

### Prompt for Reviewers

- Agent Context Card, e.g. `Agent: reviewer | depth 1/1 | budget 0 | parent: critique`
- Reviewer agent type: `gpt`
- Review target and scope materials: diff, files, plan, or artifact. For git targets, include base revision, candidate revision, and diff range.
- Goal name, review angle, coverage target, and review standard.
- Finding constraints: non-speculative issues with real anchors only; skip stylistic/speculative; avoid scope expansion unless required for a verified issue; deferred surfaces are findings only when artifact claims support or omission creates concrete current risk.
- Preserve explicit user wording and surfaces. If context gives exact labels, output shape, fields, or "do not use X", do not propose alternates unless required by a verified bug; prefer delete, hide, narrow, or reuse.
- Validation gaps: prefer the smallest proof for the current slice. If stronger proof requires new heavy infrastructure not already in the repo, default to `near-blocking` unless concrete bug evidence exists.
- Minimal Fix: smallest safe change. If unsafe, recommend minimal refactor, bound scope, list verification. Goal 5: prefer delete/inline/narrow over layers.
- Severity: `blocking` = evidence of real bug or production risk; `near-blocking` = issue exists but lacks direct failure evidence.
- Reviewer `Recommendation` is advisory; the critique owner recomputes actionability and recommendation after source verification.

```markdown
### F-001

Severity: `near-blocking` / `blocking`
Finding: <factual description>
Impact: <impact description>
Trigger: <trigger condition>
Anchor: <path:line | step/item/section> (required; must be the real triggering location)
Minimal Fix: <smallest safe change>
Recommendation:
- Should fix: yes / nice-to-have / no
- Fix now: now / next-round / backlog
- How: <one sentence describing minimal action>
```

`**Verdict:**` and `F-xxx` cards are valid only after reviewer dispatch and source verification.

## Step 4: Collect reviewer outputs

Wait for all `/fanout` reviewers. Require 2 independent GPT outputs per goal with real `Anchor` before synthesis or verification.

If reviewer-pair cardinality is unmet after `/fanout` exhausts retry, stop and return `needs_escalation` with the concrete launch-failure reason.

## Step 5: Source-material verification and owner adjudication

Start GSA (Generative Self-Aggregation):
1. Deduplicate reviewer findings and merge same-root issues.
2. Read source at each anchor. For code, read files/lines; for non-code, read steps/items/sections in the provided material.
3. Discard unverified findings. Findings raised multiple times deserve extra scrutiny, not automatic acceptance.
4. Classify each verified finding.
5. Assign actionability and recommendation.

A verified finding may still be out of current scope. When several findings reduce to one deferred-surface mismatch, collapse them and prefer trim/split/delete over implementing deferred work.

Classification taxonomy:

- `spec-required bug`: requirement or compatibility violation.
- `current production risk`: concrete current runtime, data, security, availability, or user-visible failure.
- `validation gap`: weak evidence without a proven bug.
- `cleanup/simplification`: acceptable behavior with excess breadth, duplication, or complexity.
- `out-of-scope policy/feature`: deferred surface, new policy, or adjacent behavior.

Actionability:

- `required_fix`: `spec-required bug` or concrete `current production risk`.
- `optional_trim`: cleanup/simplification that deletes, narrows, inlines, or reuses.
- `evidence_note`: validation gap without a proven current bug.
- `defer`: out-of-scope, adjacent, or deferred surface.

Recommendation assignment:

- `required_fix`: Should fix yes; Fix now now; How = minimal direct fix.
- `optional_trim`: Should fix nice-to-have; Fix now now if same-slice deletion/narrowing is small, else next-round; How = minimal trim/reuse.
- `evidence_note`: Should fix nice-to-have or no; Fix now next-round/backlog; How = smallest proof or record gap.
- `defer`: Should fix no; Fix now backlog; How = defer/omit.

For non-required_fix: use existing proof, delete/narrow/inline/reuse, omit/defer/escalate.

Reviewer findings, `Minimal Fix`, and `Recommendation` are advisory, not patch authorization. New tests, schema/state/metadata, validation infrastructure, labels, or user-facing surfaces require `required_fix` or explicit user scope.

For evidence-only findings, judge whether proposed verification would materially widen the repo's validation surface. If yes, keep `near-blocking` unless concrete bug evidence exists or the heavier verification is already in established CI.

Renumber final finding IDs and assign final severity.

## Step 6: Return the verdict

Verdict values:

- `pass`: no verified blocking findings
- `fail`: verified blocking findings exist
- `needs_escalation`: evidence is insufficient, reviewer cardinality unmet, correct base unknowable, or human judgment required

Output format:

Compact output: verdict first with target/goals, blocking findings, then top 5 near-blocking unless exhaustive review requested. Include omitted near-blocking count/categories when capped. Near-blocking does not fail verdict; high-risk evidence gaps may still require `needs_escalation`.

```markdown
**Verdict:** `pass` / `fail` / `needs_escalation`
Target: <artifact/revision>
Goals: <selected goals>
Omitted near-blocking: <none | count/categories>

### F-001

Severity: `near-blocking` / `blocking`
Actionability: `required_fix` / `optional_trim` / `evidence_note` / `defer`
Recommendation:
- Should fix: `yes` / `nice-to-have` / `no`
- Fix now: `now` / `next-round` / `backlog`
- How: <one sentence describing minimal action>
Finding: <factual description>
Impact: <impact description>
Trigger: <trigger condition>
Anchor: <path:line | step/item/section>
Minimal Fix: <smallest safe change>
```
