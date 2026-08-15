---
name: prime
description: Prime this session with your team's knowledge from VibeXP before starting a task. Retrieves the relevant memories, blueprints (standing rules), artifacts, and recent team activity for the current project over the VibeXP MCP, so work starts with everything the team already learned instead of from scratch. Use at the start of any substantial task, when the user says "prime", "load context", "what do we know about this", or before working in a project that has a VibeXP knowledge base.
argument-hint: "[task description]"
---

# Prime: start with what your team already knows

Retrieve the relevant slice of the team's VibeXP knowledge base and apply it to the task at hand. The goal is a short, high-signal context brief — not a dump of everything stored.

Context for project detection (may be empty if this is not a git repository):

Git remote: !`git remote get-url origin 2>/dev/null || echo "(no git remote)"`
Directory: !`basename "$(pwd)"`

Task provided by the user: $ARGUMENTS

## Step 0 — Check the VibeXP MCP connection

All VibeXP tools are named `vibexp_io_*` (the full tool name is prefixed with the MCP server alias the user chose, e.g. `mcp__vibexp__vibexp_io_list_teams` — match on the `vibexp_io_` part, never assume a specific alias).

If no `vibexp_io_*` tools are available in this session, STOP and help the user connect instead of guessing:

```
claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common
```

For the hosted instance the URL is `https://connect.vibexp.io/mcp/v1/common`; self-hosters use their own deployment's origin. Authentication is OAuth in the browser on first use — no API key. Full instructions: https://docs.vibexp.io — then ask the user to restart the session or run `/mcp` to authenticate, and re-run this skill.

## Step 1 — Establish the task

Use the task description from the arguments above, or infer it from the conversation. If there is no specific task, prime for general project context instead: conventions, architecture decisions, and current work in flight.

## Step 2 — Resolve team and project

Resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams` → `list_projects` matched on **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's.

No match → list what exists and ask (use one / search team-wide unfiltered). Never silently pick an unrelated project.

## Step 3 — Retrieve relevant knowledge

Run `vibexp_io_search` (semantic, cross-entity) with 2–3 complementary queries, passing `project_id` when resolved:

1. The task description itself.
2. Conventions/rules angle: e.g. "coding standards, conventions, and rules for <project or area>".
3. History angle: e.g. "past decisions, gotchas, and lessons about <key nouns from the task>".

Results are relevance-ranked with a `score` in [0,1] and are excerpts only. Collect the hits, dedupe by `id`, and keep the genuinely relevant ones (roughly: score ≥ 0.5, or the top 5–8 overall).

Also glance at recent team activity so you don't duplicate or collide with work in flight: `vibexp_io_list_feeds`, then `vibexp_io_list_feed_items` on the most relevant feed (first page is enough; full content via `vibexp_io_get_feed_item` only if an item is clearly about this task).

## Step 4 — Fetch full content for the top hits

Search returns ~300-char excerpts; always fetch the full text of what you intend to rely on:

- `memory` results → `vibexp_io_get_memory`
- `artifact` results → `vibexp_io_get_artifact` (needs `project_id` + `slug` from the result)
- `blueprint` and `prompt` results have **no MCP get tool** — work from the search excerpts, and if a blueprint looks central, run one more targeted `vibexp_io_search` using its title to surface additional chunks of it.

Skip low-value fetches; every fetch should earn its place in the brief.

## Step 5 — Brief, then work

Present a short context brief to the user before starting the task:

- **Rules to follow** — from blueprints/memories (conventions, constraints).
- **Relevant history** — decisions and gotchas that change how the task should be done.
- **Related work** — artifacts or recent feed activity to build on rather than redo.

Then do the task, actually applying what was retrieved. If retrieval found nothing relevant, say so plainly — the project's knowledge base may be cold — and recommend running `/vibexp:wrap` at the end of the session so the next prime has something to find.

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams` requires `team_id` (UUID or slug).
- List/search tools return excerpts or truncated text by design; `get_*` tools return full content. Don't treat an excerpt as the whole document.
- List endpoints cap at 10 items per page; `vibexp_io_search` allows `limit` up to 100 (default 10).
- Read-only skill: prime never creates, updates, or deletes anything in VibeXP.
