# claude-template

A minimal, portable Claude Code template. 2 agents, 3 skills, 5 hooks, path-scoped rules. Works on any project.

## What's in the box

```
.claude/
  PRINCIPLES.md          4 engineering principles (always loaded)
  CLAUDE.md.template     project config template (populated by /bootstrap)
  framework.json         manifest

  agents/
    developer.md         sonnet, worktree-isolated, full edit tools
    analyst.md           haiku, read-only, for noisy investigation

  skills/
    verify/              run tests + review diff
    plan-and-develop/    research → plan → implement workflow for medium+ changes
    compact/             synthesize session findings into docs/wiki/

  hooks/
    safety-net.sh        blocks rm -rf, force-push, etc.
    auto-format.sh       formats files after edits
    test-gate.sh         runs tests on Stop
    commit-review-gate.sh tool-augmented Sonnet review of staged diff
    agent-trace.sh       logs subagent dispatches

  rules/
    git.md, testing.md   global conventions
    {lang}.md            path-scoped, created per project by /bootstrap

  commands/
    bootstrap.md         first-run setup
```

## Setup

```bash
# Drop the framework into your project
git clone https://github.com/YOUR_USERNAME/claude-template.git .claude

# Bootstrap
claude "/bootstrap"

# Start working
claude
```

`/bootstrap` discovers your stack, populates `CLAUDE.md`, makes hooks executable, and creates language-specific rules.

## How it works

### Default flow

```
You ask Claude to do something
  │
  ├── Simple change (1-2 files)         → Edit inline
  ├── Noisy investigation needed        → Task → analyst (haiku, RO)
  ├── Multi-file or risky change        → Task → developer (sonnet, worktree)
  └── 6+ files, multi-subsystem         → /plan-and-develop

After any change:
  /verify (tests + diff review) → APPROVE | REQUEST_CHANGES | REJECT

End of session:
  /compact → drains findings into docs/wiki/
```

The main session is the orchestrator. Subagents run in fresh contexts with restricted tools and return ≤20-line summaries. Bulky artifacts (logs, full diffs) live on disk in `runs/{timestamp}/`, never in main context.

### Why 2 agents

Most multi-agent templates over-decompose. With this template, the *fresh context per dispatch* property comes from the `Task` tool itself — having more agent definitions doesn't add intelligence, just config overhead. Two definitions cover the structural distinctions that matter:

- **`developer`** centralizes worktree isolation + sonnet + edit permissions
- **`analyst`** centralizes read-only scoping + haiku for cost on noisy reads

Anything else is either a skill or an inline action.

## Hooks

| Hook | Event | Purpose |
|---|---|---|
| `safety-net.sh` | PreToolUse(Bash) | Blocks destructive commands |
| `commit-review-gate.sh` | PreToolUse(Bash) on git commit | Tool-augmented Sonnet reviews staged diff |
| `auto-format.sh` | PostToolUse(Write/Edit) | Runs language formatter |
| `test-gate.sh` | Stop | Runs test command, blocks if failing |
| `agent-trace.sh` | PreToolUse(Agent) | Logs subagent spawns to `docs/sessions/agent-trace.log` |

## Principles

See [`.claude/PRINCIPLES.md`](.claude/PRINCIPLES.md). Four constraints, every agent follows:

1. **Think Before Coding** — surface assumptions, don't hide confusion
2. **Simplicity First** — minimum code, nothing speculative
3. **Surgical Changes** — touch only what you must
4. **Goal-Driven Execution** — define success criteria, verify before claiming done

Adapted from [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills).

## Customization

- **Add language rules**: drop `.claude/rules/{lang}.md` with `paths:` frontmatter.
- **Add project skills**: drop `.claude/skills/{name}/SKILL.md` with a clear `description:` for routing.
- **Disable a hook**: remove its entry from `.claude/settings.json`.

## Roadmap

- v0.2: `/eval` skill + `evals/` smoke runner for template self-verification
- v0.2: `/context-check` skill for context-bloat audits
- v0.3: `/autonomous-dev` skill + `state/` durable iteration state for unattended overnight loops
- v0.3: `halt-monitor.sh` hook for budget/iteration caps in autonomous mode

## License

MIT
