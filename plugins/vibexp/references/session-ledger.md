# The session ledger: what prime pulled, so wrap can audit it

`/vibexp:prime` records what it retrieved in a small local file; `/vibexp:wrap`
reads it to audit that knowledge for staleness, and any mid-task step reads its
knowledge map to fetch context just-in-time. It survives context compaction and
lets wrap work even when prime ran much earlier. Local only — never written to
VibeXP.

## Location

Same directory as the scope cache (`resolve-scope.md`):

```bash
MEM="$HOME/.claude/projects/$(printf '%s' "$PWD" | tr '/' '-')/memory"
```

File: `$MEM/vibexp-last-prime.md`. Prime **overwrites** it on every run (one
ledger = the latest prime). If the directory isn't writable, continue without a
ledger — never fail the skill over it.

## Format

```markdown
---
name: vibexp-last-prime
description: What the last /vibexp:prime retrieved — wrap audits it for staleness; the knowledge map feeds just-in-time fetches
metadata:
  type: project
---

Primed <YYYY-MM-DD> · task: <one line> · transport: cli|mcp
· team <team-id> · project <project-id>

## Relied on (fetched full)
- memory <id> — <title or one-line gist>
- artifact <slug> — <title>

## Knowledge map (not fetched — pull just-in-time when work enters the area)
- blueprint <slug> — <title> — <why it might matter>
- memory <id> — <title> — <why it might matter>

## Queries run
- "<query 1>"
- "<query 2>"
```

Add to `MEMORY.md` (once): `- [Last prime](vibexp-last-prime.md) — what the last prime pulled; wrap audits it`

## Rules

- **Prime writes it** at the end of its run — every resource it fully fetched
  under *Relied on*, every relevant-but-unfetched hit under *Knowledge map*.
- **Wrap reads it** to run the staleness audit: for each *Relied on* entry, did
  this session's work contradict or outdate it? Check the ledger's date first —
  if it predates this session (a leftover from an earlier day), audit only the
  entries the current conversation actually used.
- **Mid-task JIT**: when work enters an area the brief didn't cover, check the
  knowledge map before searching from scratch — a mapped id/slug is one `get`
  call, not a new search.
- The ledger stores pointers and one-liners, never resource bodies.
