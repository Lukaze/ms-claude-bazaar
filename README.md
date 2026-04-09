# ms-claude-bazaar

Autonomous dev pipeline plugin for Claude Code CLI.

Orchestrates the full lifecycle from design → implement → validate → ship through three independent polling agents communicating via GitHub issue labels.

## How It Works

1. `/architect-and-design` creates a GitHub issue with `automaton:pipeline` + `automaton:ready-to-implement` labels
2. **Implement Agent** picks it up, uses `/feature-dev` to build it, creates a PR, waits for reviews, addresses feedback
3. **Validate Agent** picks it up after reviews, runs targeted tests → blast radius analysis → full CI → Playwright E2E
4. **Ship Agent** picks it up after validation, runs `/ship-it` to squash-merge

Each agent runs as a `/loop` on a separate dev box:

```bash
# DevBox 1
/loop 5m /pipeline-implement

# DevBox 2
/loop 5m /pipeline-validate

# DevBox 3
/loop 5m /pipeline-ship
```

## Dashboard

Check pipeline status anytime:

```
/pipeline-status
```

## Design

See [docs/2026-04-08-automaton-pipeline-design.md](docs/2026-04-08-automaton-pipeline-design.md) for the full design spec.

## License

MIT
