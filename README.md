# solo-dev

A Claude Code framework for autonomous software development. Agents, hooks, skills, and rules that work together as a `.claude/` directory drop-in.

## What's in the box

```
agents/          5 specialized agents (orchestrator, researcher, implementer, verifier, context-keeper)
hooks/           8 hooks (safety-net, readonly-guard, commit-review-gate, auto-format, test-gate, sonnet-review-gate, agent-trace, agent-trace-output)
skills/          7 skills (autoresearch, compact, compact-agents, context-check, eval, plan-and-develop, qwen-review)
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

This scans your project, populates `CLAUDE.md`, makes hooks executable, creates language-specific rules, and lets you configure review gates and the project wiki.

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
| **orchestrator** | opus | Coordinates the pipeline. Routes wiki context to agents. Never writes code. |
| **researcher** | sonnet | Explores codebases AND designs plans. Produces context summaries + plan.md. |
| **implementer** | sonnet | Executes subtasks in isolated worktrees. |
| **verifier** | sonnet | Runs tests AND reviews diffs in one pass. Classifies mistakes. |
| **context-keeper** | haiku | Maintains wiki, CLAUDE.md, rules, and mistake log. |

This 5-agent architecture (4 workers + 1 coordinator) follows SOTA research on multi-agent coordination:
- **Centralized topology** suppresses error amplification ([DeepMind, 2025](https://arxiv.org/html/2512.08296v1))
- **4-agent threshold** — coordination gains plateau beyond this ([MAST, Cemri et al.](https://arxiv.org/abs/2503.13657))
- **Structured artifact flow** between stages prevents hallucination cascading ([MetaGPT](https://arxiv.org/abs/2308.00352))

### Pipeline

```
Research+Plan → Implement → Verify → Log    (3 handoffs)
```

1. **Research+Plan**: Researcher explores codebase, produces context summary + plan.md
2. **Implement**: Implementer(s) execute subtasks from the plan
3. **Verify**: Verifier runs tests + reviews diff in one pass
4. **Log**: Context-keeper persists findings to wiki + rules

### Hooks

| Hook | Event | What it does |
|---|---|---|
| **safety-net** | PreToolUse (Bash) | Blocks `rm -rf`, `git push --force`, `git reset --hard`, etc. |
| **readonly-guard** | PreToolUse (Write/Edit) | Blocks edits to files listed as read-only in CLAUDE.md. |
| **commit-review-gate** | PreToolUse (Bash) | Tool-augmented Sonnet reviews staged diff before commit. |
| **auto-format** | PostToolUse (Write/Edit) | Runs project formatter after every edit. |
| **test-gate** | Stop | Runs the test suite before Claude finishes a task. |
| **sonnet-review-gate** | (configurable) | Per-edit tool-augmented Sonnet review. Configurable at bootstrap. |
| **agent-trace** | PreToolUse (Agent) | Logs every agent spawn to `docs/sessions/agent-trace.log`. |
| **agent-trace-output** | PostToolUse (Agent) | Captures full agent outputs to `docs/sessions/agent-outputs/`. |

### Skills

| Skill | What it does |
|---|---|
| **autoresearch** | Autonomous experiment loop (modify, measure, keep/discard, repeat). |
| **compact** | Merges session summaries and experiment results into wiki. |
| **compact-agents** | Drains agent-specific memory into wiki pages. |
| **context-check** | Audits what each agent loads into context. Reports sizes, staleness, redundancy. |
| **eval** | Framework self-testing. Runs hook smoke tests and task-based agent evals. |
| **plan-and-develop** | Structured research + plan + implement workflow. |
| **qwen-review** | Local code review via Ollama (airgapped alternative to Sonnet review). |

### Knowledge system

The framework maintains a persistent, compounding knowledge base:

```
docs/wiki/index.md       Content-oriented catalog of all wiki pages
docs/wiki/{topic}.md     Synthesized knowledge pages with cross-references
docs/wiki/mistakes.md    Structured mistake log with error taxonomy
docs/sessions/           Agent trace logs and full output captures
```

### Mistake analysis

Agents report failures with a 4-type error taxonomy:

| Type | Meaning | Escalation |
|---|---|---|
| **context** | Wrong, missing, or stale information | gotcha → rule |
| **breakage** | Broke existing behavior or reintroduced a bug | rule → hook |
| **security** | Secrets, injection, unsafe patterns | hook |
| **quality** | Missing validation, convention violations | rule |

Mistakes graduate through escalation tiers: **gotcha** (wiki) → **rule** (CLAUDE.md) → **hook** (programmatic block).

### Review gates

Both review hooks use **tool-augmented Sonnet** — the reviewer gets `Read`, `Grep`, `Glob`, `Bash(git log*)`, and `Bash(git blame*)` to explore the codebase. Review mode is configurable at bootstrap:

- **Commit only** (recommended): review at commit time
- **Per-edit only**: review every file change
- **Both**: maximum coverage
- **None**: disable AI review gates

### Complexity-gated routing

The orchestrator routes tasks by complexity:

- **Simple** (1-2 files): Skip orchestration. Use Claude Code directly.
- **Medium** (3-5 files): Full pipeline, single implementer.
- **Complex** (6+ files): Full pipeline, parallel implementers in worktrees.

### Agent observability

Every agent spawn and completion is logged via hooks:

- `docs/sessions/agent-trace.log` — TSV timeline of all agent activity
- `docs/sessions/agent-outputs/` — full output capture per agent for MAST-style trace analysis

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
- **Configure review gates**: Choose review mode during `/bootstrap` or edit `settings.json` directly.
- **Swap review hook**: Replace Sonnet with local Qwen — see `skills/qwen-review/SKILL.md`.
- **Configure autoresearch**: Set metric, run command, and time budget in CLAUDE.md during bootstrap.
- **Add eval tasks**: Drop `.md` files in `evals/tasks/` with verification checks.
- **Compact knowledge**: Run `/compact` to merge sessions into wiki, `/compact-agents` to drain agent memory.
- **Audit context**: Run `/context-check` to see what each agent loads and identify bloat.

## License

MIT
