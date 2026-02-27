---
layout: page
title: Review Pipeline
permalink: /review-pipeline/
---

# The Weekly Review Pipeline

How raw events become staff-level insights.

## Pipeline Overview

The `/weekly-review` skill runs four scripts in sequence, then performs AI synthesis:

{% mermaid %}
flowchart LR
    subgraph S1["Stage 1: Aggregate"]
        A1[(events.jsonl)] --> A2[aggregate.sh]
        A2 --> A3[summary.json]
    end
    subgraph S2["Stage 2: Visualize"]
        B1[summary.json] --> B2[charts.py]
        B2 --> B3[*.png charts]
    end
    subgraph S3["Stage 3: Template"]
        C1[summary.json] --> C2[render_md.sh]
        C2 --> C3[review.md]
    end
    subgraph S4["Stage 4: Synthesize"]
        D1[review.md] --> D2[AI Synthesis]
        D2 --> D3[Final review]
    end
    S1 --> S2 --> S3 --> S4
{% endmermaid %}

## Stage 1: Aggregate

`aggregate.sh` processes the raw event log and computes metrics.

### Input

```
~/.claude/dev-os-events.jsonl
```

Each line is a JSON event:

```json
{
  "timestamp": "2026-02-27T10:30:00Z",
  "session_id": "abc123",
  "event_type": "tool_write",
  "payload": {
    "files": ["service.rb"],
    "change_type": "refactor",
    "risk_level": "low"
  }
}
```

### Processing

The script:
1. Filters events from the last 7 days
2. Groups by event type
3. Extracts unique projects from file paths
4. Counts sessions
5. Aggregates friction by domain/subdomain
6. Collects principles invoked in decisions
7. Calculates test pass rates

### Output

`summary.json`:

```json
{
  "window": {
    "start": "2026-02-20",
    "end": "2026-02-27"
  },
  "metrics": {
    "total_events": 1259,
    "writes": 541,
    "failures": 568,
    "large_changes": 13,
    "reversals": 2,
    "decisions": 2,
    "test_runs": 116,
    "test_passes": 50
  },
  "projects": {
    "dotfiles": { "events": 676, "writes": 329, "failures": 327 },
    "pull": { "events": 522, "writes": 198, "failures": 212 }
  },
  "friction": {
    "by_domain": { "state": 511, "dependency": 3 },
    "by_subdomain": { "state:file-not-found": 232 }
  },
  "principles": {
    "self-contained artifacts": 1,
    "explicit over implicit": 1
  },
  "files_modified": ["path/to/file.rb", "..."]
}
```

## Stage 2: Visualize

`charts.py` generates visual representations of the data.

### Charts Generated

**events_by_type.png** - Distribution of event types

{% mermaid %}
xychart-beta
    title "Events by Type"
    x-axis ["tool_write", "tool_failure", "test_run", "large_change"]
    y-axis "Count" 0 --> 600
    bar [541, 568, 116, 13]
{% endmermaid %}

**friction_domains.png** - Friction breakdown by domain

{% mermaid %}
pie title Friction by Domain
    "state" : 511
    "unknown" : 48
    "dependency" : 3
{% endmermaid %}

**principles_invoked.png** - Architectural principles cited

{% mermaid %}
pie title Principles Invoked
    "self-contained" : 1
    "explicit/implicit" : 1
{% endmermaid %}

### Output

Charts saved to `~/.claude/reviews/week-of-YYYY-MM-DD/`:
- `events_by_type.png`
- `friction_domains.png`
- `principles_invoked.png`

## Stage 3: Template

`render_md.sh` generates the review markdown with placeholders.

### Template Structure

```markdown
---
layout: review
title: "Weekly Engineering Review — YYYY-MM-DD"
date: YYYY-MM-DD
---

**Window:** YYYY-MM-DD → YYYY-MM-DD

## Executive Summary

{{EXECUTIVE_SUMMARY}}

## Execution Metrics

### Overview
| Metric | Value |
|--------|-------|
| Projects touched | N |
| Sessions | N |
| Total events | N |
...

## Repeated Friction

### By Domain
- **state**: N
...

### Analysis

{{FRICTION_ANALYSIS}}

## Architectural Thinking

### Principles Invoked
- principle: N

### Analysis

{{ARCHITECTURE_ANALYSIS}}

## Discipline Flags

{{DISCIPLINE_FLAGS}}

## Cue Engagement

{{CUE_ANALYSIS}}

## Promotion-Ready Impact Bullets

{{IMPACT_BULLETS}}

## Precision Moves for Next Week

{{PRECISION_MOVES}}
```

### Output

`review.md` with metrics filled in and `{{PLACEHOLDER}}` markers for AI synthesis.

## Stage 4: AI Synthesis

Claude reads the summary and template, then fills in the six placeholders:

### {{EXECUTIVE_SUMMARY}}

Staff-level assessment including:
- Overall characterization
- Key wins (3-4 bullets)
- Concerns (3-4 bullets)
- Week character summary

### {{FRICTION_ANALYSIS}}

Deep analysis of top friction domains:
- Root cause identification
- Pattern recognition
- Deliberate practice recommendations

### {{ARCHITECTURE_ANALYSIS}}

Assessment of architectural thinking:
- Principle distribution analysis
- Strengths demonstrated
- Blindspots identified
- Recommendations

### {{DISCIPLINE_FLAGS}}

Evaluation of engineering discipline:
- Large changes without tradeoff documentation
- Reversal patterns
- Dependency churn assessment

### {{IMPACT_BULLETS}}

Promotion-ready impact bullets:
- Action verb + deliverable format
- Quantified where possible
- Business-relevant framing

### {{CUE_ANALYSIS}}

Analysis of cue engagement:
- Which cues fired
- Trigger patterns
- Dormant cue assessment
- Recommendations for improvement

### {{PRECISION_MOVES}}

Three actionable recommendations for next week:
1. **Architecture Focus** - System design improvement
2. **Skill Deepening** - Deliberate practice target
3. **Leverage Move** - Multiplier action

## Final Output

The completed review includes:

- **Metrics** - Quantified activity
- **Friction analysis** - Where time was lost
- **Architectural assessment** - Design thinking quality
- **Discipline flags** - Process compliance
- **Impact framing** - Promotion-ready narrative
- **Action items** - Concrete next steps

## Pipeline Timing

| Stage | Duration | Notes |
|-------|----------|-------|
| Aggregate | ~5s | jq processing |
| Charts | ~10s | matplotlib rendering |
| Template | ~2s | string substitution |
| AI Synthesis | ~60s | Claude analysis |

Total: ~80 seconds for a full week's review.

## Customization

### Aggregation Window

Default is 7 days. Override with:

```bash
REVIEW_DAYS=14 ~/.claude/skills/weekly-review/scripts/aggregate.sh
```

### Chart Styling

Edit `charts.py` to modify:
- Color schemes
- Figure sizes
- Label formatting

### Template Sections

Edit `render_md.sh` to add/remove sections or change structure.

---

Previous: [Friction Taxonomy](/friction-taxonomy/) | Next: [Hooks & Cues](/hooks-and-cues/)
