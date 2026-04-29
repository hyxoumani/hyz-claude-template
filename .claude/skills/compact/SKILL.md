---
name: compact
description: Synthesize session findings, recent changes, and reviewer feedback into the project wiki at docs/wiki/. Use at end of session, before /clear, or periodically as hygiene to ensure knowledge compounds across sessions.
---

# /compact

Drains accumulated knowledge into `docs/wiki/`. Wiki pages are synthesized understanding — not changelogs, not appended notes. Each page reads as current truth.

## Step 1: Gather what's unmerged

```bash
ls -t docs/sessions/*.md 2>/dev/null | head -10   # recent session summaries
git log --oneline -20                              # recent activity
ls docs/wiki/*.md 2>/dev/null                      # current wiki state
```

Read recent session summaries and `docs/wiki/index.md`. Identify what's new since the last compact.

## Step 2: Map findings to wiki pages

| Type of finding | Wiki destination |
|---|---|
| Architecture decision | `docs/wiki/{subsystem}.md` → `## Key decisions` |
| Pattern discovery | `docs/wiki/patterns.md` or domain-specific page |
| Constraint identified | `docs/wiki/{domain}.md` → `## Gotchas` |
| Bug rationale / fix | `docs/wiki/{subsystem}.md` → `## Gotchas` |
| Reviewer recurring issue | `docs/wiki/{domain}.md` → `## Gotchas` |

## Step 3: Update or create pages

**New page** (`docs/wiki/{topic}.md`, kebab-case):

```markdown
# {Topic}

{Synthesized understanding — what's true now, not how we got here.}

## Key decisions
- {Decision}: {why, what alternatives were rejected}

## Gotchas
- {Non-obvious issue}: {how to avoid}

## Related
- [{other-topic}](other-topic.md) — {relationship}
```

**Existing page**: rewrite sections to incorporate new knowledge. Don't append — synthesize.

Constraints:
- Each page ≤100 lines. Split if larger.
- Every page has a `## Related` section.
- When updating a page, check pages it links to. Update them if affected.

## Step 4: Update the index

Update `docs/wiki/index.md` with any new pages, grouped by domain (not alphabetically).

## Step 5: Lint pass

- `CLAUDE.md` still under 150 lines? Commands still correct?
- Wiki pages over 100 lines? Split.
- Orphan pages not linked from index or any other page?
- `> CONTRADICTION:` markers that can now be resolved?

Fix issues in place.

## Step 6: Log

Append one line to `docs/log.md`:

```
YYYY-MM-DD — Compact: {summary of what was merged/created/cleaned}
```

## Report

Tell the user:
- Wiki pages created or updated
- Contradictions resolved (or flagged)
- Stale content cleaned
- Open questions needing user input
