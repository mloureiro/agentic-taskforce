# Taskforce Rules

These are the core directives that **ALL** taskforce agents MUST follow.

---

## Session Setup (MANDATORY FIRST STEP)

**Before doing ANYTHING else, you MUST register your session.**

### Step 1: Verify Session ID

Check if your session has a `TF_SESSION_ID`:

```bash
echo $TF_SESSION_ID
```

If empty, **STOP and ask the user** - they need to spawn you with `tf-claude` instead of `claude`.

### Step 2: Register for Your Task

```bash
tf-register new <task-id>
```

This assigns you a color name automatically (red, blue, yellow, etc. in order).

### Step 3: If Registration Fails

If a previous session crashed and you need to take over:

```bash
tf-register claim <task-id> <color-name>
```

**If you cannot register or claim, ask the user what to do. DO NOT proceed without registration.**

---

## Identity

### Agent Names

Names are auto-assigned via `tf-register`, during the [mandatory step](#session-setup-mandatory-first-step)
**Do NOT manually pick a name.** The registration system handles this.

### Session Status

Check your current registration:

```bash
tf-register status
```

### Branch Naming

Each agent works on their own branch following this convention:

```
$TF_BRANCH_PREFIX/{{task-name}}/{{agent-name}}
```

Example: `user/fix-flaky-ci/red`

**Environment variable:** Set `TF_BRANCH_PREFIX` to your preferred prefix (e.g., your username).

---

## File Access Rules

**CRITICAL:** These rules prevent conflicts and ensure clean collaboration.

| Scope                                                 | Permission      | Examples                          |
| ----------------------------------------------------- | --------------- | --------------------------------- |
| `~/.claude-taskforce/*`                               | READ            | `RULES.md`, `FORMAT.md`, etc.     |
| `~/.claude-taskforce/{{task-name}}/*`                 | READ            | `TASK.md`, other agents' findings |
| `~/.claude-taskforce/{{task-name}}/.chat.log`          | **tf-chat ONLY** | See warning below                 |
| `~/.claude-taskforce/{{task-name}}/*-{{your-name}}.*` | FULL WRITE      | `findings-red.md`                 |

### ⛔ CHAT FILE WARNING

**IT IS STRICTLY FORBIDDEN TO READ OR WRITE `.chat.log` WITHOUT THE `tf-chat` COMMAND.**

- **DO NOT** use `cat`, `echo`, `read`, or any file operation on `.chat.log`
- **DO NOT** manually append to `.chat.log`
- **ALWAYS** use `tf-chat add` to write messages
- **ALWAYS** use `tf-chat unread` or `tf-chat search` to read messages

Violating this rule causes race conditions, corrupted entries, and coordination failures.

**Key implications:**

- Never edit another agent's files
- Never directly touch `.chat.log` - use `tf-chat` commands only
- Always use your agent name suffix for personal files

---

## Communication Rules

### Required Updates

You MUST post updates using `tf-chat add`:

1. When you join the taskforce: `tf-chat add -T JOIN -m "your introduction"`
2. Progress updates every 3-5 minutes while actively working: `tf-chat add -T PROGRESS -m "..."`
3. When you make a significant finding: `tf-chat add -T FINDING -m "..."`
4. When you have a question for others: `tf-chat add -T QUESTION -m "..."`
5. When you're entering a long wait: `tf-chat add -T WAITING -m "..."`
6. When you believe you've completed the task: `tf-chat add -T DONE -m "..."`

### Checking for Updates

Use `tf-chat` to check for messages from other agents:

- `tf-chat unread` - Show all unread messages (marks them as read)
- `tf-chat unread --status` - Just show count (does NOT mark as read)
- `tf-chat unread --status --full` - Show count + who posted what
- `tf-chat search -q "keyword"` - Search for specific topics

### Waiting Protocol

When waiting for external processes (CI, tests, other agents):

- Post that you're waiting: `tf-chat add -T WAITING -m "Waiting for CI..."`
- Use `tf-sleep` for automatic exponential backoff
- After each check, post status if there's news

### Response Expectations

- When another agent asks you a direct question (`@red:`), respond within your next update cycle
- Don't block on responses - continue other work while waiting

---

## Collaboration Rules

### No Blocking

- Never wait indefinitely for another agent
- If an agent goes silent, continue your work
- After 10+ minutes of silence from a critical agent, flag for human review

### Findings Sharing

- Keep your `findings-{{agent-name}}.md` file continuously updated
- Other agents may read your findings to avoid duplicate work
- Include: hypotheses tested, results, dead ends, code locations discovered

### When Another Agent Posts "DONE"

**DO NOT immediately stop your work.**

1. Finish your current task/test (don't start new experiments)
2. Validate their solution if possible
3. Post your assessment to the chat
4. Wait for human confirmation before fully stopping

**Why:** False positives happen. An agent might claim "done" but:

- Their fix doesn't meet all criteria
- Their fix has side effects (e.g., slower CI)
- Your parallel work might reveal a better solution

---

## Version Control Rules

Each task has its own git repository. **Commit early and often** to enable rollback and track progress.

### When to Commit

You MUST commit after:

1. **Any meaningful code change** - fixes, features, refactors
2. **Updating your findings file** - document your progress
3. **Before starting a new approach** - checkpoint your current state
4. **Before running risky operations** - tests, migrations, destructive changes

### Commit Message Format

```
[agent-name] Brief description of change

Optional longer explanation if needed.
```

Examples:
- `[red] Add transaction await to fix race condition`
- `[blue] Update findings with dead-end approaches`
- `[yellow] Checkpoint before trying alternative fix`

### Git Commands

```bash
# Stage and commit your changes
git add <files>
git commit -m "[red] Description"

# Check status
git status

# View recent commits
git log --oneline -5
```

### Important

- **Only commit files you own** (your findings file, code you're working on)
- **Never force push** - other agents may have pulled your changes
- **Keep commits atomic** - one logical change per commit
- Commits provide safety nets - if something breaks, you can rollback

---

## Completion Rules

### Individual Completion

When you believe you've solved your part:

1. Ensure your findings file is fully updated
2. Post `DONE` message to `.chat.log` with summary
3. Reference your branch and any PR if created
4. Wait for human review in your session

### Task Completion

The task is only complete when:

- All requirements from `TASK.md` are checked off
- Human has reviewed and confirmed
- Do NOT assume another agent's "DONE" means task is complete

---

## Escalation Rules

Flag for human attention when:

- You've hit the same blocker 3+ times
- A critical agent has been silent for 10+ minutes
- You need a decision that affects the overall approach
- You've found conflicting requirements
- External system access is needed (credentials, permissions)

To escalate: Post `ESCALATE` message type in chat and wait in your session.

---

## Helper CLI Tools

The following CLI tools are available in `~/.claude-taskforce/bin/` (should be in PATH).

**Use these tools.** They handle formatting, timestamps, backoff tracking, and state management automatically. Manual file operations are error-prone - the tools exist to prevent mistakes.

### Available Commands

| Command          | Purpose                        | Notes                      |
| ---------------- | ------------------------------ | -------------------------- |
| `tf-register`    | Manage session registration    | **MUST use first**         |
| `tf-chat add`     | Add an entry to chat log       | Auto-detects your identity |
| `tf-chat unread`  | Check for unread messages      | Auto-detects your task     |
| `tf-chat search`  | Search chat history            | Auto-detects your task     |
| `tf-sleep`       | Sleep with exponential backoff | Auto-detects your identity |
| `tf-status`      | Show taskforce dashboard       | Works for any task         |
| `tf-init`        | Initialize a new task folder   | Human only - do not use    |
| `tf-claude`      | Spawn a Claude session         | Human only - spawns you    |

### tf-register (REQUIRED)

Manage your session registration. **This is the first command you must run.**

```bash
tf-register new <task-id>               # Register for a task (auto-assigns color)
tf-register claim <task-id> <name>      # Take over a color (after crash)
tf-register status                       # Show your current registration
tf-register get                          # Get task and name (for scripts)
tf-register list [task-id]               # List all active registrations
```

### tf-init (Human Only)

Creates a new task folder. **Agents should NOT use this** - the human sets up the task.

```bash
tf-init -n <task-name> -d "description"
```

### tf-chat add

Add entries to the chat log (append-only). **Task and agent auto-detected from registration.**

```bash
tf-chat add -T <TYPE> -m "message"
tf-chat add -T <TYPE> --file /path/to/file
echo "message" | tf-chat add -T <TYPE>
```

**Types:** JOIN, PROGRESS, QUESTION, ANSWER, FINDING, DECISION, BLOCKER, WAITING, DONE, ESCALATE

### tf-chat unread

Check for unread messages. **Task auto-detected from registration.**

```bash
tf-chat unread                  # Show all unread messages (marks as read)
tf-chat unread --status         # Just show count (does NOT mark as read)
tf-chat unread --status --full  # Count + who posted what type
```

**Note:** Only `tf-chat unread` (without `--status`) marks messages as read.

### tf-chat search

Search chat history. **Task auto-detected from registration.**

```bash
tf-chat search -q "keyword"     # Search by content
tf-chat search -a <agent>       # Filter by agent
tf-chat search -T <TYPE>        # Filter by message type
tf-chat search -q "bug" -a red  # Combine filters
```

### tf-sleep

Smart sleep with automatic exponential backoff. **Task and agent auto-detected from registration.**

```bash
tf-sleep              # Sleep 5s (stage 0) - default
tf-sleep --continue   # Escalate if recently slept, otherwise 5s
tf-sleep --from 60    # Sleep 60s, continue from that stage
tf-sleep -u           # Wait for new messages (max 10 min)
tf-sleep -u --max 120 # Wait for new messages (max 2 min)
tf-sleep --dry-run    # Preview without sleeping
```

**Backoff sequence:** 5s → 10s → 30s → 60s → 90s → 120s → 180s (max)

**Default behavior:**
- Always starts at stage 0 (5s)

**With `--continue`:**
- If called quickly after last sleep → escalates to next stage
- If called after a gap (activity happened) → stays at stage 0

**With `--until-new-messages` / `-u`:**
- Polls for unread messages every 5 seconds
- Exits immediately when messages are found
- Times out after `--max` seconds (default: 10 minutes)

#### When to use `tf-sleep` vs regular `sleep`

**Use `tf-sleep`** for **unknown/variable wait times**:
- Polling for colleague responses → `tf-sleep -u` (waits for messages)
- Waiting for external processes with unpredictable duration
- Checking if something has changed

**Use `tf-sleep --continue`** when you're in a **retry loop**:
- After checking CI status and it's still running
- After polling and finding no updates yet
- Escalates wait time automatically to avoid hammering

**Use regular `sleep`** for **known wait times**:
- CI pipeline (~20 min) → `tf-chat add -T WAITING -m "CI running, waiting 20min"` then `sleep 1200`
- Test suite (~10 min) → `tf-chat add -T WAITING -m "Tests running, waiting 10min"` then `sleep 600`
- Build process (~5 min) → `tf-chat add -T WAITING -m "Building, waiting 5min"` then `sleep 300`

**Why?** Using `tf-sleep` for known long waits wastes resources by waking up every 3 minutes when you know nothing will change for 20 minutes. Log your wait to the chat, then use regular `sleep`.

---

## Anti-Patterns (DO NOT)

- Don't overwrite common files
- Don't work on another agent's branch without explicit coordination
- Don't claim "DONE" without meeting all requirements
- Don't go silent for extended periods without notice
- Don't make assumptions about other agents' progress - check the chat
- Don't duplicate work another agent already completed (check findings files)
- Don't make large changes without committing checkpoints
- Don't forget to commit before trying risky approaches
