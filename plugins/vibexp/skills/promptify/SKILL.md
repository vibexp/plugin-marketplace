---
name: promptify
description: Turn a prompt that worked into a reusable VibeXP team prompt. Generalizes an ad-hoc instruction from this session (or provided text) into a template with {{variables}}, factors shared boilerplate into @slug-referenced base prompts, checks for existing similar prompts to update instead of duplicating, and publishes it — optionally MCP-exposed as a native slash command in every teammate's AI tool — behind one confirmation. Use when a prompt produced a great result worth keeping, or when the user says "promptify", "save this prompt", "make this reusable", or "add this to our prompt library".
argument-hint: "[the prompt text, or a description of which instruction from this session to capture]"
---

# Promptify: from one-off prompt to team asset

A prompt that worked once and vanished into scrollback is exactly the waste VibeXP exists to stop. This skill captures it, generalizes it, and publishes it to the team library.

What to capture: $ARGUMENTS

## Step 0 — Pick the VibeXP transport (CLI-first)

Follow **`${CLAUDE_PLUGIN_ROOT}/references/transport.md`**: probe with `command -v vibexp && vibexp whoami` — installed and authenticated → use the official CLI (note: `vibexp prompt get` reads full prompt content, which MCP cannot — use it in Step 2 before proposing an update); otherwise use the `vibexp_io_*` MCP tools (match on `vibexp_io_`, never assume an alias). Neither available → STOP and offer to run `/vibexp:install` (guided CLI install + sign-in), or the MCP route, per that reference. Steps below name operations by MCP core name; on the CLI transport use the mapped command.

Resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams_and_projects` queried by **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's. Prompts require a `project_id`.

## Step 1 — Identify the source prompt

From the arguments, or find it in the session: the instruction pattern that produced the result the user wants to keep. If several candidates exist, ask which one. Capture the *pattern*, not the transcript — the instruction, its structure, and what made it work.

## Step 2 — Check the library first

Search before creating: `search` with `types: ["prompts"]` using the prompt's purpose as the query.

- A prompt for the same job already exists → propose improving it via `update_prompt` (located by `slug`) instead of adding a near-duplicate. The server versions prompt content on update. (If that prompt is a `draft`, it is someone's work-in-progress — point the user to it instead of editing it.)
- Nothing similar → proceed to create.

## Step 3 — Generalize into a template

Transform the one-off into a reusable template:

1. **Strip session-specifics** — file names, ticket numbers, one-time context that won't apply next time.
2. **Parameterize** the parts that vary with `{{variable}}` placeholders: double curly braces, `lowercase_underscore` names, self-documenting (`{{target_audience}}`, not `{{x}}`). No nesting, no defaults — unresolved placeholders are left verbatim at render time. Each variable becomes a **required argument** when the prompt is MCP-exposed, so only parameterize what genuinely varies.
3. **Factor shared boilerplate** — if part of the prompt is generic instruction the team repeats (tone rules, output format, review standards), reference a base prompt with `@base-prompt-slug` instead of inlining it. Search for an existing base prompt first; if a clearly reusable block has no base prompt yet, propose creating one alongside. References resolve recursively at render time; a literal `@` must be escaped as `@@`.
4. **Metadata**: `name` ≤ 50 chars (human-readable), `slug` ≤ 255 kebab-case (stable — it's the `@` reference handle and MCP identity), `description` ≤ 200 chars saying when to use it, up to 10 `labels` for discovery.

## Step 4 — Propose and confirm once (including how far to publish)

Show the user the finished template (body with its variables and references) plus name/slug/description/labels, and in the **same** confirmation ask how far to take it:

- **Published + MCP-exposed** (recommended for prompts the team invokes directly) — `status: "published"`, `mcp_expose: true`: the prompt becomes a **native MCP prompt** in every connected tool — in Claude Code it appears as a slash command with each `{{variable}}` as an argument. This is the full payoff.
- **Published** (recommended for base prompts that exist only to be `@`-referenced) — `status: "published"`, `mcp_expose: false`: the team standard, composable, not a slash command.

Draft status is humans' work-in-progress, not an agent waypoint: if the user isn't ready to publish, the prompt isn't ready to save — leave it in the conversation for them to refine (or for them to save as a draft themselves in the app).

## Step 5 — Create or update at the chosen status

`create_prompt` (or `update_prompt` by `slug` for the improve-existing case from Step 2) with the confirmed `status` and `mcp_expose` in the same call. On the CLI, `--mcp-expose` **defaults to true** — pass `--mcp-expose=false` explicitly for a base prompt.

Then, both optional and only when the tool exists on this server (skip silently otherwise):

- **Link**: if `link_resources` is available, link the prompt `governed-by` any blueprint whose rules it must obey (the object must be a blueprint; IDs come from search/list results — linking needs only ID + type). `@slug` base-prompt references are composition, not production — note them as future `built-from` candidates, but don't create edges for them.
- **Lock** a base prompt other prompts `@`-reference: renaming or deleting it would break every reference, which is exactly what `lock_resource` (`resource_type: "prompt"`, `slug`, and a `comment` saying why, e.g. "Referenced as @tone-rules by other prompts") prevents — content stays editable, and only a human can unlock it. Offer it as part of the Step 4 plan rather than doing it unasked. MCP-only (no CLI command) — use the per-call fallback in `transport.md`.

## Step 6 — Report

Confirm what was created or updated: name, slug, status, whether MCP-exposed, whether locked, any typed edges recorded (created / already existed / skipped — tool unavailable), and how teammates use it (`@slug` inside other prompts; the slash command name if exposed — teammates may need to reconnect/refresh their MCP session to see a newly exposed prompt).

## Conventions (apply throughout)

- Every operation except `get_user` and `list_teams_and_projects` requires `team_id` (UUID or slug; the CLI's `--team` takes the UUID).
- Never create or modify a `draft` prompt — drafts belong to the humans editing them.
- Slugs are identity: pick them like API names — stable, descriptive, kebab-case. Renaming a slug later breaks `@` references to it.
- Improve existing prompts over creating variants; the library compounds by getting sharper, not longer.
- Typed edges (`link_resources`): `governed-by` → object must be a blueprint; `built-from` → object must be a prompt; `explained-by` → object must be a memory; `supersedes` → both ends the same type. No self- or cross-project links; re-linking an existing edge is a safe no-op. Tool not available (older server) → skip linking silently, never fail the skill over it.
