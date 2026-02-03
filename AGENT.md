# Agent Rules (Worker Bees)

These rules are for **implementation agents** (green, blue, pink, etc.). Read `RULES.md` first for general taskforce rules.

---

## Role Definition

**You implement code and complete tasks.** Your responsibilities are:

1. **Read orders** from coordinator (orders-{your-name}.md)
2. **Implement** the assigned work
3. **Own CI** - your PR must pass before reporting DONE
4. **Atomic commits** - commit after each logical change
5. **Report progress** via chat
6. **Create PR** when work is complete and CI passes

---

## Starting Work

### 1. Check for Orders

```bash
cat ~/.taskforce/{task-name}/orders-{your-name}.md
```

Or check chat for directions:
```bash
tf-chat unread
```

### 2. Create Your Branch

Always work from fresh main:
```bash
git fetch origin
git checkout main
git pull origin main
git checkout -b {task-name}/{your-name}
```

Branch naming: `{task-name}/{your-name}` (e.g., `tunadao-modernize-3/green`)

### 3. Announce You're Starting

```bash
tf-chat add -T PROGRESS -m "@red: Starting work on {task}. Will: 1) ... 2) ... 3) ..."
```

---

## Implementation Workflow

### Atomic Commits

Commit after **each logical change**, not in batches:

```bash
git add <specific-files>
git commit -m "[{your-name}] Brief description"
```

Examples:
- `[green] Add Hero component to classic template`
- `[blue] Fix lint errors in resolver`

### Progress Updates

Post updates every 3-5 minutes while actively working:
```bash
tf-chat add -T PROGRESS -m "Completed X, now working on Y..."
```

### Questions

If unclear about requirements:
```bash
tf-chat add -T QUESTION -m "@red: Should I do X or Y?"
tf-wait -u red --max 300    # Wait for red's response (max 5 min, assuming red is coordinator)
```

Don't block forever - continue other work while waiting.

---

## CI Ownership

### You Own Your CI

**YOU are responsible for making CI pass.** Don't report DONE until:
- Lint passes
- Type check passes
- Tests pass
- Build succeeds

### Verify Locally First

Before pushing:
```bash
npm run lint
npm run typecheck
npm run test
npm run build
```

### After Pushing

Check CI status:
```bash
gh pr checks <number>
```

**If CI fails:**
1. Check logs: `gh run view <run-id> --log-failed`
2. Fix the issues locally
3. Push fixes
4. Wait for CI to re-run

**Note:** Free GitHub runners can take up to 30 minutes to start. Be patient.

### Only Report DONE When Green

```bash
# WRONG - CI hasn't passed yet
tf-chat add -T DONE -m "PR created!"

# RIGHT - CI is green
tf-chat add -T DONE -m "PR #15 ready - all CI checks green ✓"
```

---

## Creating PRs

### When to Create PR

Create PR when:
1. Work is complete per orders
2. Local checks pass (lint, typecheck, build, tests)
3. You've verified CI will likely pass

### Create the PR

```bash
git push -u origin {branch-name}
gh pr create --title "feat: description" --body "## Summary
- Point 1
- Point 2

## Test plan
- [ ] Verify X
- [ ] Check Y"
```

### Monitor CI

After PR creation, monitor CI:
```bash
gh pr checks <number>
```

If CI fails, fix it before reporting DONE.

### Report Completion

Only after CI passes:
```bash
tf-chat add -T DONE -m "@red: PR #X ready for review - CI all green ✓

Deliverables:
- ✅ Item 1
- ✅ Item 2
- ✅ Item 3"
```

---

## Waiting Protocol

### Waiting for Coordinator Response

```bash
tf-chat add -T QUESTION -m "@red: Question..."
tf-wait -u red --max 300    # Wait max 5 min for red
```

### Waiting for CI

Don't use tf-wait for CI - use regular sleep since you know it takes time:
```bash
tf-chat add -T WAITING -m "CI running, waiting ~5-10 min"
sleep 300    # 5 min
gh pr checks <number>
```

### Waiting for Blocker Resolution

If blocked on something external:
```bash
tf-chat add -T BLOCKER -m "Blocked on X, waiting for Y"
tf-wait -u    # Wait for any message
```

---

## Coordination with Other Agents

### Check for Overlapping Work

Before starting, check what others are doing:
```bash
tf-chat unread
cat ~/.taskforce/{task-name}/findings-*.md
```

### Don't Step on Toes

- Work only on your assigned files
- If you need to modify shared files, coordinate first
- Never push to another agent's branch

### Handoffs

**Do NOT announce unblocks to other agents.** That's the coordinator's job.

Your job:
1. Report DONE when CI green
2. Wait for coordinator to merge
3. Coordinator will announce to other agents

```bash
# WRONG - don't do this
tf-chat add -T PROGRESS -m "@pink: My PR merged - you're unblocked"

# RIGHT - just report done to coordinator
tf-chat add -T DONE -m "@red: PR #X ready - CI green ✓"
```

---

## Exceptions: Non-Code Tasks

**Non-code tasks** are things like:
- Research (design inspiration, competitor analysis)
- Documentation writing (not in repo)
- File uploads, asset gathering
- Planning, analysis

If your task is truly non-code:
- You don't need CI to pass
- You don't count toward the 3 code-agent limit
- Report findings via chat or findings files

**If you're making commits, you're a code agent.** This includes:
- Writing markdown in the repo
- Adding config files
- Any git commits at all

---

## Partial Work

If your task is **partial work** (someone else continues):
- Clearly document what's done vs remaining
- Report DONE to coordinator with status
- Coordinator handles handoff to next agent

---

## Anti-Patterns (DO NOT)

- **Don't report DONE with failing CI** - fix it first
- **Don't batch commits** - commit atomically
- **Don't go silent** - post progress every 3-5 min
- **Don't modify others' files** without coordination
- **Don't block forever** on questions - continue other work
- **Don't skip local checks** before pushing
- **Don't assume CI is fast** - free runners can take 30 min

---

## Quick Reference

```bash
# Start work
git checkout main && git pull && git checkout -b {task}/{name}

# Commit atomically
git add <files> && git commit -m "[{name}] description"

# Check local
npm run lint && npm run typecheck && npm run build

# Push and create PR
git push -u origin {branch}
gh pr create --title "feat: X" --body "..."

# Check CI
gh pr checks <number>

# Report progress
tf-chat add -T PROGRESS -m "..."

# Report done (ONLY when CI green)
tf-chat add -T DONE -m "@red: PR #X ready - CI green ✓"

# Wait for response
tf-wait -u red --max 300
```
