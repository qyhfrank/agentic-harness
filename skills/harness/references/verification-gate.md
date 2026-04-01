# Verification Gate Protocol

Verification answers whether a candidate change may proceed to evaluation.

## Gate Types

| Type | Executor | Deterministic | Cost | Trust |
| --- | --- | --- | --- | --- |
| `command` | script / test / lint / build | Yes | Low | Medium |
| `agent_review` | LLM reviewer or review skill | No | Medium | Low-Medium |
| `human_review` | Human | N/A | High | Highest |

## Verification Tiers

Gates serve different purposes at different points in the iteration cycle. Classify each gate by tier to determine how often it runs:

| Tier | Question | Frequency | Examples |
| --- | --- | --- | --- |
| 0 | Did I break anything? | `every_round` | typecheck, lint, unit tests |
| 1 | Did this specific change work? | `every_round` | isolated benchmark, profiling probe, smoke metric, targeted repro script |
| 2 | Does the whole system work? | `milestone` or `final` | full e2e suite, integration tests, manual QA |

Tier 0 and 1 form the **iteration loop** -- fast enough to run every round without slowing the propose-verify cycle. Tier 2 is the **validation gate** -- expensive, run at milestones or only at completion.

**Tier 1 is the key insight.** Most workflows default to either unit tests (Tier 0) or full e2e (Tier 2), skipping the middle. A Tier 1 gate is a **task-specific isolated probe** that answers "did this optimization actually help?" without running the full pipeline. Examples:

- A micro-benchmark that exercises the specific function being optimized (compare latency before/after)
- A profiling script that checks whether the targeted hot path improved
- A smoke metric check (bundle size, cold start time, memory peak) against a recorded baseline

When `evaluation.objective: optimize`, a Tier 1 probe is strongly recommended. Without one, every iteration pays the full Tier 2 cost to learn whether the change had any effect.

### Gate Frequency

Each gate in `verification.mandatory` and `verification.guard` supports a `frequency` field:

| Frequency | When it runs |
| --- | --- |
| `every_round` | Every round (default if omitted) |
| `milestone` | Every N rounds (`termination.milestone_interval`, default 10), plus always on the final round |
| `final` | Only during the completion verification pass |

When a gate is skipped due to frequency, it is not recorded as pass or fail. The round artifacts note which gates were skipped and why.

## Verdict Schema

Every gate produces exactly one verdict:

```yaml
verdict:
  status: pass | fail | needs_escalation
  evidence: "test output / review comment / ..."
  retryable: true | false
  anchors: []
```

Verification verdicts are inputs to evaluation. They do not directly close tasks.

## `agent_review` Hard Rules

1. Single reviewer cannot close a task. A single agent produces `needs_escalation` or a candidate opinion, never a final completion decision.
2. Evidence must contain mechanically checkable anchors (file:line, test name, behavior prediction). No anchors -> force `needs_escalation`.
3. Command backstop: when the repo has runnable deterministic checks, `agent_review` cannot be the sole mandatory gate.
4. Discovery is not closure: a review provider such as `/critique` may surface blockers, but the configured evaluator still decides round disposition and task completion.

## Rollback by Gate Type

| Gate | On `fail` |
| --- | --- |
| `command` | Signal failure to the evaluator. Default action is discard. |
| `agent_review` | Hold for escalation or confirmation. No direct rollback on unconfirmed discovery findings. |
| `human_review` | Follow the human decision. |

## Discovery / Confirmation Pipeline

Five stages for `agent_review` findings:

1. **Discovery.** Fan-out: multiple reviewers explore independently.
2. **Evidence gate.** Finding has file:line / test name / concrete prediction? Yes -> stage 3. No -> `needs_escalation`.
3. **Confirmation.** Strip original reviewer reasoning. Give finding + code to a fresh agent. Fresh agent independently verifies.
4. **Oracle lifting.** Can the finding become a failing test or repro script? Yes -> migrate to `command` domain permanently. No -> probabilistic verdict, requires quorum.
5. **Coverage certificate.** Record what was checked. "No findings" is only meaningful when accompanied by coverage evidence.

`/critique` is a good fit for discovery and structured review reporting. It is not, by itself, a completion oracle.

## Reviewer Context Contract

When running `agent_review`, pass only the minimum context needed for a reliable review:

Required inputs:
- candidate diff or changed files
- relevant surrounding files in `boundary.mutable`
- `AGENTS.md` when repository conventions matter
- the current task's verification and evaluation excerpts when the reviewer must judge against explicit acceptance rules

Do not pass as authority:
- proposer chain-of-thought or rationale dump
- earlier reviewer conclusions unless the task is explicit confirmation of a single anchored finding
- unrelated task history from other task directories

Default reviewer stance:
- discovery reviewers search for anchored findings
- confirmation reviewers verify one anchored finding at a time with fresh context
- no reviewer may close the task alone

This keeps `agent_review` useful without letting the config grow into a general reviewer-orchestration language.

## Volatile Metrics Protocol

Activate when a metric is declared volatile in the task's `evaluation.metric` config.

1. **Multiple samples.** Run N times per round (default `N=3`), take median.
2. **Minimum delta.** Improvement below `min_delta` threshold -> recommend `no_op` to the evaluator.
3. **Confirmation rounds.** Require consecutive rounds of same-direction improvement before trusting the gain.
4. **Environment lock.** Record baseline environment fingerprint (OS, runtime versions, hardware). Detect drift between rounds; flag if fingerprint changes.

### Statistical Confidence (MAD)

After 3+ rounds of data, compute confidence for each improvement using MAD (Median Absolute Deviation) as a robust noise estimator:

```
noise_estimate = MAD = median(|each_metric - median(all_metrics)|) * 1.4826
confidence = |current_improvement| / noise_estimate
```

Confidence interpretation (advisory, consumed by the evaluator):

| Confidence | Signal | Guidance |
|---|---|---|
| >= 2.0x | Strong | Improvement is well above noise floor. Trust it. |
| 1.0x - 2.0x | Marginal | Improvement is near noise boundary. Prefer confirmation runs. |
| < 1.0x | Noise | Treat as `no_op` unless another completion rule dominates. |

Record confidence in the round event under `metric.confidence`.

## Composite Gate Execution Order

1. Filter gates by frequency: skip any gate whose `frequency` does not match the current round (see Verification Tiers > Gate Frequency).
2. Run all eligible `command` gates first (cheapest, deterministic).
3. If all commands pass, run eligible `agent_review` gates.
4. If agent_review passes or escalates, queue `human_review` if configured.
5. **Short-circuit:** first mandatory `fail` from a deterministic gate -> skip remaining mandatory gates and hand failure to the evaluator.
