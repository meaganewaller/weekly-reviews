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
        subgraph start["SessionStart"]
            P6[friction escalator]
            P7[context injector]
        end
        subgraph prompt["UserPromptSubmit"]
            P10[cue-injector-prompt]
            P11[session-duration-monitor]
            P12[idea-classifier]
        end
        subgraph pre["PreToolUse"]
            P1[cue inject]
            P13[principle-reinforcer]
            P14[large-file-guard]
        end
        subgraph post["PostToolUse"]
            P2[impact extractor]
            P3[large-diff]
            P4[reversal]
            P15[principle-activator]
        end
        subgraph fail["PostToolUseFailure"]
            P5[skill-gap detector]
        end
        subgraph compact["PreCompact"]
            P16[pre-compact-snapshot]
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
| `cue-injector-*.sh` | PreToolUse, UserPromptSubmit | Matching cue guidance |
| `principle-reinforcer.sh` | PreToolUse | Active principles before writes |

### Session & Principle Hooks

Track session health and principle application.

| Hook | Trigger | What it captures |
|------|---------|------------------|
| `session-duration-monitor.sh` | UserPromptSubmit | Duration, archetype, guidance |
| `pre-compact-snapshot.sh` | PreCompact | Session health metrics |
| `principle-activator.sh` | PostToolUse | First principle invocations |
| `idea-classifier.sh` | UserPromptSubmit | Domain modeling prompts |

### Guard Hooks

Block or warn about risky operations.

| Hook | Trigger | What it guards |
|------|---------|----------------|
| `large-file-guard.sh` | PreToolUse | Session logs (blocks), large files (warns) |

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

## Session Duration Tracking

Sessions are monitored for duration and classified into archetypes:

{% mermaid %}
flowchart TD
    A[User submits prompt] --> B[session-duration-monitor.sh]
    B --> C{Duration?}
    C -->|<30 min| D[Sprint archetype]
    C -->|30-120 min| E[Flow archetype]
    C -->|>120 min| F[Marathon archetype]
    D & E & F --> G[session_duration event]
    F --> H[Suggest break/commit]
{% endmermaid %}

**Example event:**
```json
{
  "event_type": "session_duration",
  "payload": {
    "duration_minutes": 145,
    "archetype": "marathon",
    "has_task_list": true
  }
}
```

## Principle Activation

Principles are tracked through activation (first mention) and reinforcement (re-surfacing):

{% mermaid %}
flowchart TD
    A[Output mentions 'model-first'] --> B[principle-activator.sh]
    B --> C[Mark principle active]
    C --> D[principle_activated event]
    E[Later: Write to models/] --> F[principle-reinforcer.sh]
    F --> G{Active principle relevant?}
    G -->|Yes| H[Inject reminder]
    H --> I[principle_reinforced event]
{% endmermaid %}

**Example activation:**
```json
{
  "event_type": "principle_activated",
  "payload": {
    "principle": "model-first",
    "activation": "first_invocation"
  }
}
```

## Domain Modeling Detection

Prompts indicating upfront design work are tracked:

{% mermaid %}
flowchart TD
    A[User asks 'what entities do we need?'] --> B[idea-classifier.sh]
    B --> C[Pattern match: domain modeling]
    C --> D[domain_modeling event]
{% endmermaid %}

Tracked to correlate modeling frequency with reversal rates.

## Session Lifecycle

### Session Start

{% mermaid %}
flowchart TD
    A[Claude starts] --> B[session-context-injector.sh]
    A --> C[friction-escalator.sh]
    A --> D[Clear cue markers]
    B --> E[Inject recent impact/friction]
    C --> F[Warn if repeated errors]
{% endmermaid %}

### On Each Prompt

{% mermaid %}
flowchart TD
    A[User submits prompt] --> B[cue-injector-prompt.sh]
    A --> C[session-duration-monitor.sh]
    A --> D[idea-classifier.sh]
    B --> E[Inject matching cues]
    C --> F[Track duration/archetype]
    D --> G[Detect domain modeling]
{% endmermaid %}

### During Work

{% mermaid %}
flowchart LR
    A[Edit file] --> B[impact-extractor.sh]
    A --> C[large-diff-escalator.sh]
    A --> D[reversal-detector.sh]
    A --> E[principle-activator.sh]
    B --> F[Log impact]
    C --> G[Flag if large]
    D --> H[Detect undos]
    E --> I[Track principles]
{% endmermaid %}

### Before Compaction

{% mermaid %}
flowchart TD
    A[Context pressure] --> B[pre-compact-snapshot.sh]
    B --> C[Capture session health]
    C --> D[session_health event]
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
| Work for 30+ minutes | session_duration | Archetype distribution |
| Hit context limit | session_health | Session health score |
| Mention a principle | principle_activated | Principles invoked |
| Write after principle | principle_reinforced | Reinforcement depth |
| Ask about entities | domain_modeling | Modeling ratio |

---

Next: [Friction Taxonomy](/friction-taxonomy/) | [Review Pipeline](/review-pipeline/)
