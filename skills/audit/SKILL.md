---
name: audit
description: Structured review gate to surface blocking issues before merging or shipping.
reviewer_model: gpt
---

# Audit

`receiving-code-review` is only for human/external feedback.

## Hard Rules

- `reviewer_model: gpt` is binding. Confirm reviewer agent type = `gpt` before dispatch; if not, STOP.
- Dispatch `/fanout -m sample -a gpt:2` per selected goal; collect 2 independent GPT outputs with real `Anchor` per goal before GSA. Required total = `2 * selected goals`.
- No inline review or non-GPT substitution. Let `/fanout` retry launch failures; unmet cardinality => `needs_escalation`.
- Audit owner adjudicates during Step 5; DO NOT dispatch a separate adjudicator agent.
- No raw `/fanout` for review-shaped acceptance gates; audit owns verdicts, GSA, source verification, actionability.
- Do not pressure deferred/out-of-scope work. Only `required_fix` or explicit user scope justifies new guards, state, schema, validation infrastructure, broad tests, labels, or user-facing surfaces by default.

## Step 1: Load skill `/fanout`

Claude Code loads `/codex-exec` first. Codex CLI uses native gpt-5.5 xhigh.

## Step 2: Plan the review goals

### Case 1: Code

#### Goal 1: Spec and intent
Cover explicit requirements, acceptance criteria, called-out scenarios, preserved behavior/interfaces/data contracts/compatibility unless changed, root user goal not surface symptom, incomplete/conflicting/unknowable specs, and current risk only for deferred/out-of-scope surfaces.

#### Goal 2: Functional correctness
Within agreed behavior, check happy paths, edge cases, nulls, invalid inputs, state transitions, return values, runtime behavior, and side effects by condition/time/scope.
Broad code diffs: split reviewers across complementary angles: changed/removed behavior and lost invariants, caller/callee contract impact, language/framework pitfalls, wrapper/proxy/delegation correctness. Use relevant angles only; findings still need real anchors and source verification.

#### Goal 3: Validation and evidence
Check tests/assertions/logs/metrics/probes/manual checks across critical paths, boundaries, major failure modes. Conclusions need evidence. Heavy new browser/DOM harnesses, test frameworks, or validation infrastructure defaults to `near-blocking` plus lighter deterministic proof unless concrete bug evidence exists.

#### Goal 4 (optional): Risk and failure safety
Add when change touches permissions/trust boundaries, durable state/data writes, concurrency, retry, rollback, or idempotency.
Check partial failure, retry, rollback, interruption, stale state, races, ordering dependencies, idempotency gaps, cross-boundary interactions, permissions, trust boundaries, and dangerous capabilities.

#### Goal 5 (optional): Structure, simplification, and maintainability
Add for large refactors, new abstractions, high coupling, unclear ownership, or hot-path/perf-sensitive code.
Prefer delete/inline/narrow/reuse before layers. Reuse local helpers. Keep helpers only for real duplication, stable semantic boundaries, or materially better test seams. Flag unused returns/params, thin wrappers, dead generalization, no-consumer structures, excess helpers/wrappers/branches/aliases/state, unnecessary work, missed concurrency, hot-path bloat, pre-check-before-operate, missing cleanup, broad reads/loads, and wrong-depth special cases outside the owning mechanism. Skip stylistic, speculative, or scope-expanding suggestions.

### Other cases

For non-code reviews (plans/task lists), derive smallest goal set from provided material: completeness, boundaries, dependencies, acceptance, risk. Launcher supplies artifact/context. Anchor goals/findings/verification to provided material only; do not inspect unrelated workspace or adjacent files. If insufficient, return `needs_escalation` with missing context.

### Review capsule

Before dispatch define target/revision, stage (`milestone|final|ad-hoc`), goal, selected goals, non-goals, boundaries, base/candidate/diff range, missing context. Smallest set: simplification = Goal 1 + Goal 5; durable-state/permission/concurrency/retry/rollback risk adds Goal 4. Missing material context => `needs_escalation`.

Harness tasks: scope reviewers to `boundary.mutable` plus current round touched files; keep `boundary.immutable`, unrelated dirty files, workspace drift out of scope unless included. Include boundary in prompt.

Harness milestone/final gates include active task/milestone intent: `task.description`, `done_when`, task motivation, milestone objective/rationale/exit criteria, approach hypothesis/rationale, relevant non-goals/constraints, boundary, base/candidate/diff range, touched files. File scope limits inspection; it does not replace intent or acceptance context.

Git branches/worktrees include base, candidate, diff range. Harness base comes from task baseline or round base, not current `main`; if base advanced since setup, do not use `main..HEAD` unless explicit. Unknown base => `needs_escalation`; request before dispatch.

Custom goals state which standard goal they extend/replace. Reruns review diff since previous audit plus unresolved accepted findings; carry known-open IDs unless anchors changed. Consecutive harness reruns may reference the prior capsule when scope is unchanged; inline only current changed files, diff range, unresolved findings. Reopen full goals only for final release, scope change, or new cross-cutting risk.

## Step 3: Dispatch reviewers via `/fanout -m sample`

### Prompt for Reviewers

- Agent Context Card, e.g. `Agent: reviewer | depth 1/1 | budget 0 | parent: audit`
- Target/scope materials: diff, files, plan, or artifact; git targets include base revision, candidate revision, diff range.
- Goal name, angle, coverage target, standard.
- Code goals: name concrete complementary angles across the 2 reviewers when useful.
- Findings: non-speculative issues with real anchors only; skip stylistic; avoid scope expansion unless a verified issue requires it; deferred surfaces count only when artifact claims support or omission creates concrete current risk.
- Preserve explicit user wording/surfaces; if exact labels, output shape, fields, or "do not use X" appear, propose alternates only for verified bugs; prefer delete, hide, narrow, reuse.
- Validation gaps: smallest proof for current slice; new heavy infrastructure defaults to `near-blocking` unless concrete bug evidence exists.
- Minimal Fix: smallest safe change; if unsafe, recommend minimal refactor, bound scope, list verification. Goal 5 prefers delete/inline/narrow over layers.
- Severity: `blocking` = evidence of real bug or production risk; `near-blocking` = issue exists but lacks direct failure evidence.

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

Wait for all `/fanout` reviewers. Require Hard Rules quorum before synthesis/verification. If quorum remains unmet after `/fanout` retry exhausts, return `needs_escalation` with launch-failure reason. Final artifacts record quorum by goal: dispatched, received, missing. Default one full wave per stage; after fixes, rerun narrowly. >10 reviewers or second full wave requires risk reason + narrowed target.

## Step 5: Source-material verification and owner adjudication

Start GSA (Generative Self-Aggregation):
1. Deduplicate findings and merge same-root issues.
2. Read source at each anchor: code files/lines, or non-code steps/items/sections in provided material.
3. Discard unverified findings; repeats get extra scrutiny, not automatic acceptance.
4. Classify verified findings.
5. Assign actionability and recommendation.

Verified findings may still be out of scope. Collapse same deferred-surface mismatch and prefer trim/split/delete over implementing deferred work.

GSA is internal state, not a second deliverable. If recorded, keep one verification/action line per final finding beyond verdict card.

Classification -> actionability -> recommendation:

- `spec-required bug`: requirement/compat violation -> `required_fix`; yes / now / minimal direct fix.
- `current production risk`: concrete runtime/data/security/availability/user-visible failure -> `required_fix`; yes / now / minimal direct fix.
- `validation gap`: weak evidence, no proven bug -> `evidence_note`; nice-to-have/no / next-round/backlog / smallest proof or record gap.
- `cleanup/simplification`: acceptable behavior with excess breadth/duplication/complexity -> `optional_trim`; nice-to-have / now if small same-slice deletion/narrowing else next-round / minimal trim/reuse.
- `out-of-scope policy/feature`: deferred surface, new policy, or adjacent behavior -> `defer`; no / backlog / defer/omit.

Non-`required_fix`: use existing proof; delete/narrow/inline/reuse; omit/defer/escalate.

Reviewer findings, `Minimal Fix`, and `Recommendation` are advisory, not patch authorization. New tests, schema/state/metadata, validation infrastructure, labels, or user-facing surfaces require `required_fix` or explicit user scope. Evidence-only verification widening repo validation surface stays `near-blocking` unless concrete bug evidence exists or heavier verification is already in CI.

## Step 6: Return the verdict

Verdict values:

- `pass`: no verified blocking findings
- `fail`: verified blocking findings exist
- `needs_escalation`: evidence is insufficient, reviewer cardinality unmet, correct base unknowable, or human judgment required

Output format:

Compact output: verdict first with target/goals, blocking findings, then top 5 near-blocking unless exhaustive review requested. Include omitted near-blocking count/categories when capped. Near-blocking does not fail verdict; high-risk evidence gaps can require `needs_escalation`.

Storage with harness: one audit artifact owns finding details. Acceptance maps use `F-id -> classification -> actionability -> chosen_action -> evidence/commit`; do not restate Finding/Impact/Trigger/Anchor there. Manifest/state/context record verdict, quorum, finding IDs/counts, artifact path, base, candidate, diff range/stat, and final findings. No cumulative patch by default; keep latest pass/latest failure or final review only when needed.

```markdown
**Verdict:** `pass` / `fail` / `needs_escalation`
Target: <artifact/revision>
Stage: `milestone` / `final` / `ad-hoc`
Goals: <selected goals>
Reviewers: <goal=count/required, ...; total received/required>
Omitted near-blocking: <none | count/categories>

### F-001

Severity: `near-blocking` / `blocking`
Actionability: `required_fix` / `optional_trim` / `evidence_note` / `defer`
How: <one sentence describing minimal action>
Finding: <factual description>
Impact: <impact description>
Trigger: <trigger condition>
Anchor: <path:line | step/item/section>
Minimal Fix: <smallest safe change>
```
