---
name: pipeline-status
description: "Use when you want to see the status of all issues in the Automaton pipeline. Shows a dashboard table with issue number, title, current state, linked PR, time in state, and last agent comment."
---

# Pipeline Status

Dashboard showing all issues in the Automaton pipeline.

## Usage

Just invoke: `/pipeline-status`

## What It Does

1. Query all open issues with the `automaton:pipeline` label:
   ```bash
   REPO=$(gh repo view --json nameWithOwner -q '.nameWithOwner')
   gh issue list \
     --label "automaton:pipeline" \
     --assignee "@me" \
     --state all \
     --json number,title,labels,updatedAt,comments,state \
     --limit 50 \
     --repo "$REPO"
   ```

2. For each issue, extract:
   - **Issue #** and **title**
   - **Current state** from `automaton:*` labels (ready-to-implement, implementing, etc.)
   - **Linked PR** (from comments containing "PR: #")
   - **Time in current state** (from the timestamp of the last state-change comment)
   - **Last agent action** (most recent `🤖 Automaton` comment, truncated)

3. Display as a formatted table:

```
Automaton Pipeline Status
═══════════════════════════════════════════════════════════════════════════════

 #   │ Title                          │ State              │ PR    │ Time    │ Last Action
─────┼────────────────────────────────┼────────────────────┼───────┼─────────┼──────────────────
 42  │ Add session restore logging    │ 🔨 implementing    │ #1234 │ 35 min  │ Addressing review comments
 38  │ Fix MRU delta enrichment       │ ✅ ready-to-ship   │ #1230 │ 5 min   │ Validation passed
 35  │ Refactor throttle state        │ 🚀 shipping        │ #1225 │ 2 min   │ Running ship-it
 31  │ Add CV to gRPC metadata        │ 🏁 shipped         │ #1220 │ 1 day   │ Merged
 29  │ Update proto for new events    │ 🚫 blocked         │ #1218 │ 3 hrs   │ Fix loop exhausted — flaky test

═══════════════════════════════════════════════════════════════════════════════
 Pipeline: 5 issues (1 implementing, 1 ready-to-ship, 1 shipping, 1 shipped, 1 blocked)
```

## State Icons

| State | Icon |
|-------|------|
| `ready-to-implement` | 📋 |
| `implementing` | 🔨 |
| `ready-to-validate` | 🧪 |
| `validating` | 🔬 |
| `ready-to-ship` | ✅ |
| `shipping` | 🚀 |
| `shipped` | 🏁 |
| `blocked` | 🚫 |

## Options

- Show only active issues (default): excludes `shipped`
- Show all: `--all` argument to include shipped/closed issues
- Show one issue detail: `--issue 42` to show full comment history for a specific issue
