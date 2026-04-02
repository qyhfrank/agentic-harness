# Discovery Protocol

Maintenance rules for `.harness/tasks/<task_id>/discovery.md` — the durable knowledge layer between `context.md` (hot, curated each round) and `state.jsonl` (cold, append-only facts).

## Purpose

Working Memory in `context.md` is capped at 15 items and aggressively pruned. Knowledge that is valuable across 3+ rounds or across sessions — dead ends, tool quirks, environment constraints, codebase structure — gets compressed or lost, leading to re-exploration and wasted rounds.

`discovery.md` holds **curated current truth**: cross-round reusable knowledge with scope, evidence, and staleness rules. It is not a log, not a backlog, and not a second Decisions section.

See the Division of Labor table in `context-protocol.md` for how `discovery.md` fits alongside `state.jsonl` and `context.md`.

## File Structure

```markdown
# Task Discoveries

## Snapshot
- active: 6 | needs_recheck: 0 | contested: 0
- last_hygiene: round 14
- read_first: dead-002, ctr-001, env-003

## Active Index
- env-003 | environment | active | verified | r7 | npm ci broken on node 18 lockfile
- map-001 | codebase_map | active | stable | r4 | auth timeout crosses middleware and retry wrapper
- dead-002 | dead_end | active | verified | r7 | parser reorder only masks import bug
- ctr-001 | constraint | active | verified | r5 | public API must preserve v1 response shape
- tool-001 | tool_behavior | active | observed | r3 | npm test --bail exits 0 on failure in CI
- ver-003 | verification | active | verified | r6 | lint catches generated-file drift; unit tests do not

## Environment Facts

### env-003 npm ci broken on node 18 lockfile
- status: active
- confidence: verified
- claim: `npm ci` silently produces a broken install on this lockfile when run with Node 18.19.x + npm 9.x on macOS arm64.
- scope: macOS arm64, Node 18.19.x, npm 9.x
- impact: Must use `npm install` instead; CI image pinned to Node 18 triggers this.
- evidence: artifacts/round-5/npm-ci.txt, artifacts/round-7/repro.txt
- discovered: round 5
- last_verified: round 7
- revisit_when: package-lock.json changes, Node or npm version changes, CI image updates
- tags: npm, node, ci

## Dead Ends

### dead-002 parser reorder only masks import bug
- status: active
- confidence: verified
- claim: Reordering parser passes hides the import resolution bug but does not fix it.
- why_failed: Parser reorder changes evaluation order, masking the bug for the common case. Root cause is in the import resolver.
- scope: src/parser/**, src/resolver/**
- impact: Do not retry parser reorder approach. Fix must target resolver directly.
- evidence: artifacts/round-5/parser-reorder.patch, artifacts/round-7/regression.txt
- discovered: round 5
- last_verified: round 7
- revisit_when: resolver rewritten or import model changes
- tags: parser, bundler, resolver

## Codebase Map
## Tool & API Behavior
## Verification Insights
## Constraints

## Archived
- dead-001 | superseded by dead-002 (round 7) | tried import-map polyfill
- env-001 | deprecated (round 5) | npm ci workaround with --legacy-peer-deps
```

The example above is abbreviated. See Entry Schema for all required fields.

### Active Index Format

One line per active entry: `id | category | status | confidence | last_verified_round | one-line claim`. Tags are omitted from the index to save space; use `tags` fields in full entries for search.

### Archived Tombstone Format

One line per archived entry: `id | reason (round) | original one-line claim`. Reason is one of: `superseded by <id>`, `deprecated`, `incorrect`.

## Entry Schema

Every discovery entry must have these required fields:

| Field | Description |
|---|---|
| `status` | `active` / `needs_recheck` / `contested` / `deprecated` / `incorrect` / `archived` |
| `confidence` | `observed` / `verified` / `stable` |
| `claim` | One-sentence statement of the discovered fact |
| `scope` | Where this applies: file paths, platforms, versions, conditions |
| `impact` | How this affects future proposals, verification, or evaluation |
| `evidence` | Artifact paths, round references, or code locations |
| `discovered` | Round number when first recorded |
| `last_verified` | Round number when last confirmed still true |
| `revisit_when` | Conditions that would make this entry stale |
| `tags` | Searchability keywords |

Dead Ends additionally require `why_failed`: the specific reason the approach did not work.

## Status and Confidence

Status and confidence are orthogonal and managed separately.

### Status Lifecycle

`active` → `needs_recheck` (revisit_when conditions changed) → `active` (re-confirmed), `deprecated` (conditions changed), or `incorrect` (claim was wrong).

`active` → `contested` (conflicting evidence, unresolved) → `active` (resolved) or `incorrect`.

`deprecated` or `incorrect` → `archived` (after 2 clean rounds).

An entry at `needs_recheck` may be immediately returned to `active` if the agent re-confirms it before the 3-round timeout.

### Confidence Promotion

| Level | Requirement |
|---|---|
| `observed` | At least 1 artifact or anchored source-code evidence |
| `verified` | Reproduced once more, or second independent evidence source |
| `stable` | Survived 1 session recovery still valid, or unchallenged for 3+ rounds under same `revisit_when` conditions |

## Admission Gate

Not every observation belongs in `discovery.md`. An entry must pass this test:

1. **Durability**: Will this matter in 3+ rounds or across a session boundary?
2. **Re-exploration cost**: If forgotten, would the agent spend 1+ rounds rediscovering it?
3. **Specificity**: Can I state it as a scoped claim with evidence, not a vague impression?

If any answer is no, keep it in Working Memory. Pure speculation and single-round tactical notes stay in `context.md`.

## Write Discipline

### When to Write

Discovery entries are written **only during the RECORD step** of the round lifecycle. Other steps produce evidence (artifacts, verification output) but do not modify `discovery.md`.

### What Triggers a Write

| Trigger | Category | Confidence |
|---|---|---|
| Round result is `discard`, `crash`, or doom-loop pivot | Dead End | `observed` |
| Verification reveals tool/API behavior differs from expectation | Tool & API Behavior | `observed` |
| Code exploration reveals non-obvious module ownership or interaction | Codebase Map | `observed` |
| Environment difference causes CI-only or platform-only failure | Environment Fact | `observed` |
| Gate behavior pattern emerges across 2+ rounds | Verification Insight | `observed` |
| Implicit contract or invariant discovered that constrains proposals | Constraint | `observed` |

### What Does NOT Trigger a Write

- Tactical observations for the current round only (stays in Working Memory)
- Decisions about approach (goes to Decisions section in context.md)
- Metric values and round outcomes (goes to state.jsonl)
- Protocol-level feedback about harness itself (goes to feedback-note.json)

## Curation Rules

### Per-Round Light Hygiene (during RECORD)

After writing any new entries:

1. Update the Snapshot (counts, last_hygiene round, read_first list).
2. Update the Active Index to reflect new or changed entries.
3. If a new entry contradicts an existing one, apply the Conflict Resolution rules below.
4. If Working Memory contains items that were just promoted to discovery, replace with a reference: `See dead-002` instead of repeating the full claim.
5. If a discovery entry is archived and Working Memory still references it, remove the reference.

### Periodic Full Hygiene (every 5 rounds, session end, session resume)

1. Check every active entry's `revisit_when` conditions against current state.
2. Entries whose conditions changed: mark `needs_recheck`.
3. Entries marked `needs_recheck` for 3+ rounds without recheck: evaluate — still valid → `active`, conditions changed → `deprecated`, claim wrong → `incorrect`.
4. Move entries that have been `deprecated` or `incorrect` for 2+ clean rounds to Archived.
5. If active count exceeds soft cap, archive lowest-impact entries.
6. Update Snapshot read_first to the 3-5 entries most relevant to current objective.

### Conflict Resolution

Apply in order:

1. Check `revisit_when` and `scope` — are they the same?
2. Same scope, new evidence stronger: old entry → `incorrect`, new entry → `active`.
3. Different scope but both valid: narrow each entry's scope, both stay `active`.
4. `revisit_when` conditions changed: old entry → `deprecated` (expired, not wrong).
5. Cannot resolve in one round: old entry → `contested`, new observation stays in Working Memory. Add disambiguation to Next Steps.

### Wrong Facts

When an entry is discovered to be incorrect:

1. Mark status `incorrect`. Do not delete or silently edit the claim.
2. Append a new entry (or update an existing one) with the corrected understanding.
3. If any Decisions referenced the incorrect or deprecated entry, append a new Decision noting the correction.
4. After 2 clean rounds, move the incorrect entry to Archived with its original ID.
5. Never reuse IDs.

## Recovery Integration

Recovery protocol step 4a in `context-protocol.md` defines how to read `discovery.md` during recovery. The Recovery Smoke Test in `context-protocol.md` defines the extended 10-question test when `discovery.md` exists.

If `discovery.md` is corrupted or unreadable, fall back to `context.md`-only recovery and recreate `discovery.md` from Working Memory and recent artifacts as knowledge is rediscovered.

## context.md Integration

### Current State

Add one field when `discovery.md` has active entries:

```markdown
- active_discoveries: 6 active / 0 needs_recheck
```

This is a signal that `discovery.md` exists and has entries. The Snapshot inside `discovery.md` has the full details including `read_first`.

### Working Memory

Semantics are defined in `context-protocol.md` Working Memory section. Key rule: when an observation graduates to `discovery.md`, replace the bullet with a one-line reference (`→ See env-003`). Do not duplicate discovery entry content in Working Memory.

### Decisions

Decisions may reference discovery IDs for rationale: `Round 8: abandoned async batch approach per dead-002`.

## ID Format

```
<category_prefix>-<sequence_number>
```

| Category | Prefix |
|---|---|
| Environment Facts | `env` |
| Codebase Map | `map` |
| Dead Ends | `dead` |
| Tool & API Behavior | `tool` |
| Verification Insights | `ver` |
| Constraints | `ctr` |

Sequence numbers are task-scoped and never reused, even after archival.

## Caps

| Metric | Soft limit | Hard limit |
|---|---|---|
| Active entries | 15 | 20 |
| Archived tombstones | 10 | 15 |
| read_first entries | 3 | 5 |

Active Index always lists all active entries (one line per entry). When active entries hit the hard cap, archive lowest-impact entries before adding new ones.

## Backward Compatibility

- `discovery.md` is optional. Tasks without it use the existing `context.md`-only recovery path.
- No changes to `state.jsonl` schema or `config.yaml` schema.
- Existing tasks can adopt `discovery.md` at any round — create it from the scaffold template, promote relevant Working Memory items, and add `active_discoveries` to `context.md` Current State.
- To stop using `discovery.md`: delete the file (or rename to `discovery.md.bak`) and remove `active_discoveries` from `context.md`. Do not leave a stale file on disk — recovery step 4a reads it if it exists.

## Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| Active count > hard cap | Insufficient pruning | Force archive lowest-impact entries |
| Entry claim stale but status still `active` | Missed hygiene cycle | Run full hygiene, check `revisit_when` |
| Working Memory duplicates discovery content | Failed promotion discipline | Replace duplicates with ID references |
| Recovery cannot answer questions 7-10 | Snapshot stale or read_first wrong | Rebuild Snapshot from active entries |
| Same knowledge in discovery.md and Decisions | Boundary confusion | Discovery = reusable fact, Decision = committed choice. Deduplicate. |
| Incorrect entry blocks valid approach | Missing `revisit_when` check | Mark `needs_recheck` when conditions change, do not trust blindly |
| discovery.md corrupted or unreadable | File-level failure | Fall back to context.md-only recovery; recreate as discoveries emerge |
