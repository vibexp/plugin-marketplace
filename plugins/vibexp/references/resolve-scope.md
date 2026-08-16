# Resolving the VibeXP team + project

Canonical procedure for every VibeXP skill. Nothing is hardcoded: no default
team, no default project. **The repo's git URL is the only identity that
decides** — a user belongs to several teams and the owning team is not guessable
from the org (`github.com/shaharia-lab/*` and `github.com/vibexp/*` sit under
different teams). Results are cached, so repeat sessions cost nothing.

API surface: `list_teams_and_projects` is the one discovery operation — with no
arguments it returns your teams with project counts, with `query` it searches
team **and** project names, slugs, descriptions **and project `git_url`** across
every team at once, and with `team_id` alone it lists that team's projects.
(It replaced `list_teams` + `list_projects` in server v0.11.0; both survive as
deprecated aliases for one release — don't call them. The CLI still ships the
two separate commands, so the CLI path below differs by design.)

Projects carry a canonical `git_url` and are usually named `owner/repo`.
The query narrows, `git_url` decides.

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

## 3 · Project by `git_url` — one call (MCP)

`list_teams_and_projects(query=URL)` where `URL` is the **normalized HTTPS form,
no `.git`** (`https://github.com/<owner>/<repo>`) — that is the form projects
store. The response nests matching projects under their team, each with a
`score`.

**Accept only `score: 1.0` on a URL query.** Ranking runs as a ladder — exact
(`slug` / `id` / `git_url`, score 1.0), then full-text, then trigram — and only
one rung runs per query. So on a URL query 1.0 can only be a `git_url`
equality (no slug is a URL), and **anything below 1.0 proves the exact rung
found nothing** — every such hit is a name/description near-miss, not your
project. Note the nested project DTO omits `git_url`, so the score *is* the
match evidence; there is nothing else to compare.

One 1.0 hit → use its team `uuid` + project `id`. Otherwise → step 4.

## 4 · No exact hit

In order: re-query with the remote's other stored forms (`.git` suffix, `git@`
SSH form) → `query=OWNER/REPO` → `query=REPO`. These return **candidates only**.
Confirm one with the user before using it, or confirm its `git_url` on the CLI
(`vibexp project list --team <uuid>` does return `git_url`). Never adopt a
sub-1.0 hit silently.

**CLI path** — the CLI has no merged command, so sweep teams and compare
`git_url` directly (bash and zsh; `--limit 100` avoids paging):

```bash
KEY=$(git remote get-url origin)
for T in $(vibexp team list --format json --limit 100 \
  | python3 -c 'import sys,json;[print(t["id"]) for t in json.load(sys.stdin)["teams"]]'); do
  vibexp project list --team "$T" --format json --limit 100 2>/dev/null \
  | python3 -c '
import sys,json
key=sys.argv[1].rstrip("/").removesuffix(".git").lower()
for p in json.load(sys.stdin).get("projects",[]):
    if (p.get("git_url") or "").rstrip("/").removesuffix(".git").lower()==key:
        print("team_id=%s project_id=%s slug=%s"%(p["team_id"],p["id"],p["slug"]))
' "$KEY"
done
```

Outcomes: one match → use it · same `git_url` in several teams → ask once ·
none → report what exists and stop. Projects can't be created over MCP; the user
creates one in the web app with `git_url` set, then re-run the resolve.

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

**Cost:** cache hit 0 calls · cold 1 `list_teams_and_projects` on MCP.

## Traps

- **Near-miss projects are everywhere — only an exact `git_url` decides.** Real
  case: `VibeXP Team` holds both `vibexp-vibexp` (`github.com/vibexp/vibexp`)
  and a decoy `shaharia-lab-vibexp-io` whose `git_url` is the *pre-rename*
  `github.com/shaharia-lab/vibexp.io`. Same team, near-identical names — name
  similarity, org name, or "the obvious team" lands on the decoy, whose
  knowledge base is empty.
- **A repo may have no project at all, and the search still answers
  confidently.** Before `vibexp/plugin-marketplace` existed, querying its URL
  returned `shaharia-lab/claude-plugin-marketplace` at `score 0.44` — a
  plausible-looking hit for a *different repo in a different org*. This is why
  sub-1.0 is never accepted: "no project exists yet" and "here is your project"
  are indistinguishable except by the score.
- **Trailing `.git` breaks the exact rung** — measured, same repo, same minute:

  | query | result |
  |---|---|
  | `…/vibexp/plugin-marketplace` | `vibexp/plugin-marketplace` **1.0** — resolved |
  | `…/vibexp/plugin-marketplace.git` | `shaharia-lab/claude-plugin-marketplace` 0.43 **and** `vibexp/plugin-marketplace` 0.59 |

  The exact match is literal string equality against the stored
  `https://github.com/<owner>/<repo>`, so `git remote get-url` output passed
  verbatim drops to trigram — where the wrong project is listed *first* (teams
  come back in their own order, not by score). Normalize before querying.
- **`score: 1.0` only proves exact identity on a URL query.** The trigram rung
  also returns 1.0 when the query is contained whole in a project name
  (`query=plugin-marketplace` → `shaharia-lab/claude-plugin-marketplace` at
  1.0). Query the full URL and the ambiguity disappears.
- **Teams come back with no `projects` key when only the *team* matched** the
  query. That is a team-name hit, not a project hit — it resolves nothing.
- **Symptom of a wrong scope:** several *different* queries all returning zero
  results. That reads as a cold knowledge base but usually means the wrong
  project — re-verify scope before concluding the base is empty.
- **Never preview a list with `head -c`/`head -n`.** `vibexp team list` returns
  one large JSON array; truncating it silently drops teams and produces a wrong
  answer that looks complete. Project the identifying fields instead:
  ```bash
  vibexp team list --format json \
    | python3 -c 'import sys,json;[print(t["id"],"|",t["slug"],"|",t["name"]) for t in json.load(sys.stdin)["teams"]]'
  ```
- **The CLI needs UUIDs, not slugs**, despite `--help` saying "id or slug":
  `--team <slug>` fails `team_id must be a valid UUID` and `search --project
  <slug>` fails the `uuid` tag. Resolve slug → UUID first, pass UUIDs
  throughout. (Asymmetry: the MCP tools accept either for `team_id`.)
- **`vibexp project list` requires an explicit `--team <uuid>`** unless a
  default context is set — it does not sweep teams for you.
