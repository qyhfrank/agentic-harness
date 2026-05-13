---
name: brainstorming
description: "You MUST use this before any creative work - creating features, building components, adding functionality, or modifying behavior. Explores user intent, requirements and design before implementation."
---

# Brainstorming Ideas Into Designs

Help turn ideas into fully formed designs and specs through natural collaborative dialogue.

<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project, or take any implementation action until you have presented a design and the user has approved it. This applies to EVERY project regardless of perceived simplicity.
Even one-function utilities and config changes go through a short design.
</HARD-GATE>

## Checklist

You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `.context/plans/YYYY-MM-DD-<topic>.md`
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to next step** — offer the user a choice: hand off to `/harness` for adaptive execution, or stop here (user decides next step independently)

## The Process

**Understanding the idea:**

- Check out the current project state first (files, docs, recent commits)
- Before asking detailed questions, assess scope: if the request describes multiple independent subsystems (e.g., "build a platform with chat, file storage, billing, and analytics"), flag this immediately. Don't spend questions refining details of a project that needs to be decomposed first.
- If the project is too large for a single spec, help the user decompose into sub-projects: what are the independent pieces, how do they relate, what order should they be built? Then brainstorm the first sub-project through the normal design flow. Each sub-project gets its own spec → plan → implementation cycle.
- For appropriately-scoped projects, ask questions one at a time to refine the idea
- Prefer multiple choice questions when possible, but open-ended is fine too
- Only one question per message - if a topic needs more exploration, break it into multiple questions
- Focus on understanding: purpose, constraints, success criteria

**Exploring approaches:**

- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why

**Presenting the design:**

- Once you believe you understand what you're building, present the design
- Scale each section to its complexity: a few sentences if straightforward, up to 200-300 words if nuanced
- Ask after each section whether it looks right so far
- Cover: architecture, components, data flow, error handling, testing
- Be ready to go back and clarify if something doesn't make sense

**Design for isolation and clarity:**

- Break work into small units with one purpose, clear interfaces, explicit dependencies, and independent tests. If consumers must read internals or edits require broad context, improve boundaries inside the design scope.

**Working in existing codebases:**

- Explore the current structure before proposing changes. Follow existing patterns.
- Where existing code has problems that affect the work (e.g., a file that's grown too large, unclear boundaries, tangled responsibilities), include targeted improvements as part of the design - the way a good developer improves code they're working in.
- Don't propose unrelated refactoring. Stay focused on what serves the current goal.

## After the Design

Write the validated spec to `.context/plans/YYYY-MM-DD-<topic>.md` unless user preferences override. Ensure `.context/specs` is listed in `.gitignore`. Self-review: fix placeholders, contradictions, ambiguity, and scope creep inline. Then ask the user to review the written spec; revise if requested. After approval, offer only: hand off to `/harness` with the spec as context, or stop.

Do NOT invoke writing-plans, executing-plans, frontend-design, mcp-builder, or other implementation skills directly.

## Embedded Mode

When invoked by another skill (e.g. `/harness` setup), brainstorming runs in **embedded mode**. The exploration process is the same — understand context, ask clarifying questions, propose approaches — but the output is structured data instead of a spec document.

When the caller is `/harness setup`, embedded mode owns the design exchange. Do not write a design doc, create a git commit, or trigger the standalone review gate. If the conversation started in standalone brainstorming but the user then routes the task into `/harness`, stop the standalone artifact flow and return control to harness.

### Output Schema

Embedded mode produces a YAML block that the caller consumes directly. All fields are advisory — the caller decides which to adopt.

```yaml
goal: "<refined, verifiable goal statement>"
done_when_hint: "<observable acceptance condition>"

boundary_hints:
  mutable:
    - "<file or directory path>"
  immutable:
    - "<file or directory path>"

protocol_hint: direct        # direct | tdd_preferred | tdd_required

check_hints:                 # suggested verification checks
  - name: "<check name>"
    kind: command            # command | review | manual_probe
    action: "<shell command or skill call>"
    cost: cheap              # cheap | medium | expensive

planning_context:            # compact intent anchors for harness; omit empty keys
  non_goals:
    - "<explicit out-of-scope item>"
  constraints:
    - "<hard constraint>"
  assumptions:
    - "<assumption to verify>"
  decisions:
    - "<user-approved decision>"
  risks:
    - "<known risk>"
  open_questions:
    - "<non-blocking question; contract-blocking questions must be asked before output>"

milestones:                  # goal decomposition (2-5 ordered milestones)
  - title: "<milestone title>"
    objective: "<what this milestone achieves>"
    exit_criteria: "<verifiable condition>"
    approaches:
      - hypothesis: "<approach description and rationale>"
        rationale: "<why this ranks here>"
        risk_notes:
          - "<approach-specific risk>"
        score: 70            # 0-100; spread >= 15 between ranks
      - hypothesis: "<alternative approach>"
        rationale: "<why this ranks lower/higher>"
        score: 55
```

Harness mapping: `goal` -> `task.description`, `done_when_hint` -> `task.done_when`, `boundary_hints` -> `boundary`, `protocol_hint` -> `task.protocol`, `check_hints` -> `checks[]`, `planning_context` -> `plan.yaml.planning_context`, `milestones` -> `plan.yaml.milestones[]`. Caller owns validation, infers missing check `kind`, and snapshots full YAML as plan source.

### Embedded Checklist

Embedded mode: explore context, ask clarifying questions, propose 2-3 approaches, then emit the YAML block above. Skip Visual Companion, design doc/commit, spec self-review, standalone user-review gate, and terminal invocation of other skills. Caller owns approval, mapping, and next workflow.

### Mode Detection

Embedded mode activates when the caller's prompt includes `mode: embedded` in the opening context. Example preamble from a harness setup call:

```
mode: embedded
task_id: 002-example-task
caller: /harness setup
goal: "<raw user goal>"
```

When this preamble is present, follow Embedded Mode. When absent, follow the standalone Checklist (steps 1-9).

## Visual Companion

A browser companion for mockups, diagrams, and visual options. It is a tool, not a mode; acceptance only makes it available for visual questions.

**Offering the companion:** When you anticipate that upcoming questions will involve visual content (mockups, layouts, diagrams), offer it once for consent:
> "Some of what we're working on might be easier to explain if I can show it to you in a web browser. I can put together mockups, diagrams, comparisons, and other visuals as we go. This feature is still new and can be token-intensive. Want to try it? (Requires opening a local URL)"

**This offer MUST be its own message.** Do not combine it with clarifying questions, context summaries, or any other content. The message should contain ONLY the offer above and nothing else. Wait for the user's response before continuing. If they decline, proceed with text-only brainstorming.

**Per-question decision:** use browser only when seeing beats reading: mockups, wireframes, layout comparisons, architecture diagrams, side-by-side designs. Use terminal for requirements, conceptual choices, tradeoffs, text options, and scope. UI topic does not automatically mean visual.

If they agree to the companion, read the detailed guide before proceeding:
`skills/brainstorming/visual-companion.md`
