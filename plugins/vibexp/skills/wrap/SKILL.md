---
name: wrap
description: Wrap up a work session by writing back to VibeXP so the team's knowledge compounds. Distills the session into durable learnings saved as memories (deduplicated against existing ones), stores polished reusable outputs as artifacts, and posts a status update to the team feed. Use when finishing a task or session, or when the user says "wrap", "wrap up", "save learnings", "log this to vibexp", or "update the team".
argument-hint: "[optional focus, e.g. 'just the feed update']"
---

# Wrap: make this session count for the next one

Write the session's durable value back to VibeXP: learnings → memories, polished outputs → artifacts, status → feed. Be selective — a knowledge base compounds only if what goes in is worth reading later.

Focus requested by the user (may be empty — do the full wrap): $ARGUMENTS

## Step 0 — Check the VibeXP MCP connection

VibeXP tools are named `vibexp_io_*` (prefixed with whatever MCP server alias the user chose — match on `vibexp_io_`, never assume an alias). If none are available, STOP and help the user connect:

```
claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common
```

(Hosted instance: `https://connect.vibexp.io/mcp/v1/common`; self-hosters use their own origin. OAuth in the browser on first use, no API key. Docs: https://docs.vibexp.io)

## Step 1 — Resolve team and project

If `/vibexp:prime` already resolved these this session, reuse them. Otherwise: `vibexp_io_list_teams` (one team → use it; several → match this repository or ask once), then `vibexp_io_list_projects` and match the git remote URL against `git_url` (ignore `.git`, treat SSH and HTTPS forms as equal) or the directory name against name/slug. The project is required for memories and artifacts; if none matches, ask before writing anywhere.

## Step 2 — Harvest the session

Review the whole session and collect candidates in three buckets:

1. **Learnings** — durable, non-obvious knowledge that would save the next session (yours or a teammate's) real time: decisions made and *why*, gotchas hit, constraints discovered, approaches that failed and the reason. NOT: session-specific trivia, anything already obvious from the repo's code or docs, or restated task descriptions.
2. **Reusable outputs** — finished, polished content someone would deliberately open again: reports, designs, runbooks, migration notes, analyses. NOT: status chatter or half-done drafts (those belong in the feed or nowhere).
3. **Status** — what was accomplished, decisions taken, open questions; the update a teammate would want to skim.

## Step 3 — Propose, then get one confirmation

Show the user a compact write-back plan — each memory candidate in one line, each artifact with its intended slug/type, and the feed post title — and ask for a single go/adjust confirmation before writing anything. This is a shared team knowledge base; the user vets what enters it. Skip the confirmation only if the user already told you to proceed without review.

## Step 4 — Learnings → memories (dedup first, always)

For each learning, **search before creating**: `vibexp_io_search` with `types: ["memories"]` (and/or `vibexp_io_search_memories`) using the learning's key terms, scoped to the project.

- An existing memory already covers it → skip.
- An existing memory partially covers it or is now outdated → `vibexp_io_get_memory` for the full text, then `vibexp_io_update_memory` to extend or correct it. Never append contradictions — resolve them.
- Genuinely new → `vibexp_io_create_memory` with the project's `project_id`, concise self-contained `text` (a reader has no session context), helpful `metadata` (e.g. `{"category": "gotcha"}`), and `status: "active"`. If a learning is not confirmed enough to save as active, don't save it at all — draft status is for humans' work-in-progress, not agent hypotheses. Prefer updating/archiving over deleting — history matters.

This dedup discipline is the difference between knowledge that compounds and a pile of near-duplicates.

## Step 5 — Outputs → artifacts

For each reusable output:

- Check for an existing artifact first: `vibexp_io_search_artifacts` in the project (search by intended slug/title). Exists → `vibexp_io_update_artifact` (the server snapshots the previous version automatically). New → `vibexp_io_create_artifact` with a stable kebab-case `slug`, clear `title`, `description`, and a `type` that matches the team's artifact types (defaults: `general`, `work-reports`, `static-contexts`; the team may define custom ones — when unsure use `general`).
- Content is Markdown; keep individual artifacts under ~1 MB. Files (images, PDFs, etc.) can be attached with `vibexp_io_upload_attachment` (`owner_type: "artifact"`, max 5 MB/file, 10 MB/artifact) — note uploads fail with a clear error if the instance has no object storage configured; report that rather than retrying.

## Step 6 — Status → feed

1. `vibexp_io_list_feeds`, pick the topically right feed (or the general one).
2. `vibexp_io_post_to_feed` with:
   - `title`: outcome-first, e.g. "Refactored auth module — 12 files, sessions now stateless".
   - `content`: Markdown — what was done, key decisions, links (`full_url`) to the artifacts/memories just created, open questions.
   - `ai_assistant_name`: a **stable** identifier such as `Claude Code` — the same value every time, never random or timestamped.
   - `project_id`: the resolved project.

Feeds are for status and summaries ("anything you'd otherwise put in chat"); polished reusable content belongs in artifacts — don't blur the two.

## Step 7 — Report

Tell the user exactly what was written: each memory (created vs updated), each artifact, and the feed post — with their `full_url` links. If a quota/resource-limit error came back (feed posting is quota-gated), report it plainly instead of retrying.

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams` requires `team_id` (UUID or slug).
- Search/list tools return ~300-char excerpts; call `get_*` before updating anything so you edit the full text, not an excerpt.
- Quality bar over quantity: two excellent memories beat ten noisy ones. When in doubt, leave it out.
