---
layout: glossary-term
term: Session
short: Single Claude Code interaction
category: telemetry
related:
  - telemetry
  - event
  - escalation
aliases:
  - sessions
  - claude code session
---

A single Claude Code interaction, identified by a unique session_id. Sessions are the unit of work for telemetry.

## Session ID

Each session has a unique identifier:

```json
{
  "session_id": "abc123",
  "event_type": "tool_write",
  ...
}
```

## Lifecycle

### Start

Triggers `SessionStart` hooks:
- Clear cue markers
- Inject context from previous sessions
- Escalate repeated friction patterns

### During

Events are logged with the session_id:
- `tool_write` - File changes
- `tool_failure` - Errors
- `test_run` - Test execution

### End

Triggers `Stop` hooks:
- Check test status
- Verify tradeoff documentation
- Allow or block session end

## Session Gating

Cues fire **at most once per session**:

```
First "commit" prompt  → cue fires
Second "commit" prompt → cue skipped
New session starts     → markers cleared
```

## In Weekly Reviews

Sessions appear in:
- Total session count
- Per-project session breakdown
- Events-per-session metrics
