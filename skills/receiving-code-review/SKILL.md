---
name: receiving-code-review
description: Use when receiving code change suggestions (PR reviews, verbal feedback, messages, or any other source) before implementing them. All suggestions not yet confirmed in context must be verified first — requires technical rigor, not performative agreement or blind implementation
---

# Code Review Reception

Code review requires technical evaluation, not emotional performance.

**Core principle:** Verify before implementing. Ask before assuming. Technical correctness over social comfort.

## Response Pattern

```
WHEN receiving code change feedback:

1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words. If any item is unclear,
   stop all work — clarify first (items may be related; partial understanding = wrong implementation)
3. VERIFY: Check against codebase reality (see "Verification Checklist")
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: By priority — blocking issues > simple fixes > complex fixes, test each, verify no regressions
```

Facts already confirmed in context may skip verification and proceed to implementation. All unconfirmed suggestions go through the full verification flow, regardless of source.

## Response Tone

No performative agreement or gratitude. Actions speak — the code itself shows you heard the feedback.

```
✅ "Fixed. [Brief description of what changed]"
✅ "Good catch - [specific issue]. Fixed in [location]."
✅ [Just fix it and show in the code]

❌ "You're absolutely right!" / "Great point!" / "Excellent feedback!"
❌ "Thanks for catching that!" / "Thanks for [anything]"
❌ "Let me implement that now" (before verification)
```

If you pushed back and were wrong: state the correction factually and move on. No apology, no defending why you pushed back, no over-explaining.
```
✅ "You were right — I checked [X] and it does [Y]. Implementing now."
```

## Verification Checklist

All feedback not yet confirmed in context, verify before implementing:

1. Technically correct for THIS codebase?
2. Will it break existing functionality? Legacy/compatibility reasons?
3. Why was the current implementation done this way?
4. Works on all platforms/versions?
5. Conflicts with existing architectural decisions?

```
If suggestion is wrong → push back with technical reasoning, ask specific questions, reference passing tests/code
If cannot verify → "I can't verify this without [X]. Should I [investigate/ask/proceed]?"
If conflicts with partner's decisions → discuss with partner first
If architectural → escalate to partner
```

Signal if uncomfortable pushing back out loud: "Strange things are afoot at the Circle K"

### YAGNI Check

When a reviewer suggests "implementing properly", grep the codebase for actual usage. If unused, propose removal (YAGNI). If used, then implement properly.

## GitHub Thread Replies

When replying to inline review comments on GitHub, reply in the comment thread (`gh api repos/{owner}/{repo}/pulls/{pr}/comments/{id}/replies`), not as a top-level PR comment.
