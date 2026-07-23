---
name: promptify
description: Turn a prompt that worked into a reusable VibeXP team prompt. Generalizes an ad-hoc instruction from this session (or provided text) into a template with {{variables}}, factors shared boilerplate into @slug-referenced base prompts, checks for existing similar prompts to update instead of duplicating, and walks the draft → publish → MCP-expose path so it becomes available in every teammate's AI tool. Use when a prompt produced a great result worth keeping, or when the user says "promptify", "save this prompt", "make this reusable", or "add this to our prompt library".
argument-hint: "[the prompt text, or a description of which instruction from this session to capture]"
---

# Promptify: from one-off prompt to team asset

A prompt that worked once and vanished into scrollback is exactly the waste VibeXP exists to stop. This skill captures it, generalizes it, and publishes it to the team library.

What to capture: $ARGUMENTS

## Step 0 — Check the VibeXP MCP connection

VibeXP tools are named `vibexp_io_*` (prefixed with the user's MCP server alias — match on `vibexp_io_`, never assume an alias). If none are available, STOP and help the user connect:

```
claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common
```

(Hosted instance: `https://connect.vibexp.io/mcp/v1/common`; self-hosters use their own origin. OAuth in the browser, no API key. Docs: https://docs.vibexp.io)

Resolve team and project: `vibexp_io_list_teams` (one → use it; several → match this repo or ask once), `vibexp_io_list_projects` matched against the git remote. Prompts require a `project_id`.

## Step 1 — Identify the source prompt

From the arguments, or find it in the session: the instruction pattern that produced the result the user wants to keep. If several candidates exist, ask which one. Capture the *pattern*, not the transcript — the instruction, its structure, and what made it work.

## Step 2 — Check the library first

Search before creating: `vibexp_io_search` with `types: ["prompts"]` using the prompt's purpose as the query.

- A prompt for the same job already exists → propose improving it via `vibexp_io_update_prompt` (located by `slug`) instead of adding a near-duplicate. The server versions prompt content on update.
- Nothing similar → proceed to create.

## Step 3 — Generalize into a template

Transform the one-off into a reusable template:

1. **Strip session-specifics** — file names, ticket numbers, one-time context that won't apply next time.
2. **Parameterize** the parts that vary with `{{variable}}` placeholders: double curly braces, `lowercase_underscore` names, self-documenting (`{{target_audience}}`, not `{{x}}`). No nesting, no defaults — unresolved placeholders are left verbatim at render time. Each variable becomes a **required argument** when the prompt is MCP-exposed, so only parameterize what genuinely varies.
3. **Factor shared boilerplate** — if part of the prompt is generic instruction the team repeats (tone rules, output format, review standards), reference a base prompt with `@base-prompt-slug` instead of inlining it. Search for an existing base prompt first; if a clearly reusable block has no base prompt yet, propose creating one alongside. References resolve recursively at render time; a literal `@` must be escaped as `@@`.
4. **Metadata**: `name` ≤ 50 chars (human-readable), `slug` ≤ 255 kebab-case (stable — it's the `@` reference handle and MCP identity), `description` ≤ 200 chars saying when to use it, up to 10 `labels` for discovery.

## Step 4 — Propose, confirm, create as draft

Show the user the finished template (body with its variables and references) plus name/slug/description/labels, and get one confirmation. Then `vibexp_io_create_prompt` with `status: "draft"`.

## Step 5 — Publish and expose

Ask the user how far to take it:

- **Draft** — stays in the library, editable, not yet part of the team's active set.
- **Published** — `vibexp_io_update_prompt` with `status: "published"`: the team standard.
- **Published + MCP-exposed** — additionally `mcp_expose: true` (only valid on published prompts): the prompt becomes a **native MCP prompt** in every connected tool — in Claude Code it appears as a slash command with each `{{variable}}` as an argument. This is the full payoff; recommend it for prompts the team will invoke directly, skip it for base prompts that exist only to be `@`-referenced.

After creating or updating the prompt, if a `vibexp_io_link_resources` tool is available, link it `governed-by` any blueprint whose rules it must obey (the object must be a blueprint; IDs come from search/list results — blueprints have no `get_*` tool, and linking needs only ID + type). `@slug` base-prompt references are composition, not production — note them as future `built-from` candidates, but don't create edges for them. No such tool on this server → skip silently.

## Step 6 — Report

Confirm what was created or updated: name, slug, status, whether MCP-exposed, any typed edges recorded (created / already existed / skipped — tool unavailable), and how teammates use it (`@slug` inside other prompts; the slash command name if exposed — teammates may need to reconnect/refresh their MCP session to see a newly exposed prompt).

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams` requires `team_id` (UUID or slug).
- Slugs are identity: pick them like API names — stable, descriptive, kebab-case. Renaming a slug later breaks `@` references to it.
- Improve existing prompts over creating variants; the library compounds by getting sharper, not longer.
- Typed edges (`vibexp_io_link_resources`): `governed-by` → object must be a blueprint; `built-from` → object must be a prompt; `explained-by` → object must be a memory; `supersedes` → both ends the same type. No self- or cross-project links; re-linking an existing edge is a safe no-op. Tool not available (older server) → skip linking silently, never fail the skill over it.
