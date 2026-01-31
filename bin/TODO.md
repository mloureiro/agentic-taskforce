# Taskforce Helper Executables - TODO

This document describes helper executables to be implemented for taskforce operations.

---

## 1. `tf-sleep` - Smart Sleep with Exponential Backoff ✅ IMPLEMENTED

**Purpose:** Sleep with automatic stage tracking, following the exponential backoff pattern.

### Usage

```bash
# Default: sleep 5s (stage 0)
tf-sleep

# Escalate if recently slept, otherwise 5s
tf-sleep --continue

# Sleep specified time, continue from that stage
tf-sleep --from 60

# Wait for new chat messages (polls every 5s, timeout 10min)
tf-sleep --until-new-messages
tf-sleep -u --max 120  # Custom timeout

# Preview without sleeping
tf-sleep --dry-run
```

### Behavior

- **Backoff sequence:** 5s → 10s → 30s → 60s → 90s → 120s → 180s (max)
- **Default:** Always starts at stage 0 (5s)
- **With `--continue`:** Escalates if within timing window, otherwise stays at stage 0
- **With `-u`:** Polls for unread messages, exits when found or on timeout

### State Storage

Store stage state in: `~/.claude-taskforce/bin/.sleep.log`

### Output

```
[tf-sleep] red | fix-flaky-ci
Stage 3 | Sleeping 1m 0s | escalate (called 15s after last, within 70s window)
[tf-sleep] Done. Next sleep: 1m 30s
```

---

## 2. `tf-log` - Chat Log Utilities

**Purpose:** Interact with the taskforce chat log (add entries, search, check updates).

### Subcommands

#### `tf-log add` - Add Entry to Chat Log

```bash
# Add a progress entry
tf-log add --task fix-flaky-ci --agent red --type PROGRESS --message "Found potential cause in fixtures.ts"

# Add with multiline message (reads from stdin)
echo "Line 1
Line 2" | tf-log add --task fix-flaky-ci --agent red --type FINDING

# Add with message from file
tf-log add --task fix-flaky-ci --agent red --type PROGRESS --file /tmp/message.txt
```

**Output format (appended to chat-common.log):**
```
### [2024-01-28 15:30:45] red | PROGRESS
Found potential cause in fixtures.ts

```

**Valid types:** JOIN, PROGRESS, QUESTION, ANSWER, FINDING, DECISION, BLOCKER, WAITING, DONE, ESCALATE

---

#### `tf-log search` - Search Chat Log

```bash
# Search for keyword
tf-log search --task fix-flaky-ci --query "transaction"

# Search by agent
tf-log search --task fix-flaky-ci --agent blue

# Search by type
tf-log search --task fix-flaky-ci --type FINDING

# Combine filters
tf-log search --task fix-flaky-ci --agent red --type QUESTION --query "parallel"

# Limit results
tf-log search --task fix-flaky-ci --query "fix" --limit 5
```

**Output:**
```
[2024-01-28 14:35:00] red | PROGRESS
  ...transaction commit race condition...

[2024-01-28 14:50:00] yellow | FINDING
  ...transaction not awaited...

Found 2 matches.
```

---

#### `tf-log last` - Check Last Update

```bash
# Get last update from any agent
tf-log last --task fix-flaky-ci

# Get last update from specific agent
tf-log last --task fix-flaky-ci --agent blue

# Smart mode: tracks caller and reports "no news" if nothing new since last check
tf-log last --task fix-flaky-ci --agent red --smart
```

**Regular output:**
```
Last update: 2024-01-28 15:45:30
Agent: blue
Type: PROGRESS
Message preview: CI running, 8/12 jobs complete...

Time since: 3m 24s ago
```

**Smart mode output (with caching):**

First call:
```
Last update: 2024-01-28 15:45:30
Agent: blue
Type: PROGRESS
Message preview: CI running, 8/12 jobs complete...
```

Subsequent call (no new entries):
```
No news since last check (3m 24s ago).
Last seen: 2024-01-28 15:45:30 | blue | PROGRESS
```

Subsequent call (new entry found):
```
NEW UPDATE since last check!
[2024-01-28 15:48:00] blue | FINDING
Found the root cause...

Time since your last check: 2m 30s
```

**Smart mode state storage:** `~/.claude-taskforce/{{task-name}}/.last-check-{{agent}}`

---

#### `tf-log summary` - Get Chat Summary

```bash
# Get summary of chat activity
tf-log summary --task fix-flaky-ci
```

**Output:**
```
Task: fix-flaky-ci
Total entries: 24
Active agents: red, blue, yellow

Last activity: 2024-01-28 15:48:00 (5m ago)

By agent:
  red:    8 entries | last: 15:42:00 | status: WAITING
  blue:   10 entries | last: 15:48:00 | status: FINDING
  yellow: 6 entries | last: 15:30:00 | status: PROGRESS

Recent entries:
  [15:48:00] blue | FINDING - Found the root cause...
  [15:45:30] blue | PROGRESS - CI running, 8/12 jobs...
  [15:42:00] red | WAITING - Pushed fix, CI running...
```

---

## 3. `tf-init` - Initialize New Task

**Purpose:** Create a new task folder with all required files.

### Usage

```bash
# Initialize a new task
tf-init --name fix-flaky-ci

# With optional description
tf-init --name fix-flaky-ci --description "Fix the flaky CI tests in parallel runs"
```

### Behavior

Creates:
```
~/.claude-taskforce/fix-flaky-ci/
├── TASK.md           # Template with placeholders
└── chat-common.log   # Empty, ready for entries
```

**TASK.md template:**
```markdown
# Task: fix-flaky-ci

## Objective
{{description or placeholder}}

## Requirements Checklist
- [ ]

## Context


## Relevant Files


## Constraints


## Success Criteria


## Notes

```

---

## 4. `tf-status` - Taskforce Status Dashboard

**Purpose:** Quick overview of taskforce state.

### Usage

```bash
# Status for a specific task
tf-status --task fix-flaky-ci

# Status for all active tasks
tf-status --all
```

### Output

```
═══════════════════════════════════════════════════════════
TASKFORCE STATUS: fix-flaky-ci
═══════════════════════════════════════════════════════════

Agents:
  red     WAITING   last: 5m ago    branch: mloureiro/fix-flaky-ci/red
  blue    EXPLORING last: 2m ago    branch: mloureiro/fix-flaky-ci/blue
  yellow  DONE      last: 15m ago   branch: mloureiro/fix-flaky-ci/yellow

Requirements: 2/4 complete
  [x] Identify root cause
  [x] Implement fix
  [ ] CI passes 3x consecutive
  [ ] No performance regression

Recent Activity:
  [15:48] blue: Found potential issue in transaction handling
  [15:45] red: CI still running, waiting...
  [15:30] yellow: DONE - My approach validated

Alerts:
  ! yellow has been silent for 15m (last status: DONE)
```

---

## Implementation Notes

### Language

Recommend: **Bash** or **Python**
- Bash: Simpler, no dependencies
- Python: Better for complex parsing, state management

### State Files

All state files should be hidden (dot-prefixed) and stored in the task folder:
- `~/.claude-taskforce/{{task-name}}/.sleep-state-{{agent}}`
- `~/.claude-taskforce/{{task-name}}/.last-check-{{agent}}`

### Error Handling

- Validate task exists before operations
- Validate agent name is from allowed list
- Create missing directories/files as needed
- Clear error messages with usage hints

### Testing

Each executable should support:
```bash
tf-sleep --help
tf-log --help
tf-init --help
tf-status --help
```

---

## Priority Order

1. **`tf-log add`** - Most frequently used
2. **`tf-log last --smart`** - Critical for polling
3. **`tf-sleep`** - Standardizes backoff
4. **`tf-init`** - Convenience for setup
5. **`tf-log search`** - Useful but less critical
6. **`tf-log summary`** - Nice to have
7. **`tf-status`** - Dashboard, nice to have
