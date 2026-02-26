---
layout: home
title: About
permalink: /about/
---

# Weekly Engineering Reviews

This site publishes automated weekly engineering reviews generated from my development activity across all projects.

## How It Works

These reviews are powered by **Dev OS**, a personal telemetry system built on top of [Claude Code](https://claude.ai/claude-code). The system observes engineering patterns in real-time and surfaces insights at the end of each week.

### The Data Pipeline

```
Claude Code Sessions
        │
        ▼
┌───────────────────┐
│  Event Hooks      │
│  ────────────     │
│  • Impact writes  │
│  • Friction logs  │
│  • Test results   │
│  • Decisions      │
└───────────────────┘
        │
        ▼
~/.claude/dev-os-events.jsonl
        │
        ▼
┌───────────────────┐
│  /weekly-review   │
│  ────────────     │
│  • Aggregates     │
│  • Analyzes       │
│  • Synthesizes    │
└───────────────────┘
        │
        ▼
    This Site
```

### What Gets Tracked

| Event Type | Description |
|------------|-------------|
| `tool_write` | Files created or edited, with risk assessment |
| `tool_failure` | Errors classified into friction domains |
| `large_change` | Diffs exceeding 250 lines |
| `reversal` | When recent work gets undone |
| `decision_tradeoff` | Architectural decisions with documented tradeoffs |
| `test_run` | Test execution results |

### Friction Taxonomy

Tool failures are classified into domains to identify skill gaps:

- **syntax** - Parse errors, malformed input
- **type** - Type mismatches, inference failures
- **dependency** - Missing packages, version conflicts
- **permission** - Access denied, auth failures
- **state** - File not found, resource limits
- **config** - Env vars, misconfiguration
- **testing** - Test failures, assertions
- **build** - Compilation, bundling errors

## The Review Process

Each week, the `/weekly-review` skill:

1. **Aggregates** all events from the past 7 days across every project
2. **Generates** charts and metrics (commits, friction patterns, test stability)
3. **Synthesizes** staff-level analysis including:
   - Executive summary with wins and concerns
   - Friction analysis with practice recommendations
   - Architecture and principles assessment
   - Promotion-ready impact bullets
   - Precision moves for the next week

## Source

The Dev OS configuration lives in my dotfiles:

- **Repository**: [github.com/meaganewaller/.dotfiles](https://github.com/meaganewaller/.dotfiles)
- **Path**: `home/.claude/`

Key components:
- `hooks/` - Event handlers that capture telemetry
- `skills/common/weekly-review/` - The review generation skill
- `settings/` - Profile-aware Claude Code configuration

## Why This Exists

Engineering work is often invisible. Features ship, bugs get fixed, but the patterns of how work gets done—the friction encountered, the decisions made, the reversals and recoveries—fade from memory.

This system creates a persistent record that surfaces:
- Where I'm spending time (and where I'm stuck)
- What architectural principles guide my decisions
- How my skills are evolving week over week

It's a feedback loop for deliberate practice.
