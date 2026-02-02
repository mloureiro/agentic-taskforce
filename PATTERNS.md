# Taskforce Patterns

This document describes common patterns for organizing taskforce work.

---

## Pattern 1: Parallel Exploration

**Use when:** The same problem can be approached from multiple angles, and you want to find the best solution quickly.

**Example:** Fixing a flaky test where the root cause is unknown.

### Structure

```
Problem X
├── Agent red    → Branch red    → Approach 1 (e.g., investigate timing)
├── Agent blue   → Branch blue   → Approach 2 (e.g., investigate data)
└── Agent yellow → Branch yellow → Approach 3 (e.g., investigate parallelism)
```

### Coordination

- All agents work on the **same problem** but **different hypotheses**
- Each agent has their **own branch** (important for separate CI runs)
- Communication via shared chat log
- First valid solution wins, but others validate before stopping

### Branch Strategy

Each agent creates their own branch from the same base:

```bash
git checkout main
git checkout -b ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/{{agent-name}}
```

### When Complete

- One agent finds a validated fix
- Others confirm the fix or offer improvements
- Human reviews and merges winning branch

---

## Pattern 2: Domain Split

**Use when:** A feature requires work in multiple domains (e.g., API + Frontend) that can proceed in parallel.

**Example:** Adding a new field that requires backend schema changes and frontend UI.

### Structure

```
Feature X
├── Agent red (API)  ──┐
│                      ├── Agree on interface/contract FIRST
├── Agent blue (FE) ───┘
│
├── Both work in parallel using agreed contract
└── Integration at the end
```

### Coordination

1. **Contract phase:** Agents discuss and agree on the interface (API response shape, GraphQL schema, etc.)
2. **Parallel phase:** Each agent implements their domain
3. **Integration phase:** Changes merged, end-to-end validation

### Branch Strategy

**Option A: Same branch** (when agents need to see each other's changes)
```bash
# Both work on same branch, careful with file conflicts
git checkout -b ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/shared
```

**Option B: Sequential branches** (cleaner separation)
```bash
# API agent first
git checkout main
git checkout -b ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/api

# FE agent branches from API branch
git checkout ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/api
git checkout -b ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/frontend
```

### Contract Agreement

In the chat log, explicitly document the agreed interface:

```
### [2024-01-28 14:35:00] red | DECISION
@blue: Proposing this GraphQL schema for the new endpoint:

type NewFeatureResponse {
  id: ID!
  status: String!
  data: JSON
}

Query {
  newFeature(id: ID!): NewFeatureResponse
}

Please confirm or suggest changes.

### [2024-01-28 14:38:00] blue | ANSWER
@red: Confirmed. I'll mock this response shape for frontend work.
```

---

## Pattern 3: Phased Pipeline

**Use when:** A large task has natural phases where later work depends on earlier work.

**Example:** Large refactoring that needs to be done incrementally.

### Structure

```
Large Feature
│
├── Phase 1: Foundation
│   ├── Agent red   → Core changes
│   └── Agent blue  → Schema updates
│
├── Phase 2: Implementation (branches from Phase 1)
│   ├── Agent yellow → Feature A
│   └── Agent green  → Feature B
│
└── Phase 3: Integration & Polish
    └── Agent red → Final integration
```

### Coordination

1. **Phase gates:** Phase N+1 only starts when Phase N is complete
2. **Handoffs:** Explicit handoff messages in chat
3. **Branch inheritance:** Later phases branch from earlier phases

### Branch Strategy

```bash
# Phase 1
git checkout main
git checkout -b ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/phase1

# Phase 2 (after phase 1 complete)
git checkout ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/phase1
git checkout -b ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}{{task-name}}/phase2-feature-a
```

### Handoff Protocol

```
### [2024-01-28 15:00:00] red | DONE
Phase 1 complete. Foundation is ready.
Branch: ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}big-feature/phase1

Phase 2 agents: please branch from my branch and continue.
Key files changed:
- src/core/foundation.ts
- src/schema/base.ts

Notes for Phase 2: See my findings file for architectural decisions made.
```

---

## Pattern 4: Research + Execute

**Use when:** The problem needs significant investigation before implementation can begin.

**Example:** Performance optimization where bottlenecks are unknown.

### Structure

```
Problem X
│
├── Research Phase
│   └── Agent red → Investigate, profile, document findings
│
└── Execute Phase (after research)
    ├── Agent blue   → Implement fix A
    └── Agent yellow → Implement fix B
```

### Coordination

- Research agent produces comprehensive findings document
- Execute agents read findings before starting
- Research agent may continue investigating while others implement

### Research Output

The research agent should produce a detailed findings file that becomes the reference for execute agents:

```markdown
# Findings - red (Research Phase)

## Investigation Summary
{{comprehensive summary of what was discovered}}

## Identified Issues
1. Issue A: {{description, location, impact}}
2. Issue B: {{description, location, impact}}

## Recommended Fixes
1. Fix for Issue A: {{approach, estimated complexity}}
2. Fix for Issue B: {{approach, estimated complexity}}

## Relevant Code
- `path/to/slow-function.ts:45` - Main bottleneck
- `path/to/query.ts:120` - N+1 query problem
```

---

## Pattern 5: Implement + Review

**Use when:** Code quality is critical and you want built-in review.

**Example:** Security-sensitive changes, complex business logic.

### Structure

```
Feature X
├── Agent red    → Implements
└── Agent blue   → Reviews red's work, suggests improvements
```

### Coordination

- Implementer posts progress and commits
- Reviewer checks changes, asks questions, suggests improvements
- Iterative until both are satisfied

### Review Protocol

```
### [2024-01-28 15:00:00] red | PROGRESS
Implemented the authentication flow. Ready for review.
Branch: ${TF_BRANCH_PREFIX:+$TF_BRANCH_PREFIX/}auth-feature/red
Commit: abc123

Key changes:
- src/auth/handler.ts - Main auth logic
- src/middleware/validate.ts - Token validation

### [2024-01-28 15:10:00] blue | FINDING
@red: Reviewed. Found a few issues:

1. src/auth/handler.ts:45 - Missing null check on user lookup
2. src/middleware/validate.ts:30 - Token expiry not validated

Also suggest: Extract token parsing to a helper function for reusability.

### [2024-01-28 15:20:00] red | PROGRESS
@blue: Fixed both issues and extracted the helper.
New commit: def456

Please re-review.
```

---

## Choosing a Pattern

| Situation | Recommended Pattern |
|-----------|-------------------|
| Unknown root cause, need to explore | Parallel Exploration |
| Clear separation of concerns (API/FE) | Domain Split |
| Large task with dependencies | Phased Pipeline |
| Need investigation first | Research + Execute |
| High-stakes changes | Implement + Review |
| Simple parallel tasks | Domain Split (simplified) |

You can also **combine patterns**. For example:
- Parallel Exploration to find the cause
- Then Domain Split to implement the fix across API and FE
