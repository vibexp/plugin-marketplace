---
name: prime
description: Prime this session with your team's knowledge from VibeXP before starting a task. Retrieves only the slice relevant to the task — full content for what the work needs now, a knowledge map for just-in-time fetching later — so work starts with everything the team already learned without flooding the context. Use at the start of any substantial task, when the user says "prime", "load context", "what do we know about this", or before working in a project that has a VibeXP knowledge base.
argument-hint: "[task description]"
---

# Prime: start with what your team already knows

Retrieve the relevant slice of the team's VibeXP knowledge base and apply it to the task at hand. The goal is a short, high-signal context brief — not a dump of everything stored. Every token pulled that the task doesn't need actively hurts: it competes with the task for the context window. Pull what the work needs *now*; index the rest for just-in-time retrieval.

Context for project detection (may be empty if this is not a git repository):

Git remote: !`git remote get-url origin 2>/dev/null || echo "(no git remote)"`
Directory: !`basename "$(pwd)"`

Task provided by the user: $ARGUMENTS

## Step 0 — Pick the VibeXP transport (CLI-first)

Follow **`${CLAUDE_PLUGIN_ROOT}/references/transport.md`**: probe with `command -v vibexp && vibexp whoami` — installed and authenticated → use the official CLI (token-cheaper; trim responses with `--format json --jq`); otherwise use the `vibexp_io_*` MCP tools (match on the `vibexp_io_` fragment, never assume an alias). Neither available → STOP and help the user connect per that reference. Steps below name operations by MCP core name; on the CLI transport use the mapped command.

## Step 1 — Establish the task and pick the retrieval tier

Use the task description from the arguments above, or infer it from the conversation. If there is no specific task, prime for general project context instead: conventions, architecture decisions, and current work in flight.

Then size the retrieval to the task:

- **Focused** (one bug, one small change, one question) → *light tier*: 1 targeted search query, no feed scan, fetch at most 1–2 items in full.
- **Substantial** (a feature, a refactor, multi-phase or long-running work) → *full tier*: Steps 3–4 as written.

## Step 2 — Resolve team and project

Resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams_and_projects` queried by **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's.

No match → list what exists and ask (use one / search team-wide unfiltered). Never silently pick an unrelated project.

## Step 3 — Retrieve relevant knowledge

Run `search` (semantic, cross-entity) scoped to the project. Light tier: just query 1. Full tier: 2–3 complementary queries:

1. The task description itself.
2. Conventions/rules angle: e.g. "coding standards, conventions, and rules for <project or area>".
3. History angle: e.g. "past decisions, gotchas, and lessons about <key nouns from the task>".

Results are relevance-ranked with a `score` in [0,1] and are excerpts only. Collect the hits, dedupe by `id`, and keep the genuinely relevant ones (roughly: score ≥ 0.5, or the top 5–8 overall).

Scan the feed **only when collision is plausible** — the task could overlap work in flight (a shared area, an active team, a long-running change). Then: `list_feeds` → `list_feed_items` on the most relevant feed (first page is enough; full content via `get_feed_item` only if an item is clearly about this task). A solo repo or an isolated fix doesn't need the scan.

## Step 4 — Fetch full content for what the task needs *now*

Search returns ~300-char excerpts; fetch the full text of what the task will rely on immediately — **at most 3–5 fetches** (light tier: 1–2). Everything else relevant goes on the knowledge map (Step 5), not into the context.

- `memory` results → get by `id`; `artifact` results → get by `slug`.
- `blueprint` and `prompt` results: on the **CLI transport**, fetch in full (`vibexp blueprint get <slug>` / `vibexp prompt get <slug>`). Over **MCP** there is no get tool — work from the excerpts, and if a blueprint looks central, run one more targeted `search` using its title to surface additional chunks of it.

Skip low-value fetches; every fetch must earn its place in the brief.

## Step 5 — Brief + knowledge map, then work

Present a short context brief — a *distillation*, never pasted resource bodies:

- **Rules to follow** — from blueprints/memories (conventions, constraints).
- **Relevant history** — decisions and gotchas that change how the task should be done.
- **Related work** — artifacts or recent feed activity to build on rather than redo.
- **Knowledge map** — the relevant hits *not* fetched: one line each (type, id/slug, title, why it might matter later).

Then do the task, applying what was retrieved — and pull the rest **just-in-time**:

- When work enters an area on the knowledge map, fetch that resource *then* — one `get` call, no new search.
- On long multi-phase work, when a phase touches ground the brief didn't cover, run one targeted `search` for that phase instead of having front-loaded everything. (`/vibexp:report`'s `check` mode does this at each phase boundary.)

If retrieval found nothing relevant, say so plainly — the project's knowledge base may be cold — and recommend running `/vibexp:wrap` at the end of the session so the next prime has something to find.

## Step 6 — Write the session ledger

Record what this prime did per **`${CLAUDE_PLUGIN_ROOT}/references/session-ledger.md`**: overwrite `vibexp-last-prime.md` with the date, task, transport, resolved ids, the resources fetched in full (*Relied on*), the knowledge map, and the queries run. `/vibexp:wrap` audits this for staleness at session end. Unwritable → continue without it, never fail the skill.

## Conventions (apply throughout)

- Every operation except `get_user` and `list_teams_and_projects` requires `team_id` (UUID or slug).
- List/search return excerpts or truncated text by design; `get` returns full content. Don't treat an excerpt as the whole document.
- List endpoints cap at 10 items per page; `search` allows `limit` up to 100 (default 10).
- Token budget: the brief should be a small fraction of the context window. When in doubt between fetching and mapping, map — the knowledge map makes deferred context one call away.
- Read-only skill: prime never creates, updates, or deletes anything in VibeXP (the local session ledger is the only thing it writes).
