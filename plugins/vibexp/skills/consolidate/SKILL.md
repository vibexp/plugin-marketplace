---
name: consolidate
description: Tend a project's VibeXP knowledge base so knowledge compounds instead of piling up. Compacts active memories into a small set of canonical entries, compacts piles of repetitive same-series artifacts (whatever the team's recurring per-task output is) into digest artifacts, corrects or archives stale/contradicted entries, and promotes recurring lessons from feed posts. Use periodically as knowledge-base maintenance, or when the user says "consolidate", "clean up memories", "tidy vibexp", "garden the knowledge base", or complains about duplicate/outdated memories or artifacts.
argument-hint: "[optional scope, e.g. a project name or 'auth memories only']"
---

# Consolidate: garden the knowledge base

Memories and artifacts accumulate one session at a time; nobody re-reads the pile. Left alone, a project grows 30+ memories and 150+ artifacts — more than any agent can prime from, so the knowledge stops being used at all. Every active item is a standing tax on every future session: it competes for the context window, for search-result slots, and for the agent's attention. This skill is the maintenance pass that keeps the team's shared brain at a **reasonable size** — small, sharp, current, and token-efficient.

**The target shape, per project:**
- **Memories: ~5 canonical entries** (3–7 is healthy), each a dense, self-contained "brain section" — not 30 overlapping fragments. Organize by durable subsystem (e.g. backend conventions, frontend conventions, team workflow, cross-cutting lessons), not by task or date. The band scales with the project's surface — a large monorepo can justify more canonicals — but the invariant never changes: **the full canonical set must fit comfortably in one prime**.
- **Artifacts: only durable direction documents** — PRDs, benchmark reports, audits, current design docs, and *digest* artifacts that compound a recurring artifact series. Repetitive same-series artifacts (per-task logs, session reports, changelogs — whatever the team's recurring output is) should not stay active indefinitely: they get digested, then archived.

**The keep test — apply it to every single item:** would a future AI agentic session actually reach for this? Keep only what is durable, reusable, and current: conventions, decisions with their *why*, hard-won lessons, living specs and designs. Archive everything else — one-off task state, completed-work logs, superseded plans, trivia. When in doubt, archive: archived items stay recoverable, while a bloated brain helps no one.

Consolidation is not deletion and not just dedup — it is **compaction**: rewrite the knowledge into fewer, denser items, archive the absorbed sources, and verify what remains against the project's current state.

Scope requested by the user (may be empty — consolidate the current project): $ARGUMENTS

Git remote for project detection: !`git remote get-url origin 2>/dev/null || echo "(no git remote)"`

## Step 0 — Pick the VibeXP transport (CLI-first)

Follow **`${CLAUDE_PLUGIN_ROOT}/references/transport.md`**: probe with `command -v vibexp && vibexp whoami` — installed and authenticated → use the official CLI (a big win for this skill: `--format json --jq` trims the inventory pages to just id/slug/title/status at the source); otherwise use the `vibexp_io_*` MCP tools (match on the `vibexp_io_` fragment / core name, never a specific alias). Neither available → STOP and help the user connect per that reference. Steps below name operations by MCP core name; on the CLI transport use the mapped command.

## Step 1 — Resolve scope

1. Resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams` → `list_projects` matched on **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's.
2. A project the user named in the arguments overrides the match. Consolidation runs **one project at a time**; if the user wants the whole team, do it project by project and say so.

## Step 2 — Health check, then inventory at the right depth

**Read the state before choosing the effort.** One page-1 `list_resources` call each for active memories and artifacts gives `total_count` and the most recent items. Pick the pass depth from it:

- **Near the target shape** (memories within ~2× the canonical band, no visible series pile-up) → **triage pass**: verify the canonicals are still current, fold newly accumulated series items into the latest digest, and stop — report that the base is healthy. Don't run the full machinery on a tidy base.
- **Past it** → **deep pass**: the full inventory and treatment below.

For the deep pass, page through `list_resources` for the project with `status: "active"`, 10 per page, for **both** `resource_type: "memory"` and `resource_type: "artifact"`. (On older servers without `list_resources`, fall back to `search_memories` for the memory pass.) Iterate until `total_pages` is exhausted — the totals tell you the size of the problem (e.g. 30 memories / 178 artifacts means real compaction, not a tidy-up).

**Enumerate every page BEFORE acting.** `total_count` and page boundaries shift as you archive, so paginating and archiving at the same time silently skips or double-processes items. Build the full id/slug list first, then work from that fixed list.

**Time-window scope.** If the user's scope names a window (e.g. "last 30 days"), still take the full counts for the report, but focus the deep pass on items created or updated inside the window — recent accumulation is where duplication and rot live. State clearly what the window did and did not cover.

**Active only.** Draft memories and artifacts are work-in-progress owned by their author; never list, modify, or report on them. Archived items are already gardened away.

For very large sets (over ~60 memories), work the most recently updated pages first, cap the pass at a manageable batch, and tell the user what was left for a follow-up run — never silently cover only part while implying you covered it all.

## Step 3 — Diagnose

Group the inventory by topic and look for these problem classes:

1. **Near-duplicates / overlaps** — multiple memories about the same fact or convention. Confirm suspected clusters by running `search` (project-scoped; `types: ["memories"]` where supported) with the cluster's key terms and checking which entries rank together.
2. **Stale, obsolete, or contradicted** — memories/artifacts that disagree with each other or with the project's current state, or that simply no longer matter. Work this class **oldest-untouched first**: sort by `updated_at` — an entry nothing has touched in months is where rot concentrates, while recently updated entries have effectively been re-verified by the sessions that touched them. **Cross-check against reality, not just against other entries:** when running inside the repo, grep the code for named files/flags/commands; check the issue tracker for named epics. Classic stale shapes:
   - Task-state memories ("IN PROGRESS", "next step: X") — task state rots within days; durable knowledge is what survives.
   - **Obsolete** — factually fine but no longer relevant: a decision later reversed, a tool or integration since removed, a plan fully executed with nothing reusable left in it.
   - Documents for an **abandoned direction** (a feasibility study or design for something the team decided against).
   - Documents **superseded by delivery** (a design brief for a feature that has since shipped; the PRD/spec may still earn its place, the brief usually doesn't).
   - Conventions the codebase has since changed (a route structure, a pin, a tool version).
3. **Artifact pile-up** — repetitive same-series artifacts (a team's recurring per-task output: per-issue/per-PR logs, session reports, experiment notes…) accumulate fastest. Whatever the series is called in this project, the treatment is the same: the generalizable lessons belong in a digest, the individual items get archived. One-off durable documents are not pile-up. Check existing digests too: **a digest's metadata can lie** — one may claim "the individual items are archived" when they are still active (or vice versa). Trust the listing, not the claim.
4. **Unharvested feed lessons** — skim recent feed items (`list_feeds` → `list_feed_items`, first 1–2 pages; `get_feed_item` only when a title suggests a durable lesson). A lesson that keeps recurring in status posts but never made it into memory is a promotion candidate.

Before deciding anything, fetch full content (`get_resource`; on older servers, `get_memory`) for every item involved in a merge, correction, or archival. Never act on an excerpt.

## Step 4 — Propose the gardening plan, get one confirmation

Present a compact plan grouped by action, then ask for a single go/adjust confirmation:

- **Compact memories**: the target set of canonical entries (name each, one-line role), which existing memories each absorbs, which entries get archived, and the `supersedes` edges to record (canonical → each merged-away entry; only when a `link_resources` tool is available).
- **Compact artifacts**: the digest artifact(s) to create (which series and what range each covers), which individual items get archived after, and which non-series artifacts stay (PRDs, benchmarks, audits, current designs) vs. go (abandoned-direction docs, delivered briefs).
- **Correct**: which item, what changes and why (cite the evidence — the newer memory, feed post, code, or tracker state that contradicts it).
- **Promote**: new memories distilled from recurring feed lessons.

This skill rewrites shared team knowledge — the user vets the plan. If they narrow it, respect the narrowed scope exactly.

## Step 5 — Execute

**Order matters: create every replacement BEFORE archiving its sources.** A digest or canonical that exists only in your plan is not a backup.

- **Canonical memories**: pick the best-established memory as the canonical one and `update_memory` it with a **full rewrite** — combined, deduplicated, self-contained text written to be read cold by an agent, not an append of the old fragments. Write for **token efficiency**: short declarative lines, no narrative, no how-we-got-here history — every sentence must earn its tokens for a future session. Add provenance metadata such as `{"consolidated_from": ["<id>", ...], "role": "<canonical-role>"}`. Then archive the merged-away entries (`status: "archived"`). If a `link_resources` tool is available, record a `supersedes` edge from the canonical to each archived entry — the typed edge is the canonical consolidation trail, the metadata a fallback. No such tool → skip silently, never fail the merge over it.
- **Digest artifacts** (the compaction tool for a repetitive series): `create_artifact` named after the series, e.g. `<series>-digest-YYYY-MM-part-N`. Style: **generalizable lessons only, each stated once**, grouped into a short "cross-cutting lessons" list (with `(#issue/PR)` pointers for provenance) plus one section per epic/subsystem — no per-item prose. Note in the text where the repo-current durable content lives (e.g. the canonical memories) so the digest serves as the provenance layer. Metadata: `{"category": "digest", "compounded_from_count": N, "date": "...", "part": N}`. Then archive every item it covers.
- **Corrections**: `update_memory` / `update_artifact` with the fixed text — resolve the contradiction in the text itself, don't append both versions.
- **Archivals**: `status: "archived"` via the update tool. **Never use `delete_resource`** — archived items stay recoverable and out of search; deletion is not gardening. Archivals are independent — batch them in parallel groups.
- **Promotions**: feed lessons → `create_memory` (`status: "active"`) with metadata like `{"category": "lesson", "source": "feed"}`.

**Verify the end state**: after executing, re-list active items and count them. Report the real before/after numbers, and spot-check that a kept item wasn't accidentally archived.

## Step 6 — Report

Summarize what changed: memories N → M active, artifacts N → M active; what each canonical absorbed; what each digest covers; N corrected, N archived, N promoted from feeds, N `supersedes` edges recorded (created / already existed / skipped — tool unavailable) — each with a one-line reason. If nothing needed consolidating, say so plainly; a clean report is a good outcome, not a failure. Remind the user the next run's job: fold newly accumulated series items into the latest digest (or start the next part) and refresh the canonicals — consolidation is a recurring habit, not a one-time fix.

## Conventions (apply throughout)

- Every tool except `get_user` and `list_teams` requires `team_id` (UUID or slug).
- Identifiers differ per resource type: memories are read/updated by `id`; artifacts and blueprints by `project_id` + `slug`.
- Excerpts are for triage only; fetch full content before any edit.
- Archive over delete, always. Every destructive-looking action must appear in the confirmed plan first.
- The goal is fewer, denser items. If a merge would produce a bloated catch-all, keep the entries separate and sharpen each instead. Shorter beats longer: a canonical that fits an agent's context gets read; a thorough one that doesn't, doesn't.
- Typed edges (`link_resources`): `governed-by` → object must be a blueprint; `built-from` → object must be a prompt; `explained-by` → object must be a memory; `supersedes` → both ends the same type. No self- or cross-project links; re-linking an existing edge is a safe no-op. Tool not available (older server) → skip linking silently, never fail the skill over it.
