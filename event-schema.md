---
layout: page
title: Event Schema
permalink: /event-schema/
---

# Event Schema

Structure of events captured by the Dev OS telemetry system.

## Primary Event Log

All events are appended to `~/.claude/dev-os-events.jsonl` in JSON Lines format (one JSON object per line).

### Base Structure

Every event contains:

```json
{
  "timestamp": "2026-02-27T10:30:00Z",
  "session_id": "abc123",
  "event_type": "tool_write",
  "payload": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `timestamp` | ISO 8601 | When the event occurred |
| `session_id` | string | Unique identifier for the Claude Code session |
| `event_type` | string | Type of event (see below) |
| `payload` | object | Event-specific data |

## Event Types

### tool_write

Emitted when a file is created or modified.

```json
{
  "event_type": "tool_write",
  "payload": {
    "files": ["/path/to/file.rb"],
    "change_type": "refactor",
    "risk_level": "low",
    "skill_domains": ["application development"]
  }
}
```

| Payload Field | Type | Values |
|---------------|------|--------|
| `files` | string[] | Paths to files written |
| `change_type` | string | `feature`, `fix`, `refactor`, `test`, `infra`, `docs` |
| `risk_level` | string | `low`, `medium`, `high` |
| `skill_domains` | string[] | Classification of work type |

### tool_failure

Emitted when a tool execution fails.

```json
{
  "event_type": "tool_failure",
  "payload": {
    "tool_name": "Read",
    "domain": "state",
    "subdomain": "file-not-found",
    "hints": ["Check if file exists before reading"],
    "error_snippet": "No such file: /path/to/missing.rb"
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `tool_name` | string | Which tool failed |
| `domain` | string | Top-level friction category |
| `subdomain` | string | Specific friction type |
| `hints` | string[] | Suggestions for resolution |
| `error_snippet` | string | First 200 chars of error |

See [Friction Taxonomy](/friction-taxonomy/) for domain/subdomain hierarchy.

### large_change

Emitted when a diff exceeds 250 lines.

```json
{
  "event_type": "large_change",
  "payload": {
    "file_path": "/path/to/large_file.rb",
    "lines_added": 180,
    "lines_removed": 95,
    "total_lines": 275
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `file_path` | string | File with large change |
| `lines_added` | number | Lines added |
| `lines_removed` | number | Lines removed |
| `total_lines` | number | Total lines changed |

### reversal

Emitted when recent work is undone.

```json
{
  "event_type": "reversal",
  "payload": {
    "reverted_file": "/path/to/file.rb",
    "lines_removed": 75,
    "lines_added": 10,
    "reason": "Exploration rolled back"
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `reverted_file` | string | File being reverted |
| `lines_removed` | number | Lines deleted |
| `lines_added` | number | Lines added (typically small) |
| `reason` | string | Optional explanation |

### decision_tradeoff

Emitted when architectural decisions are documented.

```json
{
  "event_type": "decision_tradeoff",
  "payload": {
    "summary": "Chose jq over Python for JSON processing",
    "options": ["jq", "Python", "Node.js"],
    "tradeoffs": [
      "jq: shell-native, no dependencies, but limited logic",
      "Python: powerful, but adds dependency"
    ],
    "principles": ["convention over configuration", "explicit over implicit"]
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `summary` | string | Brief description of decision |
| `options` | string[] | Alternatives considered |
| `tradeoffs` | string[] | Pros/cons of each option |
| `principles` | string[] | Architectural principles invoked |

### test_run

Emitted when tests are executed.

```json
{
  "event_type": "test_run",
  "payload": {
    "passed": true,
    "tests_run": 47,
    "tests_passed": 47,
    "tests_failed": 0,
    "duration_seconds": 12.5,
    "test_files": ["spec/models/user_spec.rb"]
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `passed` | boolean | Overall pass/fail |
| `tests_run` | number | Total tests executed |
| `tests_passed` | number | Tests that passed |
| `tests_failed` | number | Tests that failed |
| `duration_seconds` | number | Execution time |
| `test_files` | string[] | Files tested |

### task_completed

Emitted when a task is marked complete.

```json
{
  "event_type": "task_completed",
  "payload": {
    "task_id": "task-123",
    "task_subject": "Implement user authentication",
    "validation_checks": {
      "tests_passing": true,
      "linting_clean": true
    }
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `task_id` | string | Task identifier |
| `task_subject` | string | Task description |
| `validation_checks` | object | Gate validation results |

### dependency_change

Emitted when dependency files are modified.

```json
{
  "event_type": "dependency_change",
  "payload": {
    "file": "Gemfile",
    "added": ["rails", "pg"],
    "removed": ["sqlite3"],
    "updated": []
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `file` | string | Dependency file changed |
| `added` | string[] | New dependencies |
| `removed` | string[] | Removed dependencies |
| `updated` | string[] | Updated dependencies |

### cue_fired

Emitted when a cue is injected.

```json
{
  "event_type": "cue_fired",
  "payload": {
    "cue_id": "commit",
    "trigger_type": "prompt",
    "has_macro": false,
    "query": "commit my changes"
  }
}
```

| Payload Field | Type | Description |
|---------------|------|-------------|
| `cue_id` | string | Cue identifier |
| `trigger_type` | string | `prompt`, `bash`, or `file` |
| `has_macro` | boolean | Whether macro was executed |
| `query` | string | Text that triggered the cue |

## Supporting Log Files

### impact-log.jsonl

Detailed file change tracking:

```json
{
  "timestamp": "2026-02-27T10:30:00Z",
  "file_paths": ["/path/to/file.rb"],
  "change_type": "refactor",
  "skill_domains": ["application development"],
  "risk_level": "low"
}
```

### skill-friction-log.jsonl

Error classification for pattern detection:

```json
{
  "timestamp": "2026-02-27T10:30:00Z",
  "tool_name": "Read",
  "domain": "state",
  "subdomain": "file-not-found",
  "hints": ["Check if file exists"],
  "error_snippet": "No such file..."
}
```

## Querying Events

### Recent events

```bash
tail -20 ~/.claude/dev-os-events.jsonl | jq .
```

### Filter by type

```bash
jq 'select(.event_type == "tool_failure")' ~/.claude/dev-os-events.jsonl
```

### Count by type

```bash
jq -s 'group_by(.event_type) | map({type: .[0].event_type, count: length})' \
  ~/.claude/dev-os-events.jsonl
```

### Last 7 days

```bash
SINCE=$(date -v-7d +%Y-%m-%d)
jq -s --arg since "$SINCE" '[.[] | select(.timestamp >= $since)]' \
  ~/.claude/dev-os-events.jsonl
```

### Friction by domain

```bash
jq -s '[.[] | select(.event_type == "tool_failure")]
  | group_by(.payload.domain)
  | map({domain: .[0].payload.domain, count: length})' \
  ~/.claude/dev-os-events.jsonl
```

## Log Rotation

Prevent unbounded growth:

```bash
# Keep last 10000 events
tail -10000 ~/.claude/dev-os-events.jsonl > /tmp/events.tmp
mv /tmp/events.tmp ~/.claude/dev-os-events.jsonl
```

Consider archiving before rotation:

```bash
# Archive then rotate
cp ~/.claude/dev-os-events.jsonl ~/.claude/archive/events-$(date +%Y%m%d).jsonl
tail -10000 ~/.claude/dev-os-events.jsonl > /tmp/events.tmp
mv /tmp/events.tmp ~/.claude/dev-os-events.jsonl
```

---

Previous: [Hooks & Cues](/hooks-and-cues/) | [Back to How It Works](/how-it-works/)
