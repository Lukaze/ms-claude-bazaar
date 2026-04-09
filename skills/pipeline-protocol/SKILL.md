---
name: pipeline-protocol
description: "Shared protocol for the automaton pipeline — label constants, state transitions, claim/handoff/blocked patterns. Not user-invocable. Referenced by pipeline-implement, pipeline-validate, and pipeline-ship skills."
---

# Pipeline Protocol

Shared constants and patterns for all pipeline agents. This skill is NOT user-invocable — it is referenced by the polling skills and agents.

## Label Constants

All labels use the `automaton:` prefix. An issue enters the pipeline when `/architect-and-design` adds both `automaton:pipeline` and `automaton:ready-to-implement`.

| Label | Meaning | Permanent? |
|-------|---------|------------|
| `automaton:pipeline` | Issue is managed by the automaton pipeline | Yes (never removed) |
| `automaton:ready-to-implement` | Design merged, waiting for implement agent | State label |
| `automaton:implementing` | Implement agent has claimed this issue | State label |
| `automaton:ready-to-validate` | PR created + reviews addressed, waiting for validate agent | State label |
| `automaton:validating` | Validate agent has claimed this issue | State label |
| `automaton:ready-to-ship` | Validation passed, waiting for ship agent | State label |
| `automaton:shipping` | Ship agent has claimed this issue | State label |
| `automaton:shipped` | Pipeline complete — PR merged | Terminal |
| `automaton:blocked` | Agent exhausted retry budget, needs human | Blocking |

## State Transitions

```
ready-to-implement  →  implementing        (implement agent claims)
implementing        →  ready-to-validate   (PR created + reviews addressed)
implementing        →  blocked             (retry budget exhausted)

ready-to-validate   →  validating          (validate agent claims)
validating          →  ready-to-ship       (all tests pass clean)
validating          →  ready-to-validate   (fixes pushed, re-review requested)
validating          →  blocked             (fix-and-retest budget exhausted)

ready-to-ship       →  shipping            (ship agent claims)
shipping            →  shipped             (PR merged + issue closed)
shipping            →  blocked             (merge failed)

blocked             →  (human removes label, agent reads last comment for re-entry)
```

## Claim Protocol

When an agent finds an issue in its target `ready-to-*` state, it claims it:

```bash
# 1. Remove the ready label
gh issue edit <NUMBER> --remove-label "automaton:ready-to-implement" --repo <REPO>

# 2. Add the active label
gh issue edit <NUMBER> --add-label "automaton:implementing" --repo <REPO>

# 3. Post claim comment
gh issue comment <NUMBER> --body "## 🤖 Automaton: Implement Agent Claimed
**Time:** $(date -u +%Y-%m-%dT%H:%MZ)
**Agent:** implement
**Action:** Starting implementation based on linked design doc." --repo <REPO>
```

Before claiming, verify:
- Issue has `automaton:pipeline` label
- Issue is assigned to you
- Issue does NOT already have an `*ing` label (implementing/validating/shipping)

## Handoff Protocol

When transitioning to the next state, the agent posts a structured handoff comment. The next agent reads this comment to understand context.

### Implement → Validate Handoff

```markdown
## 🤖 Automaton Handoff: Implementation Complete

**PR:** #<number>
**Branch:** automaton/<issue-number>
**Design doc:** <path-or-link>

### What changed
- <file> — <summary>
- <file> — <summary>

### Review summary
- <reviewer>: <status> (<N> comments addressed)
- <reviewer>: <status>

### Suggested test focus
- <area of concern>
- <transitive dependency>
```

### Validate → Ship Handoff

```markdown
## 🤖 Automaton Handoff: Validation Passed

**PR:** #<number>
**Branch:** automaton/<issue-number>

### Test results
- **Targeted tests:** <N>/<N> passed (<detail>)
- **Blast radius tests:** <N>/<N> passed (<detail>)
- **Full CI:** ruff ✓/✗, mypy ✓/✗, pytest <N>/<N> ✓/✗, proto sync ✓/✗
- **E2E (Playwright):** <N>/<N> passed against full Docker stack

### Fixes applied during validation
- <description> or None

### Verdict
Clear to ship. No regressions detected.
```

## Blocked Protocol

When an agent exhausts its retry budget:

```bash
# 1. Remove current state label
gh issue edit <NUMBER> --remove-label "automaton:validating" --repo <REPO>

# 2. Add blocked label
gh issue edit <NUMBER> --add-label "automaton:blocked" --repo <REPO>

# 3. Post blocked comment
```

The blocked comment MUST include:

```markdown
## 🤖 Automaton Blocked

**Agent:** <implement|validate|ship>
**Retry:** <current>/<max> cycles exhausted
**Last state before blocked:** automaton:<state>

### What was tried
1. <cycle 1 description>
2. <cycle 2 description>
3. <cycle 3 description>

### What's needed
<Human-readable description of what's blocking and what decision is needed>
```

## Unblock Recovery

When a human removes the `automaton:blocked` label, the next polling cycle picks up the issue. The agent:

1. Reads the most recent `🤖 Automaton Blocked` comment
2. Extracts `Last state before blocked` field
3. Restores that state label (e.g., `automaton:ready-to-validate`)
4. The appropriate polling agent picks it up on the next cycle

If no blocked comment is found, the agent posts a comment asking for clarification and re-applies `automaton:blocked`.

## Retry Budgets

| Agent | Budget | What counts as a retry |
|-------|--------|----------------------|
| Implement | 3 cycles | Each round of receive-reviews → address-comments |
| Validate | 3 cycles | Each round of find-failure → fix → retest |
| Ship | 2 attempts | CI iteration failures in ship-it |

## Issue Query Patterns

Each polling skill uses `gh` to find work:

```bash
# Find issues ready for implementation (assigned to me)
gh issue list --label "automaton:pipeline" --label "automaton:ready-to-implement" --assignee "@me" --json number,title,labels,body --repo <REPO>

# Find issues ready for validation
gh issue list --label "automaton:pipeline" --label "automaton:ready-to-validate" --assignee "@me" --json number,title,labels,body --repo <REPO>

# Find issues ready to ship
gh issue list --label "automaton:pipeline" --label "automaton:ready-to-ship" --assignee "@me" --json number,title,labels,body --repo <REPO>

# Find blocked issues that had their blocked label removed (need recovery)
gh issue list --label "automaton:pipeline" --assignee "@me" --json number,title,labels --repo <REPO>
# Then filter: has automaton:pipeline but no state label → needs recovery
```

## Repo Detection

Agents detect the target repo from the current git remote:

```bash
gh repo view --json nameWithOwner -q '.nameWithOwner'
```

This ensures the pipeline works in any repo that has the `automaton:` labels configured.
