---
name: pipeline-validate
description: "Use when you want to start the validate agent polling loop. Watches for GitHub issues labeled automaton:ready-to-validate, claims them, and dispatches the validate-agent to run three-layer testing (targeted, blast radius, full stack + Playwright E2E)."
---

# Pipeline Validate

Polling loop that watches for `automaton:ready-to-validate` issues and dispatches the validate-agent.

**Run via:** `/loop 5m /pipeline-validate`

**Prerequisites:** This agent's dev box needs Docker, Playwright, and full stack prerequisites installed (for `/run-aether` and `/test-e2e`).

## Polling Cycle

Each invocation performs ONE cycle. Dispatches are **fire-and-forget** — the cycle does NOT wait for the agent to finish.

### Step 1: Detect Repo

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
```

### Step 2: Check Stale In-Flight Agents

Check for issues stuck in `automaton:validating`:

```bash
gh issue list \
  --label "automaton:pipeline" \
  --label "automaton:validating" \
  --assignee "@me" \
  --state open \
  --json number,title,updatedAt,labels \
  --repo "$REPO"
```

For each: if >30 minutes since last `🤖 Automaton` comment with no transition, mark `automaton:blocked` with stale agent comment.

### Step 3: Find Work

Query for issues that are ready for validation and assigned to you:

```bash
gh issue list \
  --label "automaton:pipeline" \
  --label "automaton:ready-to-validate" \
  --assignee "@me" \
  --state open \
  --json number,title,body,labels \
  --repo "$REPO"
```

### Step 4: Triage

If **no issues found:** Exit cleanly.

If **multiple issues found:** Pick the oldest one (lowest issue number).

### Step 5: Check for Blocked Recovery

Same as pipeline-implement, but for validation-related states:
- Check for `automaton:pipeline`-only issues (no state label)
- Read the blocked comment
- If `Last state before blocked` is `automaton:ready-to-validate` or `automaton:validating`, restore `automaton:ready-to-validate`
- Otherwise, skip (belongs to a different agent)

### Step 6: Extract PR Number

1. Read the issue comments to find the most recent `🤖 Automaton Handoff` comment
2. Extract the PR number from the handoff comment (`**PR:** #<number>`)
3. If no handoff comment exists, also check the issue body for a `PR #` reference
4. If no PR number can be found, skip this issue — it may have been transitioned incorrectly

**Note:** The implement agent may have pushed to an existing design PR (not created a new one). The PR number in the handoff comment is the source of truth.

### Step 7: Claim and Dispatch (Fire-and-Forget)

For the selected issue:

1. **Claim** per pipeline-protocol:
   - Remove `automaton:ready-to-validate`
   - Add `automaton:validating`
   - Post claim comment on the issue

2. **Dispatch** the `validate-agent` with:
   - Issue number
   - Repo name
   - PR number (from handoff comment)
   - Handoff context (changed files, suggested test focus)

3. **Exit immediately.** Do NOT wait for the agent to complete. The next tick checks for stale agents via Step 2.

## Error Handling

- If `gh` commands fail, log and exit. Loop retries next tick.
- If validate-agent dispatch itself fails, apply `automaton:blocked` with error details.
- Step 2 handles stale agents — if an agent crashes without transitioning, it gets blocked after 30 minutes.
