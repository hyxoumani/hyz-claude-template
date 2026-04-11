---
name: Bootstrap and Orchestrator Architecture
description: Core framework design—bootstrap skill for project setup, orchestrator agent for task coordination, agent spawning mechanism, skill definitions
type: reference
---

## Bootstrap Skill

**Location**: `.claude/commands/bootstrap.md`

**Purpose**: First-run setup that discovers project context and configures the development environment.

**What it does** (13-step process):
1. **Framework integrity check** — verify .claude/ directories exist (agents/, skills/, hooks/, rules/, commands/)
2. **Detect existing bootstrap** — ask user to re-bootstrap or skip
3. **Make hooks executable** — chmod +x on all .claude/hooks/*.sh
4. **Discover project** — scan codebase, README, package.json/Cargo.toml, git state, tests, config files
5. **Identify key info** — extract Stack, Build, Test, Lint, Run, Entry points, Read-only files
6. **Confirm with user** — present detected config for approval/correction via AskUserQuestion
7. **Generate CLAUDE.md** — copy from template, replace _TBD_ placeholders with confirmed values
8. **Create language-specific rules** — populate .claude/rules/{language}.md based on detected stack
9. **Configure metric** (optional) — set up autoresearch optimization metric if user wants
10. **Verify hooks** — test readonly-guard and test command functionality
11. **Create directories** — mkdir -p docs/plans docs/sessions
12. **Commit** — stage portable framework files, commit as "chore: bootstrap solo-dev framework"
13. **Summary** — report what was configured, next steps

**Key outputs**:
- `CLAUDE.md` — main project config file with Stack, Build, Test, Lint, Read-only, Architecture
- `.claude/rules/{language}.md` — path-scoped rules created for each language found
- Hooks verified and executable
- Initial commit with framework files

**Note**: Bootstrap checks for "BOOTSTRAPPED" marker in CLAUDE.md to prevent re-running on already-configured projects.

---

## Orchestrator Agent

**Location**: `.claude/agents/orchestrator.md`

**Model**: claude-opus (large model for complex reasoning)

**Tools**: Agent() spawning capability + Read, Grep, Glob, Bash

**Purpose**: Central coordinator for all development tasks. Decomposes work, delegates to specialized agents, synthesizes results. Main entry point: `claude --agent orchestrator`

**Core responsibilities**:

1. **Assess task complexity** — categorize incoming work:
   - Simple (1-2 files, clear change) → direct user advice, skip agents
   - Medium (3-5 files) → spawn: researcher → planner → implementer → tester → reviewer
   - Complex (6+ files, multi-subsystem) → full pipeline with parallel implementers in worktrees

2. **Manage the development pipeline**:
   - **Research** phase: Spawn researcher with task description → returns context summary
   - **Plan** phase: Spawn planner with context + task → produces docs/plans/{feature}/research.md and plan.md
   - **Decompose**: Read plan.md, break into independent subtasks (parallelizable if file sets disjoint)
   - **Implement**: Spawn implementer(s), each gets context + specific subtask + owned files
   - **Test**: Spawn tester for each implementer → must pass before review
   - **Review**: Spawn reviewer with ONLY final diff (clean context)
   - **Decide**: If tests pass + reviewer approves → keep, else revert and retry (max 3 attempts)
   - **Log**: Spawn context-keeper to update CLAUDE.md, rules, results.tsv

3. **Spawn agents** — create isolated contexts:
   - Pass only what each agent needs (focused brief, relevant context, no dump)
   - Outputs are disposable; plans compound (redraw from plan if implementation fails)
   - Subagent isolation: never dump full conversation into subagent prompt

4. **Handle autoresearch** — when user says "go autonomous":
   - Confirm metric configured in CLAUDE.md
   - Kick off /autoresearch skill
   - NOT interfere once loop is running (skill owns entire experiment loop)

5. **Fallback procedures**:
   - Build breaks → revert to last good commit, retry simpler approach
   - OOM/timeout → log, reduce scope, retry once
   - Flaky test → run 3x, pass 2/3 = flaky (note and continue), 0/3 = real failure
   - Stuck loop → same experiment fails 3x with different approaches → skip, log "blocked"
   - Escalate → after 3 consecutive skips or out of ideas → write summary and stop

6. **Context management** — prevent degradation:
   - Subagent isolation: pass summaries, not raw file contents
   - Document & clear: for complex tasks near context limit, dump plan+progress to file, /clear, resume fresh
   - Fresh context per experiment: in autoresearch, spawn new agents each iteration (never accumulate)

---

## How Skills Are Defined

**Location**: `.claude/skills/{skill-name}/SKILL.md`

**Frontmatter** (optional YAML):
```yaml
---
description: {what the skill does}
---
```

**Structure**:
- Each skill is a markdown file with executable documentation
- Contains step-by-step instructions or workflow
- Referenced in framework.json under "skills" array
- Invoked via `/skill-name` command or explicitly spawned by orchestrator

**Current skills** (from framework.json):
1. **autoresearch** — autonomous experiment loop (ML training, benchmarks, optimization)
2. **eval** — run eval tasks against framework to measure agent quality
3. **plan-and-develop** — structured planning + implementation workflow
4. **qwen-review** — optional local code review via Qwen model (Ollama)

---

## How Skills Invoke Agents

**Mechanism**: Skills are orchestrated workflows that SPAWN agents at various stages.

**Autoresearch example** (from .claude/skills/autoresearch/SKILL.md):
```
LOOP FOREVER:
  1. Propose: Spawn FRESH researcher → propose experiment
  2. Implement: Spawn FRESH implementer in worktree → make change, git commit
  3. Run: Spawn FRESH tester → execute experiment (with timeout)
  4. Evaluate: Compare metric to baseline
  5. Log: Spawn context-keeper → append to results.tsv
  6. Repeat: Loop with fresh agents (never accumulate context)
```

**Plan-and-develop example** (from .claude/skills/plan-and-develop/SKILL.md):
```
Phase 1: Research
  → Use dynamic shell output + Grep/Read to explore codebase
  → Write findings to docs/plans/{feature}/research.md

Phase 2: Plan
  → Write docs/plans/{feature}/plan.md with subtasks, dependencies, tests

Phase 3: Review gate
  → For complex tasks: spawn second researcher to review plan for gaps
  → For medium tasks: self-review

Phase 4: Implement
  → Delegate to implementers (via orchestrator if in agent mode)
  → Or implement standalone in same session

Phase 5: Verify
  → Run full test suite
  → Commit the plan as documentation
```

**Key pattern**: Skills define workflows. Orchestrator spawns agents at defined checkpoints.
Each agent spawn gets fresh context to prevent degradation.

---

## Current Flow: Does Bootstrap Already Use Agents?

**Answer**: NO. Bootstrap is a COMMAND, not an agent.

**Bootstrap invocation**:
- User runs: `claude --command bootstrap` or orchestrator detects missing CLAUDE.md and runs bootstrap
- Bootstrap executes as a series of shell commands + user prompts (AskUserQuestion)
- NO agent spawning occurs during bootstrap
- Output: populated CLAUDE.md, rules, hooks verified, initial commit

**Post-bootstrap flow** (normal development):
- User runs: `claude --agent orchestrator` with a task description
- Orchestrator reads CLAUDE.md (now populated by bootstrap)
- Orchestrator spawns researcher → planner → implementers → testers → reviewers → context-keeper
- Full agent pipeline activated

**Bootstrap's role in the larger system**:
1. One-time setup command (before any agents run)
2. Creates CLAUDE.md that orchestrator reads on every session
3. Verifies hooks are executable (readonly-guard, test-gate, review-gate, etc.)
4. Sets metric config for autoresearch (optional, used later)
5. After bootstrap, users invoke orchestrator to do actual work

---

## Key Files Summary

| File | Purpose |
|------|---------|
| `.claude/commands/bootstrap.md` | Bootstrap skill definition — 13-step project setup |
| `.claude/agents/orchestrator.md` | Main coordinator — decomposes tasks, spawns agents, synthesizes results |
| `.claude/agents/researcher.md` | Deep code explorer — produces context summaries |
| `.claude/agents/planner.md` | Design planner — produces research.md and plan.md |
| `.claude/agents/implementer.md` | Code executor — implements subtasks in worktrees |
| `.claude/agents/tester.md` | Test runner — verifies changes |
| `.claude/agents/reviewer.md` | Code reviewer — evaluates diffs against principles |
| `.claude/agents/context-keeper.md` | Documentation maintainer — encodes lessons into CLAUDE.md and rules |
| `.claude/skills/autoresearch/SKILL.md` | Autonomous experiment loop — fresh agents per experiment |
| `.claude/skills/plan-and-develop/SKILL.md` | Planning workflow — research + plan + implement + verify |
| `.claude/skills/eval/SKILL.md` | Framework eval runner — measures agent quality |
| `.claude/skills/qwen-review/SKILL.md` | Optional local review via Qwen model |
| `.claude/framework.json` | Framework metadata — lists all agents and skills |
| `.claude/PRINCIPLES.md` | Non-negotiable engineering rules (all agents follow) |
