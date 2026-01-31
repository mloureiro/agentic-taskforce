# Agentic Taskforce

A framework for coordinating multiple AI agent instances working together on the same task.

## What is this?

Claude Taskforce enables **parallel problem-solving** by spawning multiple Claude agents that can:

- Work on the same problem from different angles simultaneously
- Communicate through a shared chat log
- Track their findings independently
- Coordinate without stepping on each other's work

Think of it as a "war room" where multiple AI agents collaborate like a team of engineers debugging a complex issue.

## Why?

Some problems benefit from parallel exploration:

- **Flaky tests** - One agent investigates timing, another checks data, a third looks at parallelism
- **Complex bugs** - Multiple hypotheses can be tested simultaneously
- **Large features** - API and frontend work can proceed in parallel after agreeing on contracts
- **Research tasks** - Different agents explore different parts of a codebase

Instead of one agent trying approaches sequentially, multiple agents can divide and conquer.

## How it works

```
~/.agentic-taskforce/          # Default, configurable via TF_ROOT
├── RULES.md                   # Core directives for agents
├── FORMAT.md                  # File formats and conventions
├── PATTERNS.md                # Coordination patterns
├── LIFECYCLE.md               # Agent states and protocols
├── bin/                       # CLI tools
│   ├── tf-claude              # Spawn agent sessions
│   ├── tf-init                # Create new tasks
│   ├── tf-register            # Agent registration
│   ├── tf-chat                # Chat log operations
│   ├── tf-sleep               # Smart exponential backoff
│   └── tf-status              # Dashboard view
│
└── <task-name>/               # Per-task folder (separate git repo)
    ├── TASK.md                # Mission briefing
    ├── .chat.log              # Agent communication
    └── findings-*.md          # Per-agent findings
```

Each agent:

1. Registers with a color name (red, blue, yellow, etc.)
2. Works on their own git branch
3. Posts updates to a shared chat log
4. Maintains their own findings file
5. Commits changes frequently for rollback safety

## Quick Start

### Setup

```bash
# Add to your shell profile (~/.zshrc or ~/.bashrc)
export TF_ROOT="/path/to/clone/location"
export PATH="$TF_ROOT/bin:$PATH"
export TF_BRANCH_PREFIX="your-username"

```

### Create a task

```bash
tf-init --name fix-flaky-tests --description "Fix the flaky CI tests in parallel runs"
```

### Spawn agents

```bash
# Terminal 1
tf-claude fix-flaky-tests

# Terminal 2
tf-claude fix-flaky-tests

# Terminal 3
tf-claude fix-flaky-tests
```

Each agent automatically gets assigned a unique color name and starts collaborating.

## Coordination Patterns

| Pattern                  | Use Case                                    |
| ------------------------ | ------------------------------------------- |
| **Parallel Exploration** | Unknown root cause, try multiple hypotheses |
| **Domain Split**         | API + Frontend work in parallel             |
| **Phased Pipeline**      | Large tasks with dependencies               |
| **Research + Execute**   | Investigation before implementation         |
| **Implement + Review**   | Built-in code review                        |

See [PATTERNS.md](PATTERNS.md) for detailed examples.

## CLI Tools

| Command             | Purpose                   |
| ------------------- | ------------------------- |
| `tf-claude [task]`  | Spawn an agent session    |
| `tf-init -n <name>` | Create a new task         |
| `tf-register`       | Manage agent registration |
| `tf-chat add`       | Post to chat log          |
| `tf-chat unread`    | Check for new messages    |
| `tf-sleep`          | Smart backoff for polling |
| `tf-status`         | View taskforce dashboard  |

## Documentation

- [RULES.md](RULES.md) - Core directives agents must follow
- [FORMAT.md](FORMAT.md) - File formats and naming conventions
- [PATTERNS.md](PATTERNS.md) - Coordination patterns and examples
- [LIFECYCLE.md](LIFECYCLE.md) - Agent states and protocols

## Requirements

- [Claude Code CLI](https://github.com/anthropics/claude-code)
- Bash 4.0+
- Git
