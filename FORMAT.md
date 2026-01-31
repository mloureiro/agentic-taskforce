# Taskforce Format Guide

This document defines file formats, naming conventions, and communication structure.

---

## Directory Structure

```
$TF_ROOT/
├── RULES.md                          # Core directives (this repo)
├── FORMAT.md                         # This file
├── PATTERNS.md                       # Taskforce patterns
├── LIFECYCLE.md                      # Agent lifecycle & states
│
└── {{task-name}}/                    # Per-task folder
    ├── TASK.md                       # Mission briefing & requirements
    ├── chat-common.log               # All agent communication (append-only)
    ├── findings-{{agent-name}}.md    # Personal findings (one per agent)
    └── ...                           # Other task-specific files
```

### Task Folder Naming

Choose a descriptive name, ideally including story number if available:
- `sc-12345-fix-flaky-tests`
- `fix-deadlock-parallel-runs`
- `feature-new-export`

---

## Chat Log Format

**File:** `chat-common.log`
**Access:** Append-only

### Entry Structure

```
### [YYYY-MM-DD HH:MM:SS] {{agent-name}} | {{TYPE}}
{{content}}

```

**Important:** Always include a blank line after each entry.

### Message Types

| Type | Purpose | When to Use |
|------|---------|-------------|
| `JOIN` | Announce joining taskforce | First message from agent |
| `PROGRESS` | Status update | Every 3-5 mins while working |
| `QUESTION` | Ask other agents | Need input from others |
| `ANSWER` | Respond to question | Replying to a `QUESTION` |
| `FINDING` | Important discovery | Found something significant |
| `DECISION` | Announce a choice | Making a choice that affects others |
| `BLOCKER` | Report being stuck | Hit an obstacle |
| `WAITING` | Entering wait state | Starting a long wait (CI, etc.) |
| `DONE` | Task complete | Believe you've finished |
| `ESCALATE` | Need human help | Requires human intervention |

### Examples

**Joining:**
```
### [2024-01-28 14:30:00] red | JOIN
Hello! I'm red, joining the taskforce.
Will focus on investigating the database seeding approach.
Branch: $TF_BRANCH_PREFIX/fix-flaky-ci/red

```

**Progress update:**
```
### [2024-01-28 14:35:00] red | PROGRESS
Identified 3 potential causes:
1. Race condition in beforeAll hooks
2. Transaction not committed before assertions
3. Parallel test interference

Starting with hypothesis #1.

```

**Question:**
```
### [2024-01-28 14:40:00] blue | QUESTION
@red: Did you check if the issue only happens with parallelism > 2?
I'm seeing something that might be related.

```

**Answer:**
```
### [2024-01-28 14:42:00] red | ANSWER
@blue: Yes, confirmed. Only fails with parallel runs.
Check my findings file for details on the patterns I've seen.

```

**Finding:**
```
### [2024-01-28 14:50:00] yellow | FINDING
Found root cause! The `beforeAll` in fixtures.ts:142 doesn't await transaction commit.

The seeding query runs, but other parallel tests start before commit finishes.

Fix: Add `await tx.commit()` before returning from fixture setup.

```

**Waiting:**
```
### [2024-01-28 15:00:00] red | WAITING
Pushed fix to branch. CI running.
Pipeline: https://app.circleci.com/pipelines/xxx/123
Expected duration: ~20 mins

Will check back with exponential backoff.

```

**Done:**
```
### [2024-01-28 15:25:00] red | DONE
CI passed on all parallel runs (tested 3x).

Summary:
- Root cause: Transaction commit race condition
- Fix: Added explicit commit await in fixture setup
- Branch: $TF_BRANCH_PREFIX/fix-flaky-ci/red
- PR: #1234

Findings file updated with full investigation notes.
Waiting for human review.

```

---

## Findings File Format

**File:** `findings-{{agent-name}}.md`
**Access:** Full write (owner only)

### Structure

```markdown
# Findings - {{agent-name}}

## Status
{{current-status: exploring | blocked | validating | done}}

## Current Focus
{{what you're working on right now}}

## Hypotheses Tested

### Hypothesis 1: {{description}}
- **Result:** {{confirmed | rejected | inconclusive}}
- **Evidence:** {{what you found}}
- **Notes:** {{any additional context}}

### Hypothesis 2: {{description}}
...

## Dead Ends
- {{approach that didn't work and why}}
- {{another dead end}}

## Key Discoveries
- {{important finding 1}}
- {{important finding 2}}

## Relevant Code Locations
- `path/to/file.ts:123` - {{why it's relevant}}
- `another/file.ts:456` - {{description}}

## Open Questions
- {{question you haven't answered yet}}

## Next Steps
- {{what you plan to do next}}
```

---

## Task File Format

**File:** `TASK.md`
**Access:** Read-only for agents (created by human)

### Structure

```markdown
# Task: {{task-name}}

## Objective
{{clear description of what needs to be achieved}}

## Requirements Checklist
- [ ] Requirement 1
- [ ] Requirement 2
- [ ] Requirement 3

## Context
{{background information, links to issues, relevant history}}

## Relevant Files
- `path/to/file.ts` - {{why it's relevant}}

## Constraints
- {{any limitations or requirements}}

## Success Criteria
{{how we know the task is complete}}

## Notes
{{any additional information}}
```

---

## Addressing Other Agents

Use `@{{agent-name}}:` prefix when directing a message to a specific agent:
- `@red: Did you check the fixtures?`
- `@all: I think I found the root cause.`

Agents should respond to direct mentions in their next update cycle.
