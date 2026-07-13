# CLAUDE.md

Guidance for AI agents working in this repo.

## What this is

The official **Claude Code plugin marketplace for VibeXP** (https://vibexp.io — the open-source, self-hostable shared knowledge base AI tools read from and write back to over MCP). This repo contains no application code: it is manifests plus skills as Markdown.

```
.claude-plugin/marketplace.json   Marketplace manifest ("vibexp") — lists the plugins below
plugins/vibexp/                   The core plugin (currently the only one)
  .claude-plugin/plugin.json      Plugin manifest (name, version, metadata)
  skills/<skill>/SKILL.md         One directory per skill; invoked as /vibexp:<skill>
  README.md                       User-facing skill documentation
```

Skills (the VibeXP knowledge loop): `prime` (read team knowledge before a task), `wrap` (write learnings/artifacts/feed update back after), `consolidate` (merge/correct/archive memories over time), `report` (feed-steered work streams), `promptify` (publish reusable prompts), `onboard` (bootstrap a project's knowledge base).

## Design rules for skills — every skill must follow these

- **Deployment-agnostic, public audience.** Never assume the hosted instance, a specific MCP server alias, or any particular team/project. Match VibeXP tools on the `vibexp_io_` name fragment; `https://connect.vibexp.io/mcp/v1/common` appears only as an *example* alongside `<your-vibexp-host>`.
- **Connection preflight.** If no `vibexp_io_*` tools are available, the skill STOPS and walks the user through `claude mcp add --transport http vibexp https://<host>/mcp/v1/common` (OAuth in browser, no API key).
- **Discovery flow.** `vibexp_io_list_teams` first (everything except `get_user`/`list_teams` needs `team_id`), then resolve the project by matching the git remote against project `git_url` (tolerate SSH/HTTPS forms, ignore `.git`), asking the user only when ambiguous.
- **Excerpt-vs-get.** Search/list tools return ~300-char excerpts; skills must fetch full content via `get_*` before relying on or editing anything. Blueprints and prompts have no MCP get tool — work from excerpts and say so.
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
