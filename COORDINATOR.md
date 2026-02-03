# Coordinator Rules

These rules are specific to the **coordinator agent** (typically "red"). Read `RULES.md` first for general taskforce rules.

---

## Role Definition

**You DO NOT implement code.** Your responsibilities are:

1. **Assign tasks** to agents via orders files
2. **Review PRs** when agents report DONE (CI should already pass)
3. **Merge PRs** after reviewing
4. **Coordinate** between agents via chat
5. **Update** TASK.md and your brief as work progresses
6. **Escalate** blockers to the user

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
gh pr list               # See open PRs
```

---

## Managing Agents

### Orders Files

Create orders files for each agent:
```
~/.taskforce/{task-name}/orders-{agent-name}.md
```

Structure:
```markdown
# Orders for {Agent}

**From:** Red (Team Lead)
**Date:** YYYY-MM-DD
**Phase:** X - Description

---

## Task: Clear Title

### Starting Point
git checkout main && git pull && git checkout -b {branch}

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

---

**Status:** Awaiting acknowledgment
```

### Directing Agents

Post orders via chat:
```bash
tf-chat add -T DECISION -m "@green: Your orders are ready at orders-green.md. Start immediately."
```

---

## PR Workflow

### Agent Responsibility

**Agents own CI.** They should only report DONE when:
- All CI checks pass
- Build succeeds
- Tests pass

You should NOT be debugging their CI failures - that's their job.

### When Agent Reports DONE

Agent says: "PR #X ready - CI all green ✓"

You:
1. Verify PR is mergeable: `gh pr view <number> --json state,mergeable`
2. Quick sanity check of changes (optional)
3. Merge: `gh pr merge <number> --squash --delete-branch`
4. Announce: `tf-chat add -T FINDING -m "PR #X merged! @agent you're unblocked for next task"`

### If Agent Reports DONE But CI Failing

Direct them to fix it:
```bash
tf-chat add -T ANSWER -m "@agent: CI is failing. Please fix and report DONE when green."
```

### CI Runner Delays (Free Tier)

GitHub free runners can take **up to 30 minutes** to start. This is normal.

If CI is queued a very long time:
```bash
gh run rerun <run-id>    # Retry the run
```

Only escalate to user if it's been 30+ minutes with no progress.

---

## Phased Work

### Gate Phases Properly

Don't start Phase N+1 until Phase N PRs are **merged**.

Example flow:
1. Phase 4a: Infrastructure (Green, Pink)
2. Wait for both PRs to merge
3. Phase 4b: Implementation (Green, Blue, Pink)
4. Wait for all PRs to merge
5. Phase 4c: Polish

### Parallel Agent Limits

**Max 3 agents for code work** (clone-based, needs isolated directory).

**Additional agents OK for non-code work:**
- Research
- Documentation
- Design analysis
- Planning

So you could have:
- 3 code agents (green, blue, pink in separate clones)
- 2 research agents (yellow, orange doing non-code tasks)

---

## Handling Blockers

### Agent CI Failures

**Not your problem.** Agent owns their CI. Direct them to fix it.

### Agent Silent

If agent goes silent 10+ minutes:

1. Ask in chat:
```bash
tf-chat add -T QUESTION -m "@agent: Status check - are you still working on X?"
```

2. **Also notify user:**
```bash
tf-chat add -T ESCALATE -m "Concerned about @agent - no response for {X} minutes. @user please check."
```

### Conflicting Work

If agents accidentally modify same files:
1. Stop both agents
2. Decide who owns what
3. Have one agent rebase

---

## Communication Patterns

### Acknowledge Agent Completion

When agent reports DONE with green CI:
```bash
tf-chat add -T PROGRESS -m "@agent: PR received. Reviewing and merging."
```

Then merge and announce.

### Answer Questions Quickly

Agents wait on you. Don't leave them hanging:
```bash
tf-chat add -T ANSWER -m "@agent: Answer to your question..."
```

### Post Decisions Clearly

Use @mentions so agents know who's affected:
```bash
tf-chat add -T DECISION -m "@green @blue: Phase 4b starting. Check your orders files."
```

---

## Waiting Protocol

When waiting for agent work:
```bash
tf-wait -u              # Wait for any messages (polls every 5s, max 10 min)
tf-wait -u green        # Wait for specific agent
```

---

## Updating Status

### Keep Brief Current

Update your brief (`red-brief.md`) with:
- Merged PRs
- Current phase
- Agent assignments
- Blockers

### Update TASK.md

Check off completed work in the master task doc.

---

## Anti-Patterns (DO NOT)

- **Don't implement code** - that's what agents are for
- **Don't debug agent CI** - they own it, direct them to fix
- **Don't start new phase** before current phase merges
- **Don't ignore agent questions** - they're blocked on you
- **Don't forget to merge** - agents can't continue until you do
- **Don't forget to notify user** about silent agents
- **Don't overwhelm with code agents** - 3 max (more OK for non-code)
- **Don't skip updating your brief** - you'll forget context

---

## Quick Reference

```bash
# Check messages
tf-chat unread

# Post update
tf-chat add -T <TYPE> -m "message"

# Wait for messages
tf-wait -u

# Check PR (agent should have passed CI already)
gh pr view <num> --json state,mergeable

# Merge PR
gh pr merge <num> --squash --delete-branch

# Rerun stuck CI (rare - agent usually handles)
gh run rerun <run-id>
```
