# Agent Lifecycle

This document describes agent states, polling protocols, and completion flow.

---

## Agent States

```
┌─────────────┐
│   JOINING   │  Reading docs, picking name, posting JOIN
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  EXPLORING  │◄─────────────────────────────────────┐
└──────┬──────┘                                      │
       │                                             │
       ├────────────────┬────────────────┐           │
       ▼                ▼                ▼           │
┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│   WAITING   │  │   BLOCKED   │  │ FOUND_FIX   │   │
│ (CI, tests) │  │  (need help)│  │             │   │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │
       │                │                │           │
       │                │                ▼           │
       │                │         ┌─────────────┐   │
       │                │         │ VALIDATING  │   │
       │                │         └──────┬──────┘   │
       │                │                │           │
       │                ▼                │           │
       │         ┌─────────────┐         │           │
       │         │  ESCALATE   │         │           │
       │         │(need human) │         │           │
       │         └─────────────┘         │           │
       │                                 │           │
       └─────────────────────────────────┼───────────┘
                                         │
                                         ▼
                                  ┌─────────────┐
                                  │    DONE     │
                                  │(wait review)│
                                  └─────────────┘
```

### State Descriptions

| State | Description | Next Actions |
|-------|-------------|--------------|
| JOINING | Reading taskforce docs, picking name | Post JOIN message, move to EXPLORING |
| EXPLORING | Actively investigating/implementing | Continue work, post PROGRESS updates |
| WAITING | Waiting for external process (CI, etc.) | Poll with backoff, move to EXPLORING when done |
| BLOCKED | Hit an obstacle, need input | Post BLOCKER, try alternatives, or ESCALATE |
| FOUND_FIX | Believe solution is found | Move to VALIDATING |
| VALIDATING | Testing the solution | If passes → DONE, if fails → EXPLORING |
| ESCALATE | Need human intervention | Wait for human, then resume |
| DONE | Task complete | Wait for human review |

---

## Joining Protocol

When you first join a taskforce:

1. **Read the docs:**
   - `$TF_ROOT/RULES.md`
   - `$TF_ROOT/FORMAT.md`
   - `$TF_ROOT/PATTERNS.md`
   - `$TF_ROOT/LIFECYCLE.md` (this file)

2. **Read the task:**
   - `$TF_ROOT/{{task-name}}/TASK.md`

3. **Check existing chat:**
   - `$TF_ROOT/{{task-name}}/chat-common.log`
   - See what names are taken, what's been discussed

4. **Pick your name:**
   - First available from: red, blue, yellow, green, pink, purple, orange, cyan

5. **Post JOIN message:**
   ```
   ### [YYYY-MM-DD HH:MM:SS] {{name}} | JOIN
   Hello! I'm {{name}}, joining the taskforce.
   Will focus on: {{your initial focus area}}
   Branch: $TF_BRANCH_PREFIX/{{task-name}}/{{name}}
   ```

6. **Wait for sync:**
   - After posting JOIN, wait up to 3 minutes
   - This gives other agents time to join and read
   - Check for responses or other JOIN messages

7. **Create your branch:**
   ```bash
   git checkout {{base-branch}}
   git checkout -b $TF_BRANCH_PREFIX/{{task-name}}/{{name}}
   ```

8. **Create your findings file:**
   - `$TF_ROOT/{{task-name}}/findings-{{name}}.md`

9. **Begin work**

---

## Polling Protocol

When waiting for external processes (CI, other agents, etc.), use exponential backoff.

### Wait Sequence

```
5s → 10s → 30s → 60s → 90s → 120s → 180s (max)
```

### Implementation

```
wait_times = [5, 10, 30, 60, 90, 120, 180]
wait_index = 0

while not done:
    # Check for updates
    check_chat_log()
    check_ci_status()  # or whatever you're waiting for

    if has_actionable_update:
        process_update()
        wait_index = 0  # Reset on activity
    else:
        # Sleep with backoff
        sleep(wait_times[min(wait_index, len(wait_times) - 1)])
        wait_index += 1
```

### Chat Etiquette While Waiting

Post a `WAITING` message before entering a long wait:

```
### [2024-01-28 15:00:00] red | WAITING
CI running for my fix branch.
Pipeline URL: https://...
Expected: ~20 mins

Will poll with backoff and report when done.
```

After each significant poll (not every 5 seconds), update if there's news:

```
### [2024-01-28 15:15:00] red | PROGRESS
CI still running. 12/15 jobs complete.
No failures yet. Continuing to wait.
```

---

## Handling "DONE" from Another Agent

When another agent posts `DONE`:

**DO NOT immediately stop.**

### Protocol

1. **Acknowledge:**
   ```
   ### [2024-01-28 15:30:00] blue | PROGRESS
   @red: Saw your DONE. Will finish my current test and then validate.
   ```

2. **Finish current work:**
   - Complete whatever test/experiment you were running
   - Don't start new experiments

3. **Validate if possible:**
   - Check their branch/PR
   - Run their fix locally if feasible
   - Verify it meets all requirements from TASK.md

4. **Report:**
   ```
   ### [2024-01-28 15:45:00] blue | PROGRESS
   @red: Validated your fix. LGTM - all requirements met.

   My parallel investigation also complete. Updating findings file.
   Waiting for human review.
   ```

**Why not stop immediately?**
- The "done" claim might be a false positive
- Their fix might have side effects (slower CI, etc.)
- Your parallel work might reveal a better solution or important context

---

## Completion Flow

### Individual Agent Completion

1. **Validate your solution:**
   - Run tests/CI
   - Verify all requirements from TASK.md

2. **Update findings file:**
   - Document everything you discovered
   - Include what worked and what didn't

3. **Post DONE:**
   ```
   ### [2024-01-28 15:50:00] red | DONE
   Fix validated. CI passed 3x consecutive runs.

   Summary:
   - Root cause: {{brief description}}
   - Fix: {{what you changed}}
   - Branch: $TF_BRANCH_PREFIX/{{task-name}}/red
   - PR: #1234 (if created)

   Findings file updated. Waiting for human review.
   ```

4. **Wait for human review:**
   - Stay in your session
   - Human will review and confirm

### Task Completion

The task is complete when:
- [ ] At least one agent has posted DONE
- [ ] Other agents have validated (or yielded)
- [ ] All requirements from TASK.md are satisfied
- [ ] Human has reviewed and confirmed

---

## Heartbeat & Silence

### Expected Activity

- Post progress every 3-5 minutes while actively working
- Post before entering long waits
- Respond to direct mentions within your next update cycle

### Detecting Silent Agents

If an agent hasn't posted for:
- **5 minutes:** Probably just focused on work
- **10 minutes:** Worth a ping in chat
- **15+ minutes:** Flag for human attention

### Pinging Silent Agents

```
### [2024-01-28 16:00:00] blue | QUESTION
@yellow: Haven't seen an update in a while. Are you still working on the parallelism angle?
```

### Escalating Silent Agent

```
### [2024-01-28 16:10:00] blue | ESCALATE
@human: Agent yellow has been silent for 15+ minutes.
Last message was about investigating parallelism issues.
May need attention.
```

---

## Error Recovery

### If You Crash/Restart

1. Read the chat log to understand current state
2. Post a reconnection message:
   ```
   ### [2024-01-28 16:30:00] red | PROGRESS
   Back online after interruption.
   Reading chat to catch up on what I missed.
   ```
3. Resume work from where you left off

### If Another Agent Crashes

- Continue your work
- Don't take over their tasks unless explicitly asked
- Flag for human if their work was critical

### If You Hit Repeated Failures

After 3 attempts at the same approach:

```
### [2024-01-28 16:45:00] red | BLOCKER
Hit a wall. Tried 3 variations of the transaction fix, all failed.

Attempts:
1. Explicit commit → Still flaky
2. Separate transaction → Breaks other tests
3. Lock mechanism → Deadlock

Options:
A) Try completely different approach
B) Need human guidance
C) Another agent wants to try?
```
