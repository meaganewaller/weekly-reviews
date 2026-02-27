---
layout: page
title: Data Capture
permalink: /data-capture/
---

# Data Capture

How development activity becomes structured events.

## The Hook System

Claude Code supports hooks—shell scripts that run in response to events. Dev OS uses hooks to observe activity and emit structured telemetry.

{% mermaid %}
flowchart TB
    subgraph session["Claude Code Session"]
        subgraph pre["PreToolUse"]
            P1[cue inject]
        end
        subgraph post["PostToolUse"]
            P2[impact extractor]
            P3[large-diff]
            P4[reversal]
        end
        subgraph fail["PostToolUseFailure"]
            P5[skill-gap detector]
        end
        subgraph start["SessionStart"]
            P6[friction escalator]
            P7[context injector]
        end
        subgraph stop["Stop"]
            P8[test blocker]
            P9[tradeoff blocker]
        end
    end
    session --> DB[(~/.claude/<br/>dev-os-events.jsonl)]
{% endmermaid %}

## Hook Categories

### Observation Hooks

Run after tool execution to log what happened.

| Hook | Trigger | What it captures |
|------|---------|------------------|
| `impact-extractor.sh` | PostToolUse | File writes with metadata |
| `skill-gap-detector.sh` | PostToolUseFailure | Errors classified by domain |
| `reversal-detector.sh` | PostToolUse | When recent work is undone |
| `large-diff-escalator.sh` | PostToolUse | Changes > 250 lines |

### Enforcement Hooks

Block actions when conditions aren't met.

| Hook | Trigger | What it enforces |
|------|---------|------------------|
| `hard-stop-test-blocker.sh` | Stop | Last test must pass |
| `pending-tradeoff-blocker.sh` | Stop | Large changes need docs |
| `task-gate.sh` | TaskCompleted | Task completion criteria |

### Injection Hooks

Add context to the session.

| Hook | Trigger | What it injects |
|------|---------|-----------------|
| `session-context-injector.sh` | SessionStart | Recent impact/friction |
| `friction-escalator.sh` | SessionStart | Repeated error warnings |
| `cue-injector-*.sh` | PreToolUse | Matching cue guidance |

## Event Creation Flow

### Code Changes

When you edit a file:

{% mermaid %}
flowchart TD
    A[Edit service.rb] --> B[impact-extractor.sh]
    B --> C[Extracts file path]
    B --> D[Classifies change type]
    B --> E[Assesses risk level]
    C & D & E --> F[tool_write event]
{% endmermaid %}

**Example event:**
```json
{
  "event_type": "tool_write",
  "payload": {
    "files": ["service.rb"],
    "change_type": "refactor",
    "risk_level": "low"
  }
}
```

### Auto-Classification

The system guesses what kind of work you're doing:

{% mermaid %}
flowchart TD
    A[File path] --> B{Contains test/spec?}
    B -->|Yes| C[change_type: test]
    B -->|No| D{Contains config?}
    D -->|Yes| E[change_type: infra]
    D -->|No| F[change_type: refactor]
{% endmermaid %}

### Risk Assessment

{% mermaid %}
flowchart TD
    A[Diff content] --> B{Has class/module/def?}
    B -->|Yes| C[risk: medium<br/>change_type: architecture]
    B -->|No| D[risk: low]
{% endmermaid %}

## Error Classification

When a tool fails, `skill-gap-detector.sh` classifies it:

{% mermaid %}
flowchart TD
    A[Tool fails] --> B[skill-gap-detector.sh]
    B --> C[Match error patterns]
    B --> D[Assign domain]
    B --> E[Assign subdomain]
    B --> F[Generate hints]
    C & D & E & F --> G[tool_failure event]
{% endmermaid %}

**Example event:**
```json
{
  "event_type": "tool_failure",
  "payload": {
    "domain": "state",
    "subdomain": "file-not-found",
    "hints": ["Check path exists"]
  }
}
```

See [Friction Taxonomy](/friction-taxonomy/) for the complete classification system.

## Large Change Detection

Changes exceeding 250 lines trigger special handling:

{% mermaid %}
flowchart TD
    A[Edit file] --> B[git diff --shortstat]
    B --> C{lines > 250?}
    C -->|No| D[no action]
    C -->|Yes| E[large_change event]
    E --> F[System prompts for tradeoff documentation]
    F --> G[decision_tradeoff event]
{% endmermaid %}

## Test Tracking

Test runs are captured automatically:

{% mermaid %}
flowchart TD
    A[Edit .rb/.ts/.js] --> B[async-test-runner.sh]
    B --> C[Run tests in background]
    C --> D[test_run event]
{% endmermaid %}

**Example event:**
```json
{
  "event_type": "test_run",
  "payload": {
    "passed": true,
    "tests": ["spec/..."]
  }
}
```

## Session Lifecycle

### Session Start

{% mermaid %}
flowchart TD
    A[Claude starts] --> B[session-context-injector.sh]
    A --> C[friction-escalator.sh]
    B --> D[Inject recent impact/friction]
    C --> E[Warn if repeated errors]
{% endmermaid %}

### During Work

{% mermaid %}
flowchart LR
    A[Edit file] --> B[impact-extractor.sh]
    A --> C[large-diff-escalator.sh]
    A --> D[reversal-detector.sh]
    B --> E[Log impact]
    C --> F[Flag if large]
    D --> G[Detect undos]
{% endmermaid %}

### Session End

{% mermaid %}
flowchart TD
    A[Stop requested] --> B{Tests passing?}
    B -->|No| C[Block stop]
    B -->|Yes| D{Large changes documented?}
    D -->|No| E[Prompt for tradeoffs]
    D -->|Yes| F[Allow stop]
{% endmermaid %}

## Storage Layout

All captured data lands in `~/.claude/`:

{% mermaid %}
flowchart TB
    subgraph claude["~/.claude/"]
        A[(dev-os-events.jsonl<br/>All events)]
        B[(impact-log.jsonl<br/>File changes)]
        C[(skill-friction-log.jsonl<br/>Errors)]
        D[idea-vault.md<br/>Captured opinions]
        E[learning-targets/<br/>Study guides]
        F[session-summaries/<br/>Snapshots]
        G[reviews/<br/>Weekly reports]
    end
{% endmermaid %}

## What Gets Captured

| You Do This | Event Created | Shows Up In |
|-------------|---------------|-------------|
| Edit file | tool_write | Writes count |
| Big edit (>250 lines) | large_change | Discipline flags |
| Update deps | dependency_change | Churn tracking |
| Delete lots of code | reversal | Reversal count |
| Tool fails | tool_failure | Friction analysis |
| Run tests | test_run | Test stability |
| Complete task | task_completed | Tasks count |
| Explain tradeoff | decision_tradeoff | Principles invoked |

---

Next: [Friction Taxonomy](/friction-taxonomy/) | [Review Pipeline](/review-pipeline/)
