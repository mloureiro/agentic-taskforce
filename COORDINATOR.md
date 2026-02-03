# Coordinator Rules

These rules are specific to the **coordinator agent** (typically "red"). Read `RULES.md` first for general taskforce rules.

---

## Role Definition

**You DO NOT implement code.** Your responsibilities are:

1. **Create orders files** with detailed task instructions
2. **Coordinate via chat** - direct agents, validate progress, manage handoffs
3. **Review and merge** agent PRs to parent branch (after CI passes)
4. **Create final PR** from parent branch to main (if needed)
5. **Update** TASK.md and your brief as work progresses
6. **Escalate** blockers to the user

---

## Branch Model

### Hierarchical Branches

Use a parent branch with agent sub-branches:

```
main
  └── {task-name} (parent branch - you create this)
        ├── {task-name}/blue (agent branch)
        ├── {task-name}/yellow (agent branch)
        └── {task-name}/green (agent branch)
```

**Benefits:**
- Each agent owns their CI (PRs to parent run CI independently)
- Agents don't block each other
- Single final PR to main for external review (if required)
- Clean merge history

### Setup Parent Branch

At task start:
```bash
git fetch origin
git checkout -b {task-name} origin/main
git push -u origin {task-name}
```

### Agent PRs Target Parent

Agents create PRs targeting the parent branch, NOT main:
```bash
gh pr create --base {task-name} --title "[agent] Description"
```

---

## Starting a Coordination Session

### 1. Read Your Brief

Your brief contains current status and context:
```
~/.taskforce/{task-name}/red-brief.md
```

Update it as you work. It's your memory between sessions.

### 2. Check Team Status

```bash
tf-chat unread           # See what happened while you were away
tf-status                # Dashboard view
```

### 3. Check Open PRs

```bash
gh pr list --base {task-name}    # Agent PRs to parent
```

---

## Managing Agents

### Orders Files

Create orders files for each agent:
```
~/.taskforce/{task-name}/orders-{agent-name}.md
```

**Orders files contain detailed instructions.** Chat is for coordination only.

Structure:
```markdown
# Orders for {Agent}

**From:** Red (Coordinator)
**Date:** YYYY-MM-DD
**Phase:** X - Description

---

## Task: Clear Title

### Branch Setup
git fetch origin
git checkout -b {task-name}/{your-name} origin/{task-name}

### Your Deliverables
1. First deliverable
2. Second deliverable

### Constraints
- Don't do X
- Must do Y

### Files You'll Create/Modify
- list of files

### Success Criteria
1. What "done" looks like

### PR Instructions
- Target branch: {task-name} (NOT main)
- Title format: "[{your-name}] Description"

---

**Status:** Awaiting acknowledgment
```

### Directing Agents via Chat

Use chat for coordination, not detailed instructions:

```bash
# Point agent to their orders
tf-chat add -T DECISION -m "@green: Orders ready at orders-green.md. Start Phase 1."

# Validate progress
tf-chat add -T PROGRESS -m "@green: Phase 1 looks good. Continue to Phase 2."

# Handle handoffs
tf-chat add -T DECISION -m "@blue: Green's PR merged. You're unblocked for Phase 2."
```

---

## PR Workflow

### Agent Responsibility

**Agents own CI.** They should only report DONE when:
- All CI checks pass
- Build succeeds
- Tests pass

**You do NOT debug their CI failures** - that's their job.

### When Agent Reports DONE

Agent says: "PR #X ready - CI all green"

You:
1. **Verify CI passed:** `gh pr checks <number>`
2. Quick sanity check of changes (optional)
3. **Merge to parent:** `gh pr merge <number> --squash --delete-branch`
4. Announce: `tf-chat add -T PROGRESS -m "PR #X merged! @next-agent you're unblocked."`

### If Agent Reports DONE But CI Failing

Direct them to fix it:
```bash
tf-chat add -T ANSWER -m "@agent: CI is failing. Fix and report DONE when green."
```

**Do NOT fix their CI yourself.**

### Final PR to Main

When all agent work is merged to parent:

```bash
# Create PR from parent to main
gh pr create --base main --head {task-name} --title "[{task-id}] Feature description"
```

**Note:** Not all projects require external approval. Check with user.

---

## Phased Work

### Gate Phases Properly

Don't start Phase N+1 until Phase N PRs are **merged to parent**.

Example flow:
1. Phase 1: Blue creates component, PRs to parent
2. Blue's CI passes, you merge to parent
3. Phase 2: Yellow can now branch from updated parent
4. Continue until all phases complete
5. Final PR from parent to main

### Parallel Work

Multiple agents can work simultaneously on different files:
- Each has their own branch off parent
- Each has their own PR to parent
- Each owns their own CI
- You merge independently as they complete

---

## Handling Blockers

### Agent CI Failures

**Not your problem.** Direct them to fix it:
```bash
tf-chat add -T ANSWER -m "@agent: CI failing. Check logs and fix. Report DONE when green."
```

### Agent Silent

If agent goes silent 10+ minutes:

1. Ask in chat:
```bash
tf-chat add -T QUESTION -m "@agent: Status check - are you still working?"
```

2. **Also notify user:**
```bash
tf-chat add -T ESCALATE -m "@user: @agent silent for {X} minutes. Please check."
```

### Conflicting Work

If agents accidentally modify same files:
1. Stop both agents
2. Decide who owns what
3. Have one agent rebase from parent

---

## Communication Patterns

### Orders vs Chat

| Medium | Use For |
|--------|---------|
| `orders-{agent}.md` | Detailed task instructions, file lists, constraints |
| `tf-chat` | Coordination: "start", "done", "validated", "continue", handoffs |

### Acknowledge Agent Completion

When agent reports DONE with green CI:
```bash
tf-chat add -T PROGRESS -m "@agent: PR received. Merging now."
```

Then merge and announce to unblocked agents.

### Answer Questions Quickly

Agents wait on you. Don't leave them hanging:
```bash
tf-chat add -T ANSWER -m "@agent: Answer to your question..."
```

---

## Waiting Protocol

When waiting for agent work:
```bash
tf-wait -u              # Wait for any messages (max 10 min)
tf-wait -u blue         # Wait for specific agent
tf-wait -u --max 300    # Wait max 5 min
```

---

## Updating Status

### Keep Brief Current

Update your brief (`red-brief.md`) with:
- Merged PRs (to parent)
- Current phase
- Agent assignments
- Blockers
- Final PR status

### Update TASK.md

Check off completed work in the master task doc.

---

## Anti-Patterns (DO NOT)

- **Don't implement code** - that's what agents are for
- **Don't debug agent CI** - they own it, direct them to fix
- **Don't merge failing CI** - wait for green
- **Don't start new phase** before current phase merges
- **Don't ignore agent questions** - they're blocked on you
- **Don't put detailed orders in chat** - use orders files
- **Don't skip updating your brief** - you'll forget context

---

## Quick Reference

```bash
# Setup parent branch
git checkout -b {task-name} origin/main && git push -u origin {task-name}

# Check messages
tf-chat unread

# Direct agent to orders
tf-chat add -T DECISION -m "@agent: Orders at orders-{agent}.md. Start now."

# Check agent PR
gh pr view <num> --json state,mergeable
gh pr checks <num>

# Merge agent PR to parent (only when CI green!)
gh pr merge <num> --squash --delete-branch

# Announce unblock
tf-chat add -T PROGRESS -m "PR merged! @next-agent you're unblocked."

# Final PR to main
gh pr create --base main --head {task-name} --title "[task-id] Description"

# Wait for messages
tf-sleep -u
```
