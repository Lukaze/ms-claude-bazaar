---
name: pipeline-implement
description: "Use when you want to start the implement agent polling loop. Watches for GitHub issues labeled automaton:ready-to-implement, claims them, and dispatches the implement-agent to build the feature from the design doc."
---

# Pipeline Implement

Polling loop that watches for `automaton:ready-to-implement` issues and dispatches the implement-agent.

**Run via:** `/loop 5m /pipeline-implement`

## Polling Cycle

Each invocation performs ONE cycle:

### Step 1: Detect Repo

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
```

### Step 2: Find Work

Query for issues that are ready for implementation and assigned to you:

```bash
gh issue list \
  --label "automaton:pipeline" \
  --label "automaton:ready-to-implement" \
  --assignee "@me" \
  --state open \
  --json number,title,body,labels \
  --repo "$REPO"
```

### Step 3: Triage

If **no issues found:** Exit cleanly. The loop will invoke again on the next tick.

If **multiple issues found:** Pick the oldest one (lowest issue number). Process one issue per cycle.

### Step 4: Check for Blocked Recovery

Before looking at ready-to-implement issues, also check for issues that have `automaton:pipeline` but NO state label (someone removed `automaton:blocked` and the issue needs re-entry):

```bash
gh issue list \
  --label "automaton:pipeline" \
  --assignee "@me" \
  --state open \
  --json number,title,labels \
  --repo "$REPO"
```

Filter for issues where the only `automaton:` label is `automaton:pipeline` (no state label). For these:
1. Read the last `🤖 Automaton Blocked` comment
2. Extract `Last state before blocked`
3. If the state is `automaton:ready-to-implement` or `automaton:implementing`, restore the `automaton:ready-to-implement` label
4. Otherwise, this issue belongs to a different agent — skip it

### Step 5: Claim and Dispatch

For the selected issue:

1. **Claim** per pipeline-protocol:
   - Remove `automaton:ready-to-implement`
   - Add `automaton:implementing`
   - Post claim comment on the issue

2. **Dispatch** the `implement-agent` with:
   - Issue number
   - Repo name
   - Issue body (contains design doc reference)

3. The implement-agent handles everything from here through handoff.

### Step 6: Post-Dispatch

After the implement-agent completes (success or blocked):
- Verify the issue has been transitioned to either `automaton:ready-to-validate` or `automaton:blocked`
- If the agent crashed without transitioning, apply `automaton:blocked` with a comment noting the unexpected failure

## Error Handling

- If `gh` commands fail (auth, network), log the error and exit. The loop retries next tick.
- If the implement-agent crashes, catch the error, label the issue `automaton:blocked`, and post a comment with the error details.
- Never leave an issue in `automaton:implementing` without an active agent working on it.
