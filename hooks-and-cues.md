---
layout: page
title: Hooks & Cues
permalink: /hooks-and-cues/
---

# Hooks & Cues

Two systems that work together to provide engineering telemetry and context-aware guidance.

## Overview

| System | Purpose | When it runs |
|--------|---------|--------------|
| **Hooks** | Observe and react to events | Automatically on Claude Code events |
| **Cues** | Inject contextual guidance | When triggers match prompts/commands/files |

## Hooks: Event-Driven Telemetry

Hooks are shell scripts that run in response to Claude Code events.

### Hook Events

| Event | When | Common Uses |
|-------|------|-------------|
| `SessionStart` | Session begins/resumes | Context injection, marker clearing |
| `UserPromptSubmit` | User sends prompt | Cue matching |
| `PreToolUse` | Before tool executes | Validation, cue injection |
| `PostToolUse` | After tool succeeds | Impact logging, pattern detection |
| `PostToolUseFailure` | After tool fails | Friction classification |
| `Stop` | Session ending | Enforcement gates |

### Example: Impact Extractor

```bash
#!/usr/bin/env bash
set -euo pipefail

# Read event payload
INPUT=$(cat)
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

if [[ -n "$FILE_PATH" ]]; then
  # Emit structured event
  echo "$INPUT" | dev-os-emit.sh "tool_write" "{\"files\": [\"$FILE_PATH\"]}"
fi
```

### Hook Output

Different events expect different output formats:

| Event | Output Format | Effect |
|-------|---------------|--------|
| PreToolUse | `{"permissionDecision": "allow"}` | Allow/block tool |
| PostToolUse | `{"hookSpecificOutput": {"context": "..."}}` | Inject into context |
| Stop | `{"ok": false, "reason": "..."}` | Block session end |

## Cues: Context-Aware Guidance

Cues inject guidance when triggers match. They're "compiled policy"—governance documents compressed into agent directives.

### Cue Format

```yaml
---
# Trigger matching (regex)
pattern: commit|push|amend           # Match user prompts
commands: git\s+(commit|push)        # Match bash commands
files: \.env$|\.env\.local$          # Match file paths

# Scope control
scope: agent                         # agent | subagent | agent, subagent

# Semantic matching (fallback)
description: Git commit workflow and version control
vocabulary: commit push amend rebase merge
---

# Commit Cue

- Use conventional commit format (feat:, fix:, refactor:)
- Sign commits with GPG
- Keep commits atomic and focused
```

### Matching Priority

1. **Regex match** - `pattern:`, `commands:`, or `files:` fields
2. **Vocabulary match** - Any word in `vocabulary:` appears in query
3. **Semantic match** - Gzip NCD similarity to `description:`

### Example Triggers

**Prompt trigger:**
```yaml
pattern: commit|push|amend
```
User types "commit my changes" → cue fires

**Command trigger:**
```yaml
commands: git\s+(commit|push|rebase)
```
Claude runs `git commit -m "..."` → cue fires

**File trigger:**
```yaml
files: \.env$|\.env\.local$
```
Claude edits `.env.local` → cue fires

### Scope Filtering

| Scope | Fires For | Use When |
|-------|-----------|----------|
| `agent` | Main agent only | Default; most cues |
| `subagent` | Spawned subagents only | Subagent-specific guidance |
| `agent, subagent` | Both contexts | Universal guidance |

### Once-Per-Session Gating

Each cue fires at most once per session to avoid repetition:

{% mermaid %}
flowchart LR
    A[First 'commit' prompt] --> B[cue fires]
    C[Second 'commit' prompt] --> D[cue skipped<br/>marker exists]
    E[New session starts] --> F[markers cleared]
{% endmermaid %}

## How They Work Together

{% mermaid %}
flowchart TD
    subgraph lifecycle["Session Lifecycle"]
        A[SessionStart] --> A1[Clear cue markers]
        A --> A2[Inject context]
        A --> A3[Escalate friction]

        A --> B[UserPromptSubmit]
        B --> B1[Match cues]
        B1 --> B2[Inject matching cue content]

        B --> C[PreToolUse]
        C --> C1[Match command cues]
        C --> C2[Match file cues]
        C1 & C2 --> C3[Inject matching cue content]

        C --> D[PostToolUse]
        D --> D1[Extract impact]
        D --> D2[Detect patterns]
        D --> D3[Check for reversal]
        D1 --> D4[(dev-os-events.jsonl)]

        D --> E[Stop]
        E --> E1{Tests passing?}
        E1 -->|No| E2[Block]
        E1 -->|Yes| E3{Tradeoffs documented?}
        E3 -->|No| E4[Prompt]
        E3 -->|Yes| E5[Allow stop]
    end
{% endmermaid %}

## Cue Engagement Tracking

When a cue fires, an event is logged:

```json
{
  "event_type": "cue_fired",
  "payload": {
    "cue_id": "commit",
    "trigger_type": "prompt",
    "has_macro": false
  }
}
```

The weekly review aggregates this to show:
- Which cues are actively providing guidance
- Trigger patterns (prompt vs bash vs file)
- Dormant cues that may need better triggers

## Creating Hooks

### Basic Structure

```bash
#!/usr/bin/env bash
set -euo pipefail

# Read event payload from stdin
INPUT=$(cat)

# Extract fields
TOOL_NAME=$(echo "$INPUT" | jq -r '.tool_name // empty')
FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty')

# Your logic
if [[ -n "$FILE_PATH" ]]; then
  # Do something
fi

# Optional output
echo '{"hookSpecificOutput": {"context": "Your message here"}}'
```

### Configuration

Add to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "~/.claude/hooks/common/PostToolUse/my-hook.sh"
      }
    ]
  }
}
```

## Creating Cues

### Basic Cue

```yaml
---
pattern: docker|container
scope: agent
description: Docker and containerization workflows
vocabulary: docker container dockerfile compose kubernetes
---

# Docker Cue

- Use multi-stage builds to reduce image size
- Never store secrets in Dockerfiles
- Pin base image versions for reproducibility
```

### Cue with Macro

Macros add dynamic content:

```yaml
---
files: Gemfile$
scope: agent
macro: prepend
---

# Ruby Dependencies

- Run `bundle install` after Gemfile changes
```

```bash
# cues/ruby-deps/macro.sh
#!/usr/bin/env bash
if [[ -f "Gemfile.lock" ]]; then
  ruby_version=$(grep -A1 "RUBY VERSION" Gemfile.lock | tail -1)
  echo "**Ruby version**: $ruby_version"
fi
```

## Testing

### Test Hook

```bash
echo '{"tool_name": "Write", "tool_input": {"file_path": "test.rb"}}' | \
  ~/.claude/hooks/common/PostToolUse/my-hook.sh
```

### Test Cue Matching

```bash
~/.claude/hooks/common/match-cues.sh prompt "commit my changes"
~/.claude/hooks/common/match-cues.sh bash "git push origin main"
~/.claude/hooks/common/match-cues.sh file ".env.local"
```

### Check Hook Health

```bash
~/.claude/hooks/common/hook-health.sh           # 24h summary
~/.claude/hooks/common/hook-health.sh --recent  # Last 10 runs
~/.claude/hooks/common/hook-health.sh --failures
```

---

Previous: [Review Pipeline](/review-pipeline/) | Next: [Event Schema](/event-schema/)
