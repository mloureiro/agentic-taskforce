# Agent Rules (Worker Bees)

These rules are for **implementation agents** (green, blue, pink, etc.). Read `RULES.md` first for general taskforce rules.

---

## Role Definition

**You implement code and complete tasks.** Your responsibilities are:

1. **Read orders** from coordinator (`orders-{your-name}.md`)
2. **Implement** the assigned work
3. **Own CI** - your PR must pass before reporting DONE
4. **Fix CI failures** - if your changes break CI, you fix them
5. **Atomic commits** - commit after each logical change
6. **Report progress** via chat
7. **Create PR** to parent branch when work is complete

---

## Branch Model

### Hierarchical Branches

You work on a sub-branch of the parent task branch:

```
main
  └── {task-name} (parent branch)
        └── {task-name}/{your-name} (YOUR branch)
```

**Your PR targets the parent branch, NOT main.**

---

## Starting Work

### 1. Check for Orders

Read your orders file:
```bash
cat ~/.taskforce/{task-name}/orders-{your-name}.md
```

Also check chat for any updates:
```bash
tf-chat unread
```

### 2. Create Your Branch

Branch from the **parent branch**, not main:
```bash
git fetch origin
git checkout -b {task-name}/{your-name} origin/{task-name}
```

Example: `sc-57566/top-partner-badge-emails/green`

### 3. Announce You're Starting

```bash
tf-chat add -T PROGRESS -m "@red: Starting work per orders-{your-name}.md"
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
- `[green] Add Hero component to template`
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
tf-wait -u red --max 300    # Wait for red's response (max 5 min)
```

Don't block forever - continue other work while waiting.

---

## CI Ownership

### You Own Your CI

**YOU are responsible for making CI pass.** This is non-negotiable.

Don't report DONE until:
- Lint passes
- Type check passes
- Tests pass
- Build succeeds

### If Your Changes Break CI

**You fix them.** Don't expect the coordinator to debug your CI.

1. Check logs: `gh pr checks <number>` or `gh run view <run-id> --log-failed`
2. Fix the issues locally
3. Push fixes
4. Wait for CI to re-run
5. Only report DONE when green

### Verify Locally First

Before pushing, run the project's verification commands. Check the project's `CLAUDE.md` or `AGENTS.md` for specific instructions.

**Common patterns:**
```bash
# Single-package projects
npm run lint && npm run typecheck && npm run build

# Monorepos (run for YOUR package only, not all)
cd <package> && npm run lint && npm run typecheck
```

**Note:** Some projects have constraints:
- **Monorepos:** Don't run checks for all packages - only your affected package(s)
- **Test limitations:** Some projects can't run tests locally (DB dependencies, time)
- **CI is authoritative:** If unsure what to run locally, push and let CI tell you

When in doubt, check the project docs or ask the coordinator.

### Only Report DONE When Green

```bash
# WRONG - CI hasn't passed yet
tf-chat add -T DONE -m "PR created!"

# WRONG - CI is failing
tf-chat add -T DONE -m "PR ready, CI has some issues"

# RIGHT - CI is green
tf-chat add -T DONE -m "@red: PR #15 ready - CI all green ✓"
```

---

## Creating PRs

### Target the Parent Branch

**IMPORTANT:** Your PR targets the parent branch, NOT main.

```bash
git push -u origin {task-name}/{your-name}
gh pr create --base {task-name} --title "[{your-name}] Description" --body "## Summary
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

**If CI fails, fix it before reporting DONE.**

### Report Completion

Only after CI passes:
```bash
tf-chat add -T DONE -m "@red: PR #X ready - CI all green ✓

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

Don't use tf-sleep for CI - use regular sleep since you know it takes time:
```bash
tf-chat add -T WAITING -r CI -m "CI running, waiting ~5-10 min"
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

## If You Need to Rebase

When coordinator merges other PRs to parent, you may need to rebase:

```bash
git fetch origin
git rebase origin/{task-name}
git push --force-with-lease
```

Then wait for CI to re-run.

---

## Exceptions: Non-Code Tasks

**Non-code tasks** are things like:
- Research (design inspiration, competitor analysis)
- Documentation writing (not in repo)
- File uploads, asset gathering
- Planning, analysis

If your task is truly non-code:
- You don't need CI to pass
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
- **Don't expect coordinator to fix your CI** - you own it
- **Don't PR to main** - PR to parent branch
- **Don't batch commits** - commit atomically
- **Don't go silent** - post progress every 3-5 min
- **Don't modify others' files** without coordination
- **Don't block forever** on questions - continue other work
- **Don't skip local checks** before pushing
- **Don't assume CI is fast** - runners can take time

---

## Quick Reference

```bash
# Start work (branch from PARENT, not main)
git fetch origin
git checkout -b {task-name}/{your-name} origin/{task-name}

# Commit atomically
git add <files> && git commit -m "[{your-name}] description"

# Check local (see project's CLAUDE.md for specific commands)
# Example: cd <package> && npm run lint && npm run typecheck

# Push and create PR (to PARENT branch)
git push -u origin {task-name}/{your-name}
gh pr create --base {task-name} --title "[{your-name}] X" --body "..."

# Check CI
gh pr checks <number>

# If CI fails - FIX IT YOURSELF
gh run view <run-id> --log-failed
# ... fix issues ...
git push

# Report progress
tf-chat add -T PROGRESS -m "..."

# Report done (ONLY when CI green)
tf-chat add -T DONE -m "@red: PR #X ready - CI green ✓"

# Wait for response
tf-wait -u red --max 300

# Rebase if needed
git fetch origin && git rebase origin/{task-name} && git push --force-with-lease
```
