# VibeXP plugin for Claude Code

Best-practice skills for the [VibeXP](https://vibexp.io) knowledge loop: read your team's knowledge before working, write back what you learned after — so every session compounds.

Requires a VibeXP connection — either the [official `vibexp` CLI](https://docs.vibexp.io) (installed + `vibexp auth login`) or an MCP connection (see the [marketplace README](../../README.md#prerequisites)). Every skill probes for the CLI first and uses it when authenticated — responses can be trimmed at the source (`--format json --jq`), so far fewer tokens enter the context, and the CLI can read full blueprint/prompt content, which MCP cannot — falling back to the MCP tools otherwise. Shared by every skill here: [`references/transport.md`](references/transport.md).

## Skills

### `/vibexp:prime [task description]`

Run at the start of a substantial task. It:

1. Detects your VibeXP **team and project** by matching the repo's git remote against each project's `git_url` — across all your teams, since the owning team isn't guessable from the org name. The mapping is cached in project memory, so later sessions resolve in zero API calls. Shared by every skill here: [`references/resolve-scope.md`](references/resolve-scope.md).
2. Sizes retrieval to the task — a focused fix gets one targeted search and no feed scan; substantial work gets complementary queries plus a feed check when it could collide with work in flight.
3. Fetches full content only for what the task needs *now* (a hard cap of 3–5 fetches) and presents a short context brief — rules to follow, relevant history, related work — plus a **knowledge map** of the relevant-but-unfetched hits, pulled **just-in-time** later when the work actually enters those areas.
4. Records what it pulled in a local **session ledger** ([`references/session-ledger.md`](references/session-ledger.md)) so `/vibexp:wrap` can audit that knowledge for staleness at session end.

Read-only: prime never writes to VibeXP.

### `/vibexp:wrap [optional focus]`

Run when finishing a task or session. It:

1. **Audits what prime pulled**: reads the session ledger and checks each relied-on resource against what the session actually found — contradicted or outdated entries get corrected at the source, so every session leaves what it read more correct than it found it.
2. Distills the session into durable learnings, reusable outputs, and a status summary, and checks local knowledge (CLAUDE.md and friends, agent memory) for team-relevant changes to sync back.
3. Shows you a compact write-back plan and asks for one confirmation.
4. Saves learnings **merge-first**: updating an existing memory is the expected outcome, creating one is the exception (wanting 3+ new memories in one wrap is treated as a signal to fold into existing canonicals instead).
5. Saves polished outputs as artifacts — preferring to extend a living document over minting per-session artifacts — and records typed relations (`governed-by`, `supersedes`, `explained-by`, `built-from`) when the server supports linking.
6. Posts a status update to your team feed, and ends with a health check: when active memories or artifacts have visibly outgrown the target shape, it recommends running `/vibexp:consolidate`.

### `/vibexp:consolidate [optional scope]`

Run periodically as knowledge-base maintenance (one project at a time). It:

1. **Health-checks first**: page-1 totals decide between a cheap triage pass (base near target shape — verify canonicals, fold new series items into the latest digest, stop) and the full deep pass.
2. Inventories the project's active memories and artifacts (drafts are humans' work-in-progress and are never touched).
3. Diagnoses near-duplicates, stale/contradicted entries (oldest-untouched first, verified against the codebase where possible), artifact pile-up, and recurring feed lessons that never became memory.
4. Shows you a gardening plan — compact / correct / archive / promote — and asks for one confirmation.
5. Executes with full provenance: merged-away and stale items are **archived, never deleted**, and canonical memories record which entries they consolidated — as `supersedes` edges when the server supports resource linking.

### `/vibexp:report [start <what> | checkpoint | check]`

Run long or autonomous work as a steerable feed thread. `start` opens one feed item for the work stream with the plan and a steering invitation; `checkpoint` posts milestone replies in-thread (linking artifacts rather than pasting them); `check` reads the thread before each major phase, treats human replies as course corrections — applying them, acknowledging in-thread, and closing with a final outcome reply — and refreshes context just-in-time from prime's knowledge map for the upcoming phase.

### `/vibexp:promptify [prompt text or which instruction to capture]`

Turn a prompt that worked into a team asset. Checks the library for an existing prompt to improve first, strips session-specifics, parameterizes with `{{variables}}`, factors shared boilerplate into `@slug`-referenced base prompts, then walks draft → published → MCP-exposed — at which point it becomes a native slash command (with variables as arguments) in every teammate's connected tool. Published prompts are linked `governed-by` applicable blueprints when the server supports resource linking.

### `/vibexp:onboard [optional project]`

Bootstrap a repository new to VibeXP. Inventories the repo's AI config (`CLAUDE.md`, `.cursorrules`, `AGENTS.md`, `.cursor/`, copilot instructions, …), dedups against anything already imported, then — behind one confirmation — imports the config files verbatim as per-tool blueprints and seeds 2–4 high-value memories (architecture overview, dev workflow, unwritten conventions), linking each seed memory `governed-by` the blueprint that states its rule when the server supports resource linking. Flags and excludes secrets. Ends by pointing at the loop: prime → wrap → consolidate.
