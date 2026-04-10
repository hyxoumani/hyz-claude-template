# Researcher Memory

This directory holds findings from research phases conducted by researcher agents.

## What goes here

- **Architecture decisions**: Design rationale for key choices
- **Patterns discovered**: Structural insights about the codebase
- **Constraints identified**: Things that can't change and why
- **Gotchas/blockers**: Common pitfalls and non-obvious behaviors

## File structure

Each research topic gets its own file: `{topic}.md`

Use standard memory frontmatter:

```markdown
---
name: Topic Name
description: One-line summary for future relevance decisions
type: research
---

Content...
```

## Not here

- Implementation details or code snippets (those stay in the codebase)
- Session-ephemeral work notes (those stay in conversation memory)
- Debugging solutions (those belong in commit messages)

## Audience

These memories inform future research phases and help agents understand why the system works the way it does.
