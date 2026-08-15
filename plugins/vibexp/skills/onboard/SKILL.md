---
name: onboard
description: Bootstrap a repository's VibeXP knowledge base so the knowledge loop has something to start from. Imports the repo's existing AI configuration (CLAUDE.md, .cursorrules, AGENTS.md, .cursor/, .claude/, copilot instructions) as per-tool blueprints, and seeds a small set of high-value memories (architecture overview, dev workflow, unwritten conventions). Use on a project new to VibeXP, when /vibexp:prime finds an empty knowledge base, or when the user says "onboard", "set up vibexp for this repo", or "import our AI config".
argument-hint: "[optional: which VibeXP project to onboard into]"
---

# Onboard: solve the cold start

The knowledge loop only pays off once there's something to retrieve. This skill seeds a new project's knowledge base from what the repo already knows about itself.

Target project (may be empty — auto-detect): $ARGUMENTS

Git remote: !`git remote get-url origin 2>/dev/null || echo "(no git remote)"`

## Step 0 — Pick the VibeXP transport (CLI-first)

Follow **`${CLAUDE_PLUGIN_ROOT}/references/transport.md`**: probe with `command -v vibexp && vibexp whoami` — installed and authenticated → use the official CLI (note: `vibexp blueprint get` reads full blueprint content, which MCP cannot — it makes the Step 3 dedup much sharper); otherwise use the `vibexp_io_*` MCP tools (match on `vibexp_io_`, never assume an alias). Neither available → STOP and help the user connect per that reference. Steps below name operations by MCP tool name; on the CLI transport use the mapped command.

## Step 1 — Resolve team and project

1. Resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams` → `list_projects` matched on **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's.
2. A project named in **Target project** above overrides the match.
3. **No matching project?** Projects cannot be created over MCP. Tell the user to create it in the VibeXP web app (New Project — set the `git_url` to this repo so future auto-detection works; the GitHub import option can also bring in AI config automatically, which overlaps with Step 2 — if they use it, this skill's blueprint import becomes a dedup pass rather than a first import). Wait for them to confirm, then re-run `vibexp_io_list_projects`.

## Step 2 — Inventory the repo's AI configuration

Look for the files AI tools already read, at the repo root and standard locations:

- `CLAUDE.md`, `CLAUDE.local.md`, `.claude/` (agents, rules, commands worth sharing)
- `.cursorrules`, `.cursor/rules/*`
- `AGENTS.md`, `.agents/`
- `.github/copilot-instructions.md`
- `GEMINI.md`, `.gemini/`, `.codex/`

Also note the repo's self-description sources: `README.md`, `CONTRIBUTING.md`, `docs/` architecture pages, top-level `Makefile`/`package.json` scripts.

## Step 3 — Dedup against what's already in VibeXP

Before importing anything, check what exists: `vibexp_io_search` scoped to the project with `types: ["blueprints", "memories"]` using each candidate file's key topics (blueprints have no MCP get tool, so judge from excerpts and titles). Anything already imported — by a teammate, a previous run, or VibeXP's native GitHub import — is skipped or proposed as an update, never duplicated.

## Step 4 — Propose the onboarding plan, get one confirmation

Present a compact plan and confirm once before writing:

- **Blueprints** — one per config file, e.g.: `CLAUDE.md` → slug `claude-md`, title "Claude Code instructions", metadata `{"tool": "claude-code", "source_path": "CLAUDE.md"}`. Content imported as-is (these files are already written as AI instructions — don't rewrite them), `type: "general"`, `status: "active"`.
- **Seed memories** — a *small* set (typically 2–4) of durable, high-value facts distilled from the repo, each self-contained: an architecture overview (what the system is, its main components, how they talk), the dev workflow (build/test/run commands that actually work), and any conventions you observed in the code that no config file states. Skip anything a config-file blueprint already covers — memory is for knowledge, not copies.
- **Typed edges** (only when a `vibexp_io_link_resources` tool is available — otherwise omit this section from the plan): each seed memory `governed-by` the imported blueprint that states the rule it distills (only when a specific blueprint really states it); a new blueprint that replaces a *different* existing one → `supersedes`. Updating the same blueprint needs no edge — the server versions content, and self-links are rejected.

Quality bar over completeness: onboarding should leave the knowledge base *worth priming from*, not stuffed.

## Step 5 — Execute

- Blueprints: `vibexp_io_create_blueprint` (or `vibexp_io_update_blueprint` for agreed updates — located by `project_id` + `slug`).
- Memories: `vibexp_io_create_memory` with `status: "active"` and metadata like `{"category": "architecture", "source": "onboarding"}`.
- Edges: after the writes, record the planned links via `vibexp_io_link_resources` (the project's `project_id`, each end's type + UUID from the create/update responses). No such tool on this server → skip silently, never fail the onboarding over it.

## Step 6 — Report and hand off to the loop

Summarize what was imported (blueprints with slugs, memories created, typed edges recorded — created / already existed / skipped when the tool is unavailable) with links where returned. Then point the user at the loop these seeds feed:

- `/vibexp:prime` at the start of the next task — it will now find something.
- `/vibexp:wrap` at the end of sessions — the base grows from real work.
- `/vibexp:consolidate` periodically — the base stays sharp.

**Keep sources in sync:** the imported blueprints are snapshots. When `CLAUDE.md` (or another source file) changes meaningfully in the repo, update the corresponding blueprint (`vibexp_io_update_blueprint` — the server keeps version history), or re-run this skill for a dedup-aware refresh.

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams` requires `team_id` (UUID or slug).
- Import instructions verbatim; distill knowledge selectively. Blueprints mirror the repo's config files; memories add what's written nowhere.
- Never import secrets: if a config file embeds tokens, keys, or internal URLs that shouldn't be team-visible, flag the lines and exclude them from the imported blueprint.
- Typed edges (`vibexp_io_link_resources`): `governed-by` → object must be a blueprint; `built-from` → object must be a prompt; `explained-by` → object must be a memory; `supersedes` → both ends the same type. No self- or cross-project links; re-linking an existing edge is a safe no-op. Tool not available (older server) → skip linking silently, never fail the skill over it.
