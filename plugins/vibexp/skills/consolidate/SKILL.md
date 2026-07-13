---
name: consolidate
description: Tend a project's VibeXP memory so knowledge compounds instead of piling up. Reviews the project's active memories to merge near-duplicates, correct or archive stale and contradicted entries, and promote recurring lessons from feed posts into durable memory. Use periodically as knowledge-base maintenance, or when the user says "consolidate", "clean up memories", "tidy vibexp", "garden the knowledge base", or complains about duplicate/outdated memories.
argument-hint: "[optional scope, e.g. a project name or 'auth memories only']"
---

# Consolidate: garden the knowledge base

Memories accumulate one session at a time; nobody re-reads the pile. This skill is the maintenance pass that keeps the knowledge base worth priming from: fewer, sharper, current memories.

Scope requested by the user (may be empty — consolidate the current project): $ARGUMENTS

Git remote for project detection: !`git remote get-url origin 2>/dev/null || echo "(no git remote)"`

## Step 0 — Check the VibeXP MCP connection

VibeXP tools are named `vibexp_io_*` (prefixed with the user's MCP server alias — match on `vibexp_io_`, never assume an alias). If none are available, STOP and help the user connect:

```
claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common
```

(Hosted instance: `https://connect.vibexp.io/mcp/v1/common`; self-hosters use their own origin. OAuth in the browser, no API key. Docs: https://docs.vibexp.io)

## Step 1 — Resolve scope

1. Team: `vibexp_io_list_teams` (one → use it; several → match this repo or ask once).
2. Project: `vibexp_io_list_projects`, match the git remote against `git_url` (ignore `.git`; SSH and HTTPS forms are equal) or the directory name — or use the project the user named in the arguments. Consolidation runs **one project at a time**; if the user wants the whole team, do it project by project and say so.

## Step 2 — Inventory the memories

Page through `vibexp_io_search_memories` for the project with `status: "active"` (10 per page; iterate until `total_pages` is exhausted). Results are ~300-char excerpts — enough to cluster, not enough to edit from.

**Active memories only.** Draft memories are out of scope — they are work-in-progress owned by their author; never list, modify, or report on them. Archived memories are also out of scope (they were already gardened away).

For large sets (over ~60 memories), work the most recently updated pages first, cap the pass at a manageable batch, and tell the user what was left for a follow-up run — never silently cover only part while implying you covered it all.

## Step 3 — Diagnose

Group the inventory by topic and look for three problem classes:

1. **Near-duplicates / overlaps** — multiple memories about the same fact or convention. Confirm suspected clusters by running `vibexp_io_search` (`types: ["memories"]`, project-scoped) with the cluster's key terms and checking which entries rank together.
2. **Stale or contradicted** — memories that disagree with each other, or (when running inside the repo) with the current code. Verify claims against the codebase where cheap: a memory that names a file, flag, or command can be checked with a quick grep before being trusted or corrected.
3. **Unharvested feed lessons** — skim recent feed items (`vibexp_io_list_feeds` → `vibexp_io_list_feed_items`, first 1–2 pages; `vibexp_io_get_feed_item` only when a title suggests a durable lesson). A lesson that keeps recurring in status posts but never made it into memory is a promotion candidate.

Before deciding anything, fetch full text with `vibexp_io_get_memory` for every memory involved in a merge, correction, or archival. Never act on an excerpt.

## Step 4 — Propose the gardening plan, get one confirmation

Present a compact plan grouped by action, then ask for a single go/adjust confirmation:

- **Merge**: which memories combine, one-line summary of the canonical text, which entries get archived.
- **Correct**: which memory, what changes and why (cite the evidence — the newer memory, feed post, or code that contradicts it).
- **Archive**: which memory, why it no longer earns its place.
- **Promote**: new memories distilled from recurring feed lessons.

This skill rewrites shared team knowledge — the user vets the plan. If they narrow it, respect the narrowed scope exactly.

## Step 5 — Execute

- **Merges**: pick the best-established memory as the canonical one and `vibexp_io_update_memory` it with the combined, self-contained text; add provenance metadata such as `{"consolidated_from": ["<id>", ...]}`. Then archive the merged-away entries (`status: "archived"`).
- **Corrections**: `vibexp_io_update_memory` with the fixed text — resolve the contradiction in the text itself, don't append both versions.
- **Archivals**: `vibexp_io_update_memory` with `status: "archived"`. **Never use `vibexp_io_delete_resource`** — archived memories stay recoverable and out of search; deletion is not gardening.
- **Promotions**: feed lessons → `vibexp_io_create_memory` (`status: "active"`) with metadata like `{"category": "lesson", "source": "feed"}`.

## Step 6 — Report

Summarize what changed: N merged into M, N corrected, N archived, N created from feed lessons — each with a one-line reason, and the before/after active-memory count. If nothing needed consolidating, say so plainly; a clean report is a good outcome, not a failure.

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams` requires `team_id` (UUID or slug).
- Excerpts are for triage only; `vibexp_io_get_memory` before any edit.
- Archive over delete, always. Every destructive-looking action must appear in the confirmed plan first.
- The goal is fewer, better memories — if a merge would produce a bloated catch-all, keep the memories separate and sharpen each instead.
