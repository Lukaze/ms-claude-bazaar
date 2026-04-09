---
name: pipeline-ship
description: "Use when you want to start the ship agent polling loop. Watches for GitHub issues labeled automaton:ready-to-ship, claims them, and runs the /ship-it skill to iterate CI, squash-merge the PR, and close the issue."
---

# Pipeline Ship

Polling loop that watches for `automaton:ready-to-ship` issues and runs `/ship-it` to merge.

**Run via:** `/loop 5m /pipeline-ship`

## Polling Cycle

Each invocation performs ONE cycle:

### Step 1: Detect Repo

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
```

### Step 2: Find Work

```bash
gh issue list \
  --label "automaton:pipeline" \
  --label "automaton:ready-to-ship" \
  --assignee "@me" \
  --state open \
  --json number,title,body,labels \
  --repo "$REPO"
```

### Step 3: Triage

If **no issues found:** Exit cleanly.

If **multiple issues found:** Pick the oldest one (lowest issue number).

### Step 4: Check for Blocked Recovery

Same pattern as other agents:
- Check for `automaton:pipeline`-only issues
- If `Last state before blocked` is `automaton:ready-to-ship` or `automaton:shipping`, restore `automaton:ready-to-ship`

### Step 5: Claim

1. **Claim** per pipeline-protocol:
   - Remove `automaton:ready-to-ship`
   - Add `automaton:shipping`
   - Post claim comment on the issue

### Step 6: Ship

1. Read the validation handoff comment to get the PR number
2. Invoke `/ship-it <PR_NUMBER>`
   - ship-it handles: iterating CI, addressing any last comments, squash-merging with --delete-branch
3. Retry budget: 2 attempts. If ship-it fails twice, transition to `automaton:blocked`.

### Step 7: Close Out

After successful merge:

1. Remove `automaton:shipping` label
2. Add `automaton:shipped` label
3. Close the issue with a final comment:

```markdown
## 🤖 Automaton: Shipped

**PR:** #<number> — merged via squash
**Branch:** automaton/<issue-number> — deleted
**Pipeline duration:** <time from ready-to-implement to shipped>

This issue has been fully implemented, validated, and shipped through the Automaton pipeline.
```

## Error Handling

- If `/ship-it` fails due to merge conflicts, attempt rebase on main and retry.
- If `/ship-it` fails due to CI, let ship-it's own iteration handle it (it has its own retry logic).
- If the PR was already merged (by a human), just update labels to `automaton:shipped` and close the issue.
- Never leave an issue in `automaton:shipping` without resolution.
