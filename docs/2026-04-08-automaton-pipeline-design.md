# Automaton Pipeline Design

**Date:** 2026-04-08
**Status:** v0.1 — Initial implementation

## Overview

An autonomous dev pipeline for Claude Code CLI that takes a designed GitHub issue through implementation → validation → shipping via three independent polling agents communicating through GitHub issue labels and comments.

## Architecture

```
/architect-and-design (human + AI)
        │
        │ creates issue + labels: automaton:pipeline + automaton:ready-to-implement
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ DevBox-1: /loop 5m /pipeline-implement                          │
│                                                                 │
│  Polls for: automaton:ready-to-implement                        │
│  Dispatches: implement-agent                                    │
│  Does: /feature-dev → PR → wait reviews → address comments      │
│  Outputs: automaton:ready-to-validate                           │
└─────────────────────────────────────────────────────────────────┘
        │
        │ human review gate: Automaton (required) + 1 other
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ DevBox-2: /loop 5m /pipeline-validate                           │
│                                                                 │
│  Polls for: automaton:ready-to-validate                         │
│  Dispatches: validate-agent                                     │
│  Does: targeted tests → blast radius → full CI → Playwright E2E │
│  Outputs: automaton:ready-to-ship (or fix loop → re-review)     │
└─────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│ DevBox-3: /loop 5m /pipeline-ship                               │
│                                                                 │
│  Polls for: automaton:ready-to-ship                             │
│  Does: /ship-it → squash merge → close issue                    │
│  Outputs: automaton:shipped                                     │
└─────────────────────────────────────────────────────────────────┘
```

## State Machine

| From | To | Trigger |
|------|----|---------|
| `ready-to-implement` | `implementing` | Implement agent claims |
| `implementing` | `ready-to-validate` | PR created + reviews addressed |
| `implementing` | `blocked` | Retry budget exhausted |
| `ready-to-validate` | `validating` | Validate agent claims (after verifying reviews) |
| `validating` | `ready-to-ship` | All tests pass clean |
| `validating` | `ready-to-validate` | Fixes pushed, re-review requested |
| `validating` | `blocked` | Fix-and-retest budget exhausted |
| `ready-to-ship` | `shipping` | Ship agent claims |
| `shipping` | `shipped` | PR merged |
| `shipping` | `blocked` | Merge failed |
| `blocked` | *(re-entry)* | Human removes label, agent reads comment for state |

## Label Protocol

All labels prefixed `automaton:`. The `automaton:pipeline` label is permanent and marks issues as pipeline-managed. State labels rotate.

## Communication

All agent-to-agent communication happens through GitHub issue comments using structured formats (see pipeline-protocol skill). No side channels.

## Review Gate

- Automaton (lukas): Required on every PR
- One additional reviewer from {Bas, Marius, ...}: Required
- Validate agent verifies approvals before claiming

## Validation Layers

1. **Targeted tests** — changed files mapped to their test files, run first for fast feedback
2. **Blast radius** — import/call graph tracing to find transitive dependents, run those tests
3. **Full stack** — /full-ci-check (ruff, mypy, pytest, proto) + /run-aether (Docker stack) + /test-e2e (Playwright)

All three layers must pass. Playwright E2E against the full Docker stack is mandatory.

## Retry Budgets

| Agent | Budget | Scope |
|-------|--------|-------|
| Implement | 3 cycles | Review-address rounds |
| Validate | 3 cycles | Fix-and-retest rounds |
| Ship | 2 attempts | CI/merge attempts |

Exhausted budget → `automaton:blocked` + detailed comment.

## Scope

- Personal pipeline only (issues assigned to @me)
- Only issues labeled `automaton:pipeline` (created by /architect-and-design)
- Does not pick up bugs, external issues, or unlabeled work

## Plugin Structure

```
plugin-repo/
├── .claude-plugin/plugin.json
├── skills/
│   ├── pipeline-protocol/SKILL.md     (shared protocol, not user-invocable)
│   ├── pipeline-implement/SKILL.md    (polling loop)
│   ├── pipeline-validate/SKILL.md     (polling loop)
│   ├── pipeline-ship/SKILL.md         (polling loop)
│   └── pipeline-status/SKILL.md       (dashboard)
├── agents/
│   ├── implement-agent.md
│   ├── validate-agent.md
│   ├── review-evaluator.md
│   └── blast-radius-analyzer.md
└── docs/
    └── 2026-04-08-automaton-pipeline-design.md
```

## Dependencies on Aether Skills

The pipeline agents delegate to existing Aether project skills:
- `/feature-dev` — all code writing
- `/full-ci-check` — CI validation
- `/run-aether` — start full Docker stack
- `/test-e2e` — Playwright E2E
- `/ship-it` — PR merge
- `/sync-main` — branch sync

## Required Aether Modification

`/architect-and-design` must be modified to add `automaton:pipeline` + `automaton:ready-to-implement` labels when creating GitHub issues.
