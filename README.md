# solo-dev

A Claude Code framework for autonomous software development. Agents, hooks, skills, and rules that work together as a `.claude/` directory drop-in.

## What's in the box

```
agents/          7 specialized agents (orchestrator, researcher, planner, implementer, tester, reviewer, context-keeper)
hooks/           6 hooks (safety-net, readonly-guard, commit-review-gate, auto-format, test-gate, sonnet-review-gate)
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
| **orchestrator** | opus | Coordinates the pipeline. Scans wiki, routes context to agents. Never writes code. |
| **researcher** | haiku | Explores codebases. Reads wiki + mistakes log before researching. Produces context summaries. |
| **planner** | sonnet | Designs implementation plans from research + wiki knowledge. |
| **implementer** | sonnet | Executes subtasks in isolated worktrees. Checks mistakes log before coding. |
| **tester** | sonnet | Runs tests, reports structured results. |
| **reviewer** | sonnet | Reviews diffs with clean context. Classifies mistakes with error taxonomy. |
| **context-keeper** | haiku | Maintains wiki, CLAUDE.md, rules, session logs, and mistakes log. |

### Hooks

| Hook | Event | What it does |
|---|---|---|
| **safety-net** | PreToolUse (Bash) | Blocks `rm -rf`, `git push --force`, `git reset --hard`, etc. |
| **readonly-guard** | PreToolUse (Write/Edit) | Blocks edits to files listed as read-only in CLAUDE.md. |
| **commit-review-gate** | PreToolUse (Bash) | Tool-augmented Sonnet reviews staged diff before commit. Can read project files, grep callers, check conventions. |
| **auto-format** | PostToolUse (Write/Edit) | Runs project formatter after every edit. |
| **test-gate** | Stop | Runs the test suite before Claude finishes a task. |
| **sonnet-review-gate** | (configurable) | Per-edit tool-augmented Sonnet review. Configurable at bootstrap. |

### Skills

| Skill | What it does |
|---|---|
| **autoresearch** | Autonomous experiment loop (modify, measure, keep/discard, repeat). Runs overnight. |
| **compact** | Merges session summaries and experiment results into wiki. Runs mistake pattern detection. |
| **compact-agents** | Drains agent-specific memory into wiki pages. Reconciles contradictions. |
| **context-check** | Audits what each agent loads into context. Reports sizes, staleness, redundancy. |
| **eval** | Framework self-testing. Runs hook smoke tests and task-based agent evals. |
| **plan-and-develop** | Structured research → plan → implement workflow. |
| **qwen-review** | Local code review via Ollama (airgapped alternative to Sonnet review). |

### Knowledge system

The framework maintains a persistent, compounding knowledge base inspired by [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f):

```
docs/wiki/index.md       Content-oriented catalog of all wiki pages
docs/wiki/{topic}.md     Synthesized knowledge pages with cross-references
docs/wiki/mistakes.md    Structured mistake log with error taxonomy
docs/log.md              Append-only chronological knowledge changelog
docs/sessions/           Ephemeral session summaries (feed into wiki)
```

**How it works:**
- The **context-keeper** synthesizes all findings into wiki pages — not append-only logs, but living documents that get rewritten as understanding deepens.
- Every wiki page has cross-references, key decisions, and gotchas.
- The **orchestrator** reads the wiki index at task start and routes relevant pages to each agent's brief.
- All agents read relevant wiki pages before starting work — pre-synthesized knowledge, not raw re-discovery.

### Mistake analysis

Agents report failures in a structured format. The context-keeper logs them to `docs/wiki/mistakes.md` with:

- **Error taxonomy**: 9 types (stale-context, missing-context, wrong-assumption, breaking-change, missing-validation, security, convention-violation, scope-creep, regression)
- **Root cause chains**: 5 Whys style — traces from symptom to systemic cause
- **Escalation tiers**: mistakes graduate through increasing automation:
  - **Gotcha** (wiki) → agent reads and avoids manually
  - **Rule** (CLAUDE.md/rules) → loaded into context automatically
  - **Hook** (pre-commit/pre-edit) → blocked programmatically

Pattern detection during `/compact` identifies recurring mistake types and domains, and recommends escalation.

### Review gates

Both review hooks use **tool-augmented Sonnet** — the reviewer gets `Read`, `Grep`, `Glob`, `Bash(git log*)`, and `Bash(git blame*)` to explore the codebase, check conventions, and grep for callers. Review mode is configurable at bootstrap:

- **Commit only** (recommended): review at commit time
- **Per-edit only**: review every file change
- **Both**: maximum coverage
- **None**: disable AI review gates

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
- **Configure review gates**: Choose review mode during `/bootstrap` or edit `settings.json` directly.
- **Swap review hook**: Replace Sonnet with local Qwen — see `skills/qwen-review/SKILL.md`.
- **Configure autoresearch**: Set metric, run command, and time budget in CLAUDE.md during bootstrap.
- **Add eval tasks**: Drop `.md` files in `evals/tasks/` with verification checks.
- **Compact knowledge**: Run `/compact` to merge sessions into wiki, `/compact-agents` to drain agent memory.
- **Audit context**: Run `/context-check` to see what each agent loads and identify bloat.

## License

MIT
