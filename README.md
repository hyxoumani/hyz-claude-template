# solo-dev

A Claude Code framework for autonomous software development. Agents, hooks, skills, and rules that work together as a `.claude/` directory drop-in.

## What's in the box

```
agents/          7 specialized agents (orchestrator, researcher, planner, implementer, tester, reviewer, context-keeper)
hooks/           6 hooks (safety-net, readonly-guard, commit-review-gate, auto-format, test-gate, sonnet-review-gate)
skills/          4 skills (autoresearch, eval, plan-and-develop, qwen-review)
rules/           Path-scoped rules (rust, testing, git, documentation, agent-memory)
commands/        Bootstrap command for first-time setup
evals/           Smoke tests for the framework itself
```

## Setup

1. Copy this repo into your project as `.claude/`:

```bash
git clone https://github.com/YOUR_USERNAME/claude-ref.git .claude
```

2. Bootstrap:

```bash
claude "/bootstrap"
```

This scans your project, populates `CLAUDE.md`, makes hooks executable, and creates language-specific rules.

3. Start working:

```bash
# Simple tasks — use Claude Code directly
claude

# Complex tasks — use the orchestrator
claude --agent orchestrator
```

## Architecture

### Agents

| Agent | Model | Role |
|---|---|---|
| **orchestrator** | opus | Coordinates the pipeline. Never writes code. |
| **researcher** | haiku | Explores codebases, produces context summaries. |
| **planner** | sonnet | Designs implementation plans from research. |
| **implementer** | sonnet | Executes subtasks in isolated worktrees. |
| **tester** | sonnet | Runs tests, reports structured results. |
| **reviewer** | sonnet | Reviews diffs with clean context (never sees implementation reasoning). |
| **context-keeper** | haiku | Maintains CLAUDE.md, rules, session logs. |

### Hooks

| Hook | Event | What it does |
|---|---|---|
| **safety-net** | PreToolUse (Bash) | Blocks `rm -rf`, `git push --force`, `git reset --hard`, etc. |
| **readonly-guard** | PreToolUse (Write/Edit) | Blocks edits to files listed as read-only in CLAUDE.md. |
| **commit-review-gate** | PreToolUse (Bash) | Sends staged diff to headless Claude Sonnet before commit. |
| **auto-format** | PostToolUse (Write/Edit) | Runs project formatter after every edit. |
| **test-gate** | Stop | Runs the test suite before Claude finishes a task. |
| **sonnet-review-gate** | (opt-in) | Per-edit Sonnet review. Available but not active by default. |

### Skills

| Skill | What it does |
|---|---|
| **autoresearch** | Autonomous experiment loop (modify, measure, keep/discard, repeat). Runs overnight. |
| **eval** | Framework self-testing. Runs hook smoke tests and task-based agent evals. |
| **plan-and-develop** | Structured research → plan → implement workflow. |
| **qwen-review** | Local code review via Ollama (airgapped alternative to Sonnet review). |

### Complexity-gated routing

The orchestrator routes tasks by complexity:

- **Simple** (1-2 files): Skip orchestration. Use Claude Code directly.
- **Medium** (3-5 files): researcher → planner → implementer → tester → reviewer.
- **Complex** (6+ files): Full pipeline with parallel implementers in worktrees.

## Principles

See [PRINCIPLES.md](PRINCIPLES.md). Ten engineering rules every agent follows:

1. Spec before implement
2. Root cause or nothing
3. Simplicity criterion
4. Minimal diff
5. Verify, don't trust
6. Keep or discard, no half-states
7. Compound knowledge
8. Autonomous by default
9. Search before building
10. Errors are for the next reader

## Running evals

```bash
# Smoke test all hooks
bash evals/run-smoke.sh

# Test a specific hook
bash evals/run-smoke.sh hook-safety-net

# Full eval runner (after bootstrap, in a git repo)
bash .claude/hooks/eval-runner.sh --trials 3
```

## Customization

- **Add language rules**: Create `.claude/rules/{language}.md` with `paths` frontmatter.
- **Swap review hook**: Replace Sonnet with local Qwen — see `skills/qwen-review/SKILL.md`.
- **Configure autoresearch**: Set metric, run command, and time budget in CLAUDE.md during bootstrap.
- **Add eval tasks**: Drop `.md` files in `evals/tasks/` with verification checks.

## License

MIT
