---
name: pipeline-validate
description: "Use when you want to start the validate agent polling loop. Watches for GitHub issues labeled automaton:ready-to-validate, claims them, and dispatches the validate-agent to run three-layer testing (targeted, blast radius, full stack + Playwright E2E)."
---

# Pipeline Validate

Polling loop that watches for `automaton:ready-to-validate` issues and dispatches the validate-agent.

**Run via:** `/loop 5m /pipeline-validate`

**Prerequisites:** This agent's dev box needs Docker, Playwright, and full stack prerequisites installed (for `/run-aether` and `/test-e2e`).

## Polling Cycle

Each invocation performs ONE cycle:

### Step 1: Detect Repo

```bash
REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
```

### Step 2: Find Work

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

### Step 3: Triage

If **no issues found:** Exit cleanly.

If **multiple issues found:** Pick the oldest one (lowest issue number).

### Step 4: Check for Blocked Recovery

Same as pipeline-implement, but for validation-related states:
- Check for `automaton:pipeline`-only issues (no state label)
- Read the blocked comment
- If `Last state before blocked` is `automaton:ready-to-validate` or `automaton:validating`, restore `automaton:ready-to-validate`
- Otherwise, skip (belongs to a different agent)

### Step 5: Pre-Claim Verification

Before claiming, verify the PR has the required reviews:

1. Read the most recent handoff comment to get the PR number
2. Check PR reviews:
   ```bash
   gh pr view <PR_NUMBER> --json reviews --repo "$REPO"
   ```
3. Required: Automaton (lukas) approval + 1 other approval
4. If reviews are not present, **do not claim**. Exit and let the next tick check again. The implement agent already requested reviews — they just haven't landed yet.

### Step 6: Claim and Dispatch

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

3. The validate-agent handles three-layer testing through handoff.

### Step 7: Post-Dispatch

After the validate-agent completes:
- Verify the issue has been transitioned to `automaton:ready-to-ship`, `automaton:ready-to-validate` (re-review loop), or `automaton:blocked`
- If the agent crashed without transitioning, apply `automaton:blocked` with error details

## Error Handling

- If `gh` commands fail, log and exit. Loop retries next tick.
- If validate-agent crashes, label `automaton:blocked` with crash details.
- If Docker stack fails to start, the validate-agent handles this (transitions to blocked). The polling skill doesn't need to manage Docker.
- Never leave an issue in `automaton:validating` without an active agent.
