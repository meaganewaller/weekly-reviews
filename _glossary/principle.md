---
layout: glossary-term
term: Principle
short: Architectural guideline invoked in decisions
category: analysis
related:
  - tradeoff
  - decision
aliases:
  - principles
  - architectural principle
---

An architectural guideline invoked when making decisions, captured in tradeoff documentation.

## Common Principles

| Principle | Meaning |
|-----------|---------|
| Explicit over implicit | Favor clarity over magic |
| Convention over configuration | Use sensible defaults |
| Self-contained artifacts | Minimize dependencies |
| Progressive enhancement | Build incrementally |
| Design for failure | Expect and handle errors |
| Optimize for the common case | Fast path for typical usage |
| Documentation as specification | Docs are source of truth |

## Tracking

Principles are extracted from `decision_tradeoff` events:

```json
{
  "event_type": "decision_tradeoff",
  "payload": {
    "principles": [
      "explicit over implicit",
      "convention over configuration"
    ]
  }
}
```

## In Weekly Reviews

Principles appear in:
- **Principles invoked** chart
- **Architectural thinking** analysis
- **Blindspot detection** - missing principles

## Analysis

The weekly review analyzes principle usage:
- Distribution across decisions
- Missing principles (e.g., no resilience principles despite failures)
- Consistency of application
