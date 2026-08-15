# Resolving the VibeXP team + project

Canonical procedure for every VibeXP skill. Nothing is hardcoded: no default
team, no default project. **The repo's git URL is the only identity that
decides** — a user belongs to several teams and the owning team is not guessable
from the org (`github.com/shaharia-lab/*` and `github.com/vibexp/*` sit under
different teams). Results are cached, so repeat sessions cost nothing.

API constraints: `list_projects` **requires** `team_id` (no cross-team search →
iterate teams); projects carry a canonical `git_url` and are usually named
`owner/repo`; `search` matches **name/description only**, max 10 per page.
So `search` narrows, `git_url` decides.

## 1 · Identity

`git remote get-url origin` → `KEY = host/owner/repo` (drop scheme, leading
`git@`, trailing `.git`; compare case-insensitively). Keep `OWNER`, `REPO`.
No remote → step 4 with no candidate; ask.

## 2 · Cache (0 API calls)

```bash
MEM="$HOME/.claude/projects/$(printf '%s' "$PWD" | tr '/' '-')/memory"
cat "$MEM/vibexp-scope.md" 2>/dev/null
```

`git_url` matches `KEY` → use the cached ids, done. Mismatch → ignore it and
resolve fresh; a mapping for another repo is never a fallback.

## 3 · Teams

`list_teams`. One → use it. Several → **order** them: slug/name resembling
`OWNER` first, then the rest. Ordering picks who is *queried first*, never what
is *used*.

## 4 · Project by `git_url`

Per team, in order: `list_projects(team_id, search=REPO)`, normalize each
`git_url`, compare to `KEY`. **First exact match wins — stop.**

Nothing matched → retry with `search=OWNER`, then paginate full project lists
(`page` → `total_pages`). Matching on name/slug is a last resort needing user
confirmation.

Outcomes: one match → use it · same `git_url` in several teams → ask once ·
none → report what exists and stop. Projects can't be created over MCP; the user
creates one in the web app with `git_url` set, then re-run `list_projects`.

## 5 · Cache the result

Write `$MEM/vibexp-scope.md` (`mkdir -p "$MEM"` first; if that fails, continue
uncached — never fail the skill over the cache):

```markdown
---
name: vibexp-scope
description: VibeXP team + project ids for this repo — reuse instead of re-resolving
metadata:
  type: project
---

`github.com/<owner>/<repo>` → team **<Name>** (`<team-uuid>`, slug `<team-slug>`)
· project **<owner>/<repo>** (`<project-uuid>`, slug `<project-slug>`).
Matched on the project's `git_url`, <YYYY-MM-DD>. Re-resolve only if a call
rejects these ids or the remote changes.
```

Add to `MEMORY.md`: `- [VibeXP scope](vibexp-scope.md) — team + project ids for this repo`

## 6 · Invalidate

Cached ids rejected (not-found/forbidden) → discard, resolve once from step 3,
rewrite. Don't loop.

**Cost:** cache hit 0 calls · cold 1 `list_teams` + typically 1 `list_projects`.
