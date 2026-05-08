---
name: critique
description: Use when the user requests a structured review or needs to surface blocking issues.
---

# Critique

Critique is a self-contained agentic review workflow. It dispatches reviewers, verifies findings against source, and returns a verdict. Do not route critique findings through `receiving-code-review`; that skill is for human-supplied or externally authored feedback only.

## Step 1: Load skill `/fanout`

## Step 2: Plan the review goals

### Case 1: Code

#### Goal 1: Spec and intent
- Are all explicit requirements, acceptance criteria, and called-out scenarios covered?
- Are existing behavior, interfaces, data contracts, and compatibility that were not requested to change preserved, without introducing unrequested scope expansion or maintenance cost?
- Does the implementation address the real problem, root cause, or user goal rather than only fixing a surface symptom or adjacent issue?
- If the spec is incomplete, conflicting, or impossible to determine from the source material, are the gaps called out explicitly?
- If some surface is explicitly deferred or out of scope, are findings limited to current risks instead of pressuring the artifact to implement the deferred surface?

#### Goal 2: Functional correctness
- Within the agreed target behavior, are the happy path, edge cases, null values, and invalid inputs handled correctly?
- Are state transitions, return values, and runtime behavior correct?
- Do allowed side effects happen only under the right conditions, at the right time, and within the right scope?

#### Goal 3: Validation and evidence
- Are tests, assertions, logs, metrics, probes, or manual verification steps sufficient to support the key conclusions?
- Does the validation cover critical paths, key boundaries, and major failure modes?
- Are review conclusions based on provided evidence rather than "the code looks right"?
- If closing an evidence gap would require introducing new heavy browser/DOM harnesses, test frameworks, or other validation infrastructure that the target repo does not already use, prefer reporting a `near-blocking` gap and recommending a lighter deterministic probe first.

#### Goal 4 (optional): Risk and failure safety
Add this goal when the change touches permissions/trust boundaries, durable state/data writes, or concurrency/retry/rollback/idempotency.
- After partial failure, retry, rollback, or interruption recovery, could the change leave behind bad state, data corruption, or dangerous side effects?
- Could stale state, race conditions, ordering dependencies, idempotency gaps, or cross-boundary interactions cause incorrect behavior?
- Are permissions, trust boundaries, and dangerous capabilities constrained correctly?

#### Goal 5 (optional): Structure, simplification, and maintainability (reuse / simplicity / efficiency)
Add this goal when the change introduces a large refactor or new abstractions, high coupling/unclear ownership boundaries, or hot-path/perf-sensitive code.
- Default bias: smallest structure that preserves clarity, correctness, and testability.
- Reuse existing local helpers instead of duplicates or hand-rolled string/path/env/type checks.
- Prefer fewer helpers, wrappers, branches, return values, schema aliases, and state; remove unused returns/params, thin wrappers, dead generalization, and structures with no current consumer.
- Keep a helper only when it removes real duplication, captures a stable semantic boundary, or creates a materially better test seam.
- Remove unnecessary work, missed concurrency, hot-path bloat, pre-check-before-operate, missing cleanup, and overly broad reads or loads.

For structure or simplification findings, prefer the smallest safe change with a minimal diff. Prefer deleting, inlining, or narrowing over adding another layer. Skip purely stylistic, speculative, or scope-expanding suggestions.

#### Additional optional goals: compatibility and migration, concurrency and timing, data and state consistency, observability and operability, performance and resource usage, and similar angles

### Other cases
For non-code reviews such as plan documents or task lists, generate the smallest sufficient set of review goals from the material, prioritizing completeness, boundaries, dependencies, acceptance, and risk.
The critique launcher must provide the review artifact and enough context to judge it. For non-code review, anchor goals, findings, and verification to the provided material only. Do not inspect unrelated workspace state or adjacent files to invent findings. If the provided material is insufficient to judge, return `needs_escalation` and name the missing context.

### Review capsule
Before dispatch, define the target artifact/revision, original goal, selected goals, and known non-goals or scope boundaries when provided or material. Choose the smallest goal set needed for the asked decision: targeted simplification usually uses Goal 1 + Goal 5; durable-state, permission, concurrency, retry, or rollback risk adds Goal 4. If missing context materially affects judgment, return `needs_escalation` instead of broadening scope.

When the review target is a harness task, scope reviewers to `boundary.mutable` plus the current round's touched files by default. Treat `boundary.immutable`, unrelated dirty files, and pre-existing workspace drift as out of scope unless the review request explicitly includes them. Include this scope boundary in the reviewer prompt so unrelated dirty-tree state does not become a finding.

When the target is a git branch or worktree, include the exact review base and candidate revision in the review capsule. For harness tasks, derive the base from the task baseline or recorded round base, not from the current `main` by default. If `main` or the base branch has advanced since task setup, do not use `main..HEAD` unless that is explicitly the intended review range. If the correct base cannot be determined, return `needs_escalation` and request the missing base instead of dispatching reviewers against a guessed range.

## Step 3: Dispatch reviewers via `/fanout -m sample`

Review is Thinker work. Default reviewer agent type is `gpt`. Refer to `using-agents`, use gpt-5.5 xhigh (for Codex CLI) or load `/codex-exec` (for Claude Code) first.
For each goal, use `/fanout -m sample -a gpt:2` to dispatch 2 independent reviewers with the same prompt. Override the agent type only if the user explicitly requests a different reviewer type. Total reviewers required: 2 * number of goals. Inline review is not allowed.
If reviewer dispatch is blocked by execution-surface unavailability, policy, or another launch failure, let `/fanout` apply its own retry policy first. Return `needs_escalation` only if `/fanout` exhausts retry and reviewer-pair cardinality is still unmet. Do not silently substitute another reviewer type or inline review.

### Prompt for Reviewers

- Agent Context Card, such as `Agent: reviewer | depth 1/1 | budget 0 | parent: critique`
- Reviewer agent type: `gpt` by default for critique unless the user explicitly overrides it
- Review target and scope materials, such as a diff, files, a plan, or another artifact. For git review targets, include the base revision, candidate revision, and diff range reviewers must use.
- Goal name, review angle, coverage target, and review standard
- Finding format and constraints: report only non-speculative issues with a real anchor; skip purely stylistic or speculative suggestions; avoid scope expansion unless required to fix a verified issue; if the target materials explicitly defer a surface, do not treat lack of support itself as a finding unless the artifact claims support or omission creates a concrete current risk
- Preserve explicit user wording and surfaces. If context gives exact labels, output shape, fields, or "do not use X", do not propose alternate labels/fields/surfaces unless required by a verified bug; prefer delete, hide, narrow, or reuse.
- For validation-gap findings, prefer the smallest proof needed for the current slice. If stronger evidence would require adding new heavy verification infrastructure not already present in the target repo, default to `near-blocking` unless there is already concrete bug evidence.
- Minimal Fix: prioritize correctness and artifact quality. Start with the smallest safe change (add tests/guards when required). If a small patch cannot fix the issue safely, recommend the minimal necessary refactor and state why; bound scope and list verification steps. For Goal 5 findings, prefer minimal diffs and deleting/inlining/narrowing over adding layers
- Severity: `near-blocking` = the issue exists but lacks direct failure evidence; `blocking` = there is evidence of a real bug or production risk

```markdown
### F-001

Severity: `near-blocking` / `blocking`
Finding: <factual description>
Impact: <impact description>
Trigger: <trigger condition>
Anchor: <path:line | step/item/section> (required; must be the real triggering location)
Minimal Fix: <describe the smallest safe change>
```

- `**Verdict:**` and `F-xxx` cards are valid only after reviewer dispatch and source verification. If `/fanout` cannot satisfy reviewer cardinality, return `needs_escalation`; do not emit critique-shaped inline review.

## Step 4: Collect reviewer outputs

Wait for all `/fanout` reviewers to return. Ensure you have 2 independent reviewer outputs per goal, each with a real `Anchor`. Do not start synthesis or verification until this is true.
If reviewer-pair cardinality is still unmet after `/fanout` exhausts retry, stop and return `needs_escalation` with the concrete launch-failure reason.

## Step 5: Source-material verification

Start GSA (Generative Self-Aggregation) to merge duplicate issues and mark them.
Verify each anchor and finding against the source material. For code review, go back to files and lines; for other review types, go back to steps, items, or sections from the provided review material. Unverified findings do not enter the final result. Findings raised multiple times deserve extra scrutiny.
After source verification, do owner-side adjudication inside `/critique` itself: a verified finding may still be out of current scope. When several findings reduce to one deferred-surface mismatch, collapse them and prefer trim/split/delete over implementing the deferred work.
Classify each verified finding before fixes:

- `spec-required bug`: requirement or compatibility violation.
- `current production risk`: concrete current runtime, data, security, availability, or user-visible failure.
- `validation gap`: weak evidence without a proven bug.
- `cleanup/simplification`: acceptable behavior with excess breadth, duplication, or complexity.
- `out-of-scope policy/feature`: deferred surface, new policy, or adjacent behavior.

Project each finding into implementation actionability after verification; individual reviewers do not self-authorize it:

- `required_fix`: `spec-required bug` or concrete `current production risk`.
- `optional_trim`: cleanup/simplification that deletes, narrows, inlines, or reuses.
- `evidence_note`: validation gap without a proven current bug.
- `defer`: out-of-scope, adjacent, or deferred surface.

Only `required_fix` justifies new guards, state, schema, validation infra, or broad tests by default. For the rest: use smallest existing proof, delete/narrow/inline/reuse, or omit/defer/escalate.
Reviewer findings and `Minimal Fix` text are advisory, not patch authorization. New tests, schema/state/metadata, validation infrastructure, labels, or user-facing surfaces require `required_fix` or explicit user scope; otherwise prefer delete, narrow, inline, reuse, omit, or defer.
For evidence-only findings, also judge whether the proposed verification would materially widen the target repo's validation surface. If yes, keep the finding `near-blocking` unless there is already concrete bug evidence or the heavier verification is part of the repo's established CI path.
Renumber finding IDs and assign the final severity level.

## Step 6: Return the verdict

Verdict values:

- `pass`: no verified blocking findings
- `fail`: verified blocking findings exist
- `needs_escalation`: evidence is insufficient or human judgment is required

Output format

Keep output compact: verdict first with target and selected goals, then verified blocking findings, then at most the top 5 near-blocking findings unless the user asks for exhaustive review. Include omitted near-blocking count/categories when capped. Near-blocking findings do not fail the verdict; high-risk evidence gaps may still require `needs_escalation`.

```markdown
**Verdict:** `pass` / `fail` / `needs_escalation`
Target: <artifact/revision>
Goals: <selected goals>
Omitted near-blocking: <none | count/categories>

### F-001

Severity: `near-blocking` / `blocking`
Actionability: `required_fix` / `optional_trim` / `evidence_note` / `defer`
Finding: <factual description>
Impact: <impact description>
Trigger: <trigger condition>
Anchor: <path:line | step/item/section>
Minimal Fix: <smallest safe change>
```
