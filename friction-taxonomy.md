---
layout: page
title: Friction Taxonomy
permalink: /friction-taxonomy/
---

# Friction Taxonomy

How tool failures are classified for pattern detection.

## Overview

When a tool fails during a Claude Code session, the `skill-gap-detector.sh` hook classifies the error into a hierarchical taxonomy. This classification enables:

- **Pattern detection** across sessions
- **Skill gap identification** over time
- **Deliberate practice recommendations**

## Primary Domains

{% mermaid %}
flowchart TB
    F[Friction] --> syntax[syntax<br/>Parse errors]
    F --> type[type<br/>Type mismatches]
    F --> dependency[dependency<br/>Missing packages]
    F --> permission[permission<br/>Access denied]
    F --> network[network<br/>Connection errors]
    F --> state[state<br/>File not found]
    F --> config[config<br/>Env vars]
    F --> testing[testing<br/>Test failures]
    F --> build[build<br/>Compilation errors]
{% endmermaid %}

## Domain Details

### State

The most common friction domain. Occurs when operations encounter unexpected system state.

{% mermaid %}
flowchart LR
    A[state] --> B[file-not-found]
    A --> C[resource-limit]
    A --> D[conflict]
    A --> E[command-failed]
    A --> F[type-mismatch]
{% endmermaid %}

**Common causes:**
- Working with newly created files before they exist
- Operations on large files without pagination
- Concurrent access to shared resources

**Practice recommendations:**
- Add existence checks before file operations
- Use chunked reads for files > 1000 lines
- Implement retry logic with backoff

### Syntax

Parse errors from malformed input.

{% mermaid %}
flowchart LR
    A[syntax] --> B[parse]
    A --> C[encoding]
    A --> D[format]
{% endmermaid %}

**Common causes:**
- Invalid JSON in tool arguments
- Mixed tabs/spaces in YAML
- Unclosed brackets in code

**Practice recommendations:**
- Validate JSON/YAML before writing
- Use linters during editing
- Double-check multiline strings

### Type

Type system complaints.

{% mermaid %}
flowchart LR
    A[type] --> B[ruby-sorbet]
    A --> C[typescript]
    A --> D[rust-borrow]
    A --> E[inference]
{% endmermaid %}

**Common causes:**
- Nil/null in unexpected places
- Interface mismatches
- Generic type confusion

**Practice recommendations:**
- Add type annotations at boundaries
- Use strict null checks
- Write types before implementation

### Dependency

Package and module resolution failures.

{% mermaid %}
flowchart LR
    A[dependency] --> B[ruby-bundler]
    A --> C[node-npm]
    A --> D[rust-cargo]
    A --> E[python-module]
    A --> F[version]
{% endmermaid %}

**Common causes:**
- Missing bundle install after Gemfile changes
- Conflicting version requirements
- Corrupted lock files

**Practice recommendations:**
- Run install commands after dependency changes
- Keep lock files in sync
- Use exact versions for critical deps

### Permission

Access control failures.

{% mermaid %}
flowchart LR
    A[permission] --> B[auth]
    A --> C[file]
    A --> D[network]
    A --> E[process]
{% endmermaid %}

**Common causes:**
- Expired tokens
- Wrong file ownership
- Missing sudo

**Practice recommendations:**
- Check credentials before operations
- Use least-privilege principles
- Verify file permissions

### Network

Connection and communication failures.

{% mermaid %}
flowchart LR
    A[network] --> B[timeout]
    A --> C[connection]
    A --> D[ssl]
    A --> E[dns]
{% endmermaid %}

**Common causes:**
- Slow services
- VPN disconnects
- Expired certificates

**Practice recommendations:**
- Add timeout handling
- Implement retry with exponential backoff
- Verify connectivity before operations

### Config

Environment and configuration issues.

{% mermaid %}
flowchart LR
    A[config] --> B[env-var]
    A --> C[file]
    A --> D[path]
    A --> E[invalid]
{% endmermaid %}

**Common causes:**
- Missing .env files
- Wrong environment active
- Outdated config after changes

**Practice recommendations:**
- Document required env vars
- Use config validation at startup
- Provide sensible defaults

### Testing

Test execution failures.

{% mermaid %}
flowchart LR
    A[testing] --> B[assertion]
    A --> C[setup]
    A --> D[timeout]
    A --> E[flaky]
{% endmermaid %}

**Common causes:**
- Actual bugs in code
- Stale fixtures
- Race conditions

**Practice recommendations:**
- Fix tests immediately
- Use deterministic fixtures
- Isolate flaky tests

### Build

Compilation and bundling failures.

{% mermaid %}
flowchart LR
    A[build] --> B[compile]
    A --> C[bundle]
    A --> D[link]
    A --> E[target]
{% endmermaid %}

**Common causes:**
- Syntax errors preventing compilation
- Missing build dependencies
- Platform-specific code issues

**Practice recommendations:**
- Run builds frequently
- Keep build tools updated
- Use CI to catch cross-platform issues

## Pattern Detection

The weekly review aggregates friction by domain and subdomain to identify patterns:

```
### By Domain
- state: 511
- unknown: 48
- dependency: 3

### By Subdomain
- state:file-not-found: 232
- state:resource-limit: 226
- state:command-failed: 41
```

High concentrations in a domain indicate skill gaps that warrant deliberate practice.

## Escalation

When the same friction domain appears repeatedly (3+ times), `friction-escalator.sh` surfaces a warning at the start of your next session:

```
SessionStart hook additional context:
Repeated friction in [state]: 20 hits.
Subdomains: file-not-found(10) resource-limit(7) command-failed(3)
Hint: Command returned non-zero exit code; check if target exists...
```

This creates a feedback loop for improvement.

## Classification Logic

The `skill-gap-detector.sh` hook uses pattern matching on error messages:

```bash
# Example classification rules
if echo "$error" | grep -qE "No such file|not found|does not exist"; then
  DOMAIN="state"
  SUBDOMAIN="file-not-found"
elif echo "$error" | grep -qE "too large|limit exceeded|timeout"; then
  DOMAIN="state"
  SUBDOMAIN="resource-limit"
elif echo "$error" | grep -qE "permission denied|access denied"; then
  DOMAIN="permission"
  SUBDOMAIN="file"
fi
```

Unrecognized errors default to `domain: unknown`.

## Adding Custom Patterns

To improve classification accuracy, extend the pattern matching in `skill-gap-detector.sh`:

```bash
# Add before the fallback case
elif echo "$error" | grep -qE "your specific pattern"; then
  DOMAIN="appropriate_domain"
  SUBDOMAIN="specific_subdomain"
fi
```

---

Previous: [Data Capture](/data-capture/) | Next: [Review Pipeline](/review-pipeline/)
