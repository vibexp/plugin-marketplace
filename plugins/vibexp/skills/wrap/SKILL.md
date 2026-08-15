---
name: wrap
description: Wrap up a work session by writing back to VibeXP so the team's knowledge compounds. Audits the knowledge the session primed with and corrects what went stale, distills the session into durable learnings — merged into existing memories first, created only when genuinely new — stores polished reusable outputs as artifacts, syncs local knowledge changes (CLAUDE.md, agent memory), and posts a status update to the team feed. Use when finishing a task or session, or when the user says "wrap", "wrap up", "save learnings", "log this to vibexp", or "update the team".
argument-hint: "[optional focus, e.g. 'just the feed update']"
---

# Wrap: make this session count for the next one

Write the session's durable value back to VibeXP: corrections → existing resources, learnings → memories, polished outputs → artifacts, status → feed. Be selective — a knowledge base compounds only if what goes in is worth reading later, and it compounds fastest when existing knowledge gets *sharper*, not when new entries pile up next to it. **Updating is the expected outcome; creating is the exception.**

Focus requested by the user (may be empty — do the full wrap): $ARGUMENTS

## Step 0 — Pick the VibeXP transport (CLI-first)

Follow **`${CLAUDE_PLUGIN_ROOT}/references/transport.md`**: probe with `command -v vibexp && vibexp whoami` — installed and authenticated → use the official CLI (trim responses with `--format json --jq`; write bodies via `--body-file`); otherwise use the `vibexp_io_*` MCP tools (match on `vibexp_io_`, never assume an alias). Neither available → STOP and help the user connect per that reference. Steps below name operations by MCP core name; on the CLI transport use the mapped command.

## Step 1 — Resolve team and project

Reuse what `/vibexp:prime` resolved this session (the session ledger records it); otherwise resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams` → `list_projects` matched on **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's.

The project is required for memories and artifacts — no match → ask before writing anywhere.

## Step 2 — Audit what prime pulled

Read the session ledger (**`${CLAUDE_PLUGIN_ROOT}/references/session-ledger.md`** — `vibexp-last-prime.md` in the local memory dir). For each resource under *Relied on*, ask: **did this session's work contradict, outdate, or extend it?** A convention that changed, a decision that got reversed, a gotcha that turned out to be fixed, a doc the session made obsolete. If the ledger predates this session, audit only the entries the conversation actually used; no ledger → skip this step (note it in the report) — don't reconstruct it with searches.

Contradicted or outdated entries become the **corrections bucket**: the fix is an update to *that* resource, resolving the contradiction in its text. This is how primed knowledge stays trustworthy — every session leaves what it read more correct than it found it.

## Step 3 — Harvest the session

Review the whole session and collect candidates in four buckets:

1. **Corrections** — from Step 2: existing resources the session proved stale, each with the evidence and the fixed text.
2. **Learnings** — durable, non-obvious knowledge that would save the next session (yours or a teammate's) real time: decisions made and *why*, gotchas hit, constraints discovered, approaches that failed and the reason. NOT: session-specific trivia, anything already obvious from the repo's code or docs, or restated task descriptions.
3. **Reusable outputs** — finished, polished content someone would deliberately open again: reports, designs, runbooks, migration notes, analyses. NOT: status chatter or half-done drafts (those belong in the feed or nowhere).
4. **Status** — what was accomplished, decisions taken, open questions; the update a teammate would want to skim.

## Step 4 — Check local knowledge for changes to sync

The session may have changed knowledge that lives *outside* VibeXP:

- **Repo AI config**: `git status`/`git diff` on `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `.claude/`, `.cursor/` — meaningful changes to a file that was imported as a blueprint (see `/vibexp:onboard`) become a proposed `update_blueprint` so the team copy doesn't drift from the repo.
- **Local agent memory**: files in the local memory dir newer than the ledger's date. Team-relevant facts (project state, conventions, decisions) are sync candidates as memory updates/creations; **personal entries (`type: user`) and preferences never sync** — they're the user's, not the team's.

Anything found joins the plan in Step 5.

## Step 5 — Propose, then get one confirmation

Show the user a compact write-back plan — each correction (resource + what changes and why), each memory candidate marked **update** or **create** (one line each), each artifact with its intended slug/type, each local-knowledge sync, any typed edges to record (one line each, e.g. `artifact deploy-runbook —governed-by→ blueprint claude-md`; see Step 8), and the feed post title — and ask for a single go/adjust confirmation before writing anything. This is a shared team knowledge base; the user vets what enters it. Skip the confirmation only if the user already told you to proceed without review.

## Step 6 — Corrections and learnings → memories (merge-first, always)

Execute corrections first: `get` the full text, then `update_memory` / `update_artifact` with the contradiction resolved in the text — never append both versions.

For each learning, **search before creating**: `search` with `types: ["memories"]` using the learning's key terms, scoped to the project. Then, in order of preference:

- An existing memory already covers it → skip.
- An existing memory covers the same ground → `get` the full text, then `update_memory` to fold the learning in — extend, sharpen, correct.
- Genuinely new — the searches came back empty of anything adjacent → `create_memory` with the project's `project_id`, concise self-contained `text` (a reader has no session context), helpful `metadata` (e.g. `{"category": "gotcha"}`), and `status: "active"`.

**Creation smell**: wanting to create more than 2–3 new memories in one wrap almost always means the learnings should be folded into the project's existing canonical memories instead — recheck before creating. If a learning is not confirmed enough to save as active, don't save it at all — draft status is for humans' work-in-progress, not agent hypotheses.

**Write for token efficiency** — every memory is read by future sessions on a budget: short declarative lines, no narrative, no how-we-got-here history; every sentence must earn its tokens.

## Step 7 — Outputs → artifacts

For each reusable output, in order of preference:

- A **living document** for this already exists (a runbook, design doc, or series digest the output extends) → `update_artifact` (the server snapshots the previous version automatically; on the CLI, record `--change-summary`).
- Genuinely new → `create_artifact` with a stable kebab-case `slug`, clear `title`, `description`, and a `type` matching the team's artifact types (defaults: `general`, `work-reports`, `static-contexts`; when unsure use `general`). Prefer slugs a future session would update again (`deploy-runbook`, not `deploy-notes-2026-08-16`) — per-session artifacts are the pile `/vibexp:consolidate` has to clean up later.
- Content is Markdown; keep individual artifacts under ~1 MB. Files (images, PDFs, etc.) can be attached with `upload_attachment` (`owner_type: "artifact"`, max 5 MB/file, 10 MB/artifact) — uploads fail with a clear error if the instance has no object storage configured; report that rather than retrying.

## Step 8 — Link what you wrote (typed edges)

If a `link_resources` operation is available (CLI: `relations create --origin ai`), record how each resource created or updated in Steps 6–7 relates to existing resources, so the team's knowledge graph maintains itself as a side effect of the wrap. Propose only edges that are clearly true:

- `governed-by` — the memory/artifact follows a rule a specific blueprint states (the object must be a blueprint).
- `supersedes` — a new artifact replaces a *different*, older artifact (both ends must be the same resource type; updating the same resource needs no edge — the server versions content on update).
- `explained-by` — an artifact whose rationale lives in a memory (the object must be a memory).
- `built-from` — an artifact a team prompt produced (the object must be a prompt).

Call it with the project's `project_id` and each end's type + UUID — create/update responses and search/list results include IDs. These edges were part of the confirmed Step 5 plan; report each one's outcome in Step 10.

## Step 9 — Status → feed

1. `list_feeds`, pick the topically right feed (or the general one).
2. `post_to_feed` with:
   - `title`: outcome-first, e.g. "Refactored auth module — 12 files, sessions now stateless".
   - `content`: Markdown — what was done, key decisions, links (`full_url`) to the artifacts/memories just created or corrected, open questions.
   - `ai_assistant_name`: a **stable** identifier such as `Claude Code` — the same value every time, never random or timestamped (CLI: `--author "Claude Code"`, never the CLI default).
   - `project_id`: the resolved project.

Feeds are for status and summaries ("anything you'd otherwise put in chat"); polished reusable content belongs in artifacts — don't blur the two.

## Step 10 — Report, with a health signal

Tell the user exactly what was written: each correction, each memory (created vs updated), each artifact, each local-knowledge sync, each typed edge (created / already existed / skipped — unavailable), and the feed post — with their `full_url` links. If a quota/resource-limit error came back (feed posting is quota-gated), report it plainly instead of retrying.

Then check the base's health cheaply — one `list_resources` page-1 call each for active memories and artifacts gives `total_count`. Active memories well past the canonical band (say, over ~15) or a visibly growing same-series artifact pile → end with: **recommend running `/vibexp:consolidate`**. This is how the loop stays closed — consolidation triggered by state, not by someone remembering.

## Conventions (apply throughout)

- Every operation except `get_user` and `list_teams` requires `team_id` (UUID or slug; the CLI's `--team` takes the UUID).
- Search/list return ~300-char excerpts; call `get` before updating anything so you edit the full text, not an excerpt.
- Update over create; archive over delete; quality bar over quantity — two excellent memories beat ten noisy ones. When in doubt, leave it out.
- Typed edges: `governed-by` → object must be a blueprint; `built-from` → object must be a prompt; `explained-by` → object must be a memory; `supersedes` → both ends the same type. No self- or cross-project links; re-linking an existing edge is a safe no-op. Operation not available (older server) → skip linking silently, never fail the skill over it.
