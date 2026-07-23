# VibeXP plugin for Claude Code

Best-practice skills for the [VibeXP](https://vibexp.io) knowledge loop: read your team's knowledge before working, write back what you learned after — so every session compounds.

Requires a VibeXP MCP connection (see the [marketplace README](../../README.md#prerequisites)).

## Skills

### `/vibexp:prime [task description]`

Run at the start of a substantial task. It:

1. Detects your VibeXP project by matching the repo's git remote against your projects.
2. Semantically searches memories, blueprints, artifacts, and prompts for what's relevant to the task, and checks recent feed activity for work in flight.
3. Fetches the full content of the top hits and presents a short context brief — rules to follow, relevant history, related work — then starts the task with that knowledge applied.

Read-only: prime never writes to VibeXP.

### `/vibexp:wrap [optional focus]`

Run when finishing a task or session. It:

1. Distills the session into durable learnings, reusable outputs, and a status summary.
2. Shows you a compact write-back plan and asks for one confirmation.
3. Saves learnings as memories — **deduplicating first**: existing memories are extended or corrected rather than duplicated.
4. Saves polished outputs as artifacts (updates existing slugs so the server keeps version history).
5. Records typed relations between what it wrote and existing resources — `governed-by` a blueprint, `supersedes` a replaced artifact, `explained-by` a memory, `built-from` a prompt — when the server supports resource linking (skipped silently otherwise).
6. Posts a status update to your team feed with links to everything created.

### `/vibexp:consolidate [optional scope]`

Run periodically as knowledge-base maintenance (one project at a time). It:

1. Inventories the project's active memories (drafts are humans' work-in-progress and are never touched).
2. Diagnoses three problem classes: near-duplicates, stale/contradicted entries (verified against the codebase where possible), and recurring feed lessons that never became memory.
3. Shows you a gardening plan — merge / correct / archive / promote — and asks for one confirmation.
4. Executes with full provenance: merged-away and stale memories are **archived, never deleted**, and merged canonical memories record which entries they consolidated — as `supersedes` edges when the server supports resource linking.

### `/vibexp:report [start <what> | checkpoint | check]`

Run long or autonomous work as a steerable feed thread. `start` opens one feed item for the work stream with the plan and a steering invitation; `checkpoint` posts milestone replies in-thread (linking artifacts rather than pasting them); `check` reads the thread before each major phase and treats human replies as course corrections — applying them, acknowledging in-thread, and closing with a final outcome reply.

### `/vibexp:promptify [prompt text or which instruction to capture]`

Turn a prompt that worked into a team asset. Checks the library for an existing prompt to improve first, strips session-specifics, parameterizes with `{{variables}}`, factors shared boilerplate into `@slug`-referenced base prompts, then walks draft → published → MCP-exposed — at which point it becomes a native slash command (with variables as arguments) in every teammate's connected tool. Published prompts are linked `governed-by` applicable blueprints when the server supports resource linking.

### `/vibexp:onboard [optional project]`

Bootstrap a repository new to VibeXP. Inventories the repo's AI config (`CLAUDE.md`, `.cursorrules`, `AGENTS.md`, `.cursor/`, copilot instructions, …), dedups against anything already imported, then — behind one confirmation — imports the config files verbatim as per-tool blueprints and seeds 2–4 high-value memories (architecture overview, dev workflow, unwritten conventions), linking each seed memory `governed-by` the blueprint that states its rule when the server supports resource linking. Flags and excludes secrets. Ends by pointing at the loop: prime → wrap → consolidate.
