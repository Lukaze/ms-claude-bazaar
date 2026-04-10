---
name: pipeline-implement
description: "Use when you want to start the implement agent polling loop. Watches for GitHub issues labeled automaton:ready-to-implement, claims them, and dispatches the implement-agent to build the feature from the design doc."
---

# Pipeline Implement

Polling loop that watches for `automaton:ready-to-implement` issues and dispatches the implement-agent.

**Run via:** `/loop 5m /pipeline-implement`

## Polling Cycle

Each invocation performs ONE cycle. Dispatches are **fire-and-forget** — the cycle does NOT wait for the agent to finish. This allows the next tick to pick up new work or check on stale agents.

### Step 1: Detect Repo

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
```

### Step 2: Check Stale In-Flight Agents

Before looking for new work, check for issues stuck in `automaton:implementing`:

```bash
gh issue list \
  --label "automaton:pipeline" \
  --label "automaton:implementing" \
  --assignee "@me" \
  --state open \
  --json number,title,updatedAt,labels \
  --repo "$REPO"
```

For each issue in `automaton:implementing`:
1. Read the issue comments to find the most recent `🤖 Automaton` comment
2. Check how long ago it was posted
3. **If >30 minutes since the last agent comment** and no transition happened:
   - The agent likely crashed or stalled
   - Transition to `automaton:blocked` with comment: "Implement agent stalled — no activity for 30+ minutes. Last comment: <timestamp>."
4. **If <30 minutes:** Agent is still working, leave it alone

This prevents issues from getting stuck in `implementing` indefinitely when an agent crashes without transitioning.

### Step 3: Check for Blocked Recovery

Check for issues that have `automaton:pipeline` but NO state label (someone removed `automaton:blocked` and the issue needs re-entry):

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

### Step 4: Find New Work

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

If **no issues found:** Exit cleanly. The loop will invoke again on the next tick.

If **multiple issues found:** Pick the oldest one (lowest issue number). Dispatch one new agent per cycle.

### Step 5: Pre-Flight Check

Before dispatching, verify the issue isn't already done:

1. **Check if issue is already shipped:** Look for `automaton:shipped` label or closed state
2. **Check if a PR already exists:** Search for PRs referencing this issue number
   ```bash
   gh pr list --search "<issue-number>" --state open --json number,title --repo "$REPO"
   ```
3. **Check if the design PR branch already has implementation commits** (another agent may have done it)

If the work is already done, remove `automaton:ready-to-implement` and add the appropriate next-state label (or `automaton:shipped`). Post a comment explaining. Do NOT dispatch an agent for already-completed work.

### Step 6: Claim and Dispatch (Fire-and-Forget)

For the selected issue:

1. **Claim** per pipeline-protocol:
   - Remove `automaton:ready-to-implement`
   - Add `automaton:implementing`
   - Post claim comment on the issue

2. **Dispatch** the `implement-agent` **in the background** with:
   - Issue number
   - Repo name
   - Issue body (contains design doc reference)

3. **Exit immediately.** Do NOT wait for the agent to complete. The next tick will check on it via Step 2 (stale agent check).

## Error Handling

- If `gh` commands fail (auth, network), log the error and exit. The loop retries next tick.
- If the implement-agent dispatch itself fails, apply `automaton:blocked` to the issue with error details.
- Step 2 handles the case where an agent crashes without transitioning — stale agents get blocked after 30 minutes.
