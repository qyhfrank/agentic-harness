# Plan Protocol

Phase A: interactive config finalization. Guide the user from draft `config.yaml` to a finalized, executable contract.

## Entry Conditions

- The current task config exists with `draft: true`
- User is present for interactive Q and A

## Procedure

### 1. Load Current State

Resolve the current task, then read its `config.yaml` and `context.md`. Identify which fields are already filled vs empty.

### 2. Task Definition

If the user's goal is broad or ambiguous, invoke the `brainstorming` skill to explore intent, requirements, and scope before filling fields. Skip for well-defined tasks.

If the current task description still looks like generic setup language (for example `set up a harness`), treat that as missing goal state and ask for the real task objective before proceeding.

If the current task description is present but still too broad to define boundaries or metrics (for example `optimize this repo`), ask one narrowing question before proceeding.

Confirm or refine these fields through conversation:

```yaml
task:
  id: "<task_id>"   # immutable after scaffold — do not rename
  name: ""
  description: ""
  source: ""
```

`task.id` is set during scaffold and must not be changed during plan. It is used as the directory name, branch name, worktree name, ledger path, and artifact path. If the user wants a different ID, re-scaffold a new task instead.

If the user provided a goal during scaffold, `description` is pre-filled. Confirm it captures the intent.

### 3. Boundary Definition

This is the most critical step. Guide the user to define:

```yaml
boundary:
  mutable: []
  immutable: []
```

Approach:
1. Start from the goal. Ask: "Which files or directories need to change to achieve this?"
2. Suggest candidates based on repo structure and goal.
3. Ask: "Is there anything that must NOT be touched?"
4. Validate: every mutable path exists on disk. Warn on overly broad patterns.

If the user is unsure, suggest starting narrow and note that boundaries can be widened later.

### 4. Verification Strategy

Help the user choose verification gates:

```yaml
verification:
  mandatory:
    - type: command
      run: ""
  guard:
    - run: ""
  escalation:
    - condition: ""
      action: ""
```

Discovery flow:
1. Check what test/lint/build commands exist from scaffold's repo assessment.
2. Suggest mandatory gate = primary test command.
3. Suggest guard gates = lint, typecheck, format check.
4. Ask if `agent_review` or `human_review` is needed.
5. Dry-run: execute each proposed command once. Report results.

If dry-run fails, the command is wrong or the repo has pre-existing failures. Resolve before finalizing.

Keep the config linear. Prefer an ordered sequence of deterministic checks, optional review stages, and simple escalation conditions. Do not invent branching gate graphs or nested policy DSLs in config.

### 5. Evaluation Strategy

Help the user define what counts as progress and completion:

```yaml
evaluation:
  objective: satisfy | optimize
  acceptance_criteria: []
  metric:
    name: ""
    direction: increase | decrease | none
    volatile: false
    samples: 3
    min_delta: 0
    confirmation_runs: 2
  close_authority:
    type: command_backed | human_review | confirmed_review
```

Guide:
- `satisfy`: ask "What specific criteria mean the task is done?" -> fill `acceptance_criteria`
- `optimize`: ask "What metric are you optimizing and which direction is better?"
- If there is no meaningful numeric metric, set `direction: none`
- `close_authority` answers who may declare task completion

Explain explicitly: verifier outputs are inputs; evaluator rules decide keep, no-op, discard, and completion.

### 6. Stop Guards

```yaml
termination:
  stop_guards:
    budget: { max_rounds: 50 }
    stagnation: { rounds: 5 }
```

Guide:
- Budget: suggest 50 rounds default, adjust based on task scope; use `max_rounds: -1` only when the user explicitly wants an open-ended run
- Stagnation: suggest 5 rounds default

### 7. Entropy Configuration

Review defaults, adjust only if user has specific needs:

```yaml
entropy:
  doom_loop_threshold: 3
  doom_loop_action: reread_and_pivot
```

Explain briefly: "If the same error appears 3 times, the agent will re-read all files and try a different approach. Want to adjust this?"

### 8. Finalize

Before removing `draft: true`:

1. **Vocabulary check**: verify all config values use defined vocabulary:
   - `evaluation.objective` must be `satisfy` or `optimize`
   - `evaluation.metric.direction` must be `increase`, `decrease`, or `none`
   - `rollback.strategy` must be `revert_commit` or `reset_to_last_pass`
2. Show the complete config to the user for review.
3. Run mandatory gate dry-run one final time.
4. Confirm: "This config looks ready. Shall I finalize it?"

On confirmation:
- Remove the `draft: true` line from config.yaml.
- Update context.md: set phase to "plan (complete)", record key decisions.
- Report: "Config finalized. Run `/harness run` to start."

## Cross-Session Continuity

Plan mode may span multiple sessions. On re-entry:
1. Read context.md to see what was already discussed.
2. Read config.yaml to see which fields are filled.
3. Resume from the first unfilled section, not from the beginning.

## Interaction Style

- Ask one question at a time. Do not dump all fields at once.
- Suggest concrete values based on repo analysis. Let the user confirm or adjust.
- If the user says "just use defaults", fill reasonable defaults and show for confirmation.
- If the user provides a spec or plan file, extract answers from it before asking.

## Feedback Note

After finalizing config, write a structured feedback note per `feedback-protocol.md`.

Auto-detect: compare finalized config against the scaffold draft. Any field where the user overrode the draft value generates a `bad_default` or `protocol_gap` event (source: `human`).
