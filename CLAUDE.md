# CLAUDE.md

Guidance for AI agents working in this repo.

## What this is

The official **Claude Code plugin marketplace for VibeXP** (https://vibexp.io — the open-source, self-hostable shared knowledge base AI tools read from and write back to over MCP). This repo contains no application code: it is manifests plus skills as Markdown.

```
.claude-plugin/marketplace.json   Marketplace manifest ("vibexp") — lists the plugins below
plugins/vibexp/                   The core plugin (currently the only one)
  .claude-plugin/plugin.json      Plugin manifest (name, version, metadata)
  skills/<skill>/SKILL.md         One directory per skill; invoked as /vibexp:<skill>
  references/                     Shared procedures every skill references:
                                  transport.md (CLI-first transport selection),
                                  resolve-scope.md (team/project from git URL),
                                  session-ledger.md (what prime pulled, for wrap's audit)
  README.md                       User-facing skill documentation
```

Skills (the VibeXP knowledge loop): `prime` (read team knowledge before a task), `wrap` (write learnings/artifacts/feed update back after), `consolidate` (merge/correct/archive memories over time), `report` (feed-steered work streams), `promptify` (publish reusable prompts), `onboard` (bootstrap a project's knowledge base).

## Design rules for skills — every skill must follow these

- **Deployment-agnostic, public audience.** Never assume the hosted instance, a specific MCP server alias, or any particular team/project. Match VibeXP tools on the `vibexp_io_` name fragment; `https://connect.vibexp.io/mcp/v1/common` appears only as an *example* alongside `<your-vibexp-host>`.
- **Transport preflight (CLI-first).** Step 0 of every skill follows `references/transport.md`: probe `command -v vibexp && vibexp whoami` (exit 0 = installed + authenticated; `auth status` exits 0 even when logged out — don't use it) → use the official CLI; else the `vibexp_io_*` MCP tools; neither → STOP and walk the user through `vibexp auth login` or `claude mcp add --transport http vibexp https://<host>/mcp/v1/common` (OAuth in browser, no API key). Skill bodies name operations by MCP core name; the CLI mapping lives in transport.md. CLI `--team` takes the team UUID, not a slug.
- **Token discipline.** Prime pulls only what the task needs now (capped fetches) and leaves a knowledge map for just-in-time retrieval; wrap merges into existing resources before creating; on the CLI, responses are trimmed with `--format json --jq`.
- **Session ledger.** Prime records what it retrieved in a local `vibexp-last-prime.md` (`references/session-ledger.md`); wrap audits those resources for staleness and corrects them at the source; report's `check` mode uses the ledger's knowledge map at phase boundaries.
- **Discovery flow.** `vibexp_io_list_teams_and_projects` first (everything except `get_user`/`list_teams_and_projects` needs `team_id`) — one cross-team call, queried with the repo's normalized `git_url`; only an exact hit (`score: 1.0`) resolves scope, anything lower is a candidate needing confirmation. `list_teams`/`list_projects` are deprecated aliases (server v0.11.0) and must not be used.
- **Excerpt-vs-get.** Search/list tools return ~300-char excerpts; skills must fetch full content via `get_*` before relying on or editing anything. Blueprints and prompts have no MCP get tool — on the MCP transport, work from excerpts and say so; on the CLI, `vibexp blueprint get` / `vibexp prompt get` return full content.
- **Write discipline.** One consolidated user confirmation before writing to shared team knowledge. Dedup-search before creating memories/prompts/blueprints; update existing resources rather than duplicating (the server versions content on update). Archive over delete — `vibexp_io_delete_resource` is never used by these skills.
- **Draft status is human territory.** Skills never create or modify `draft` resources; agent knowledge is either good enough to be `active` or not saved.
- **Feeds vs artifacts.** Status/progress goes to feeds with a stable `ai_assistant_name` (never random/timestamped); polished reusable outputs are artifacts, linked from feed posts.

## Conventions & workflow

- SKILL.md frontmatter keeps `name`, `description` (triggers included; combined limit 1,536 chars), `argument-hint`. Body style: numbered steps, a Step 0 connection preflight, and a closing "Conventions (apply throughout)" section.
- Keep the three docs in sync when skills change: the skill's SKILL.md, `plugins/vibexp/README.md`, and the root `README.md` table. Bump `plugins/vibexp/.claude-plugin/plugin.json` `version` on any user-visible change (users only receive updates when it changes).
- Validate before committing: `claude plugin validate .` and `claude plugin validate plugins/vibexp`.
- Test locally: `claude plugin marketplace add ./` then `/plugin install vibexp@vibexp`.
- Ground any new tool usage in the real MCP surface: tool definitions live in the main repo at `vibexp/backend/internal/server/mcp_*.go` (siblings of this repo under `~/Projects/`), user docs at https://docs.vibexp.io.

## License

MIT. Don't commit secrets; this is a public repo.
