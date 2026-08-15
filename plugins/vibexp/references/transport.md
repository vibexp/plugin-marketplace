# Choosing the VibeXP transport: CLI first, MCP fallback

VibeXP is reachable two ways and they hit the same API: the **official CLI**
(`vibexp`) and the **MCP tools** (core names like `search`, `get_resource`,
prefixed per installation, e.g. `mcp__<alias>__vibexp_io_search` — match on the
`vibexp_io_` fragment, never assume an alias). Every skill picks its transport
once, in Step 0, with this preflight.

## Preflight

```bash
command -v vibexp >/dev/null 2>&1 && vibexp whoami >/dev/null 2>&1
```

- **Exit 0** → the CLI is installed *and* authenticated: **use the CLI** for
  this session. (Probe with `whoami`, not `auth status` — `auth status` exits 0
  even when logged out; `whoami` exits non-zero.)
- **Non-zero** → use the `vibexp_io_*` MCP tools if any are available.
- **CLI installed but unauthenticated, and no MCP tools** → STOP and offer both
  fixes; either works:
  - CLI: `vibexp auth login` (browser OAuth; `--with-api-key` for headless).
  - MCP: `claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common`
    (hosted instance: `https://connect.vibexp.io/mcp/v1/common`; OAuth in the
    browser, no API key), then restart or `/mcp`. Docs: https://docs.vibexp.io
- **Neither installed** → same STOP; the MCP route needs no local install.

Why CLI first: responses can be trimmed at the source (`--format json` +
`--jq`), so far fewer tokens enter the context; and the CLI has full reads for
blueprints and prompts (`blueprint get`, `prompt get`), which MCP lacks.

Note the chosen transport in the session ledger (see `session-ledger.md`).

## Command mapping

Skills name operations by MCP core name; on the CLI transport use:

| MCP core name | CLI |
|---|---|
| `list_teams` | `vibexp team list` |
| `list_projects` | `vibexp project list --team <t>` |
| `search` | `vibexp search "<q>" --team <t> --project <p> [--type memories\|artifacts\|blueprints\|prompts] [--limit N]` |
| `get_resource` (memory) | `vibexp memory get <id> --team <t>` |
| `get_resource` (artifact) | `vibexp artifact get <slug> --team <t> --project <p>` |
| *(no MCP equivalent)* | `vibexp blueprint get <slug>` · `vibexp prompt get <slug>` — full content, CLI only |
| `list_resources` | `vibexp memory list` / `artifact list` / `blueprint list` / `prompt list` (paged) |
| `create_memory` / `update_memory` | `vibexp memory create\|update <id> --body-file <f> [--status <s>]` |
| `create_artifact` / `update_artifact` | `vibexp artifact create\|update <slug> --body-file <f> [--change-summary "…"]` |
| `create_blueprint` / `update_blueprint` | `vibexp blueprint create\|update <slug> --body-file <f>` |
| `create_prompt` / `update_prompt` / `render_prompt` | `vibexp prompt create\|update\|render` |
| `list_feeds` / `list_feed_items` / `get_feed_item` | `vibexp feed list` / `feed items` / `feed get-item <item-id>` |
| `post_to_feed` / `reply_to_feed_item` | `vibexp feed post\|reply --feed <id> --title "…" --body-file <f> --author "<name>"` |
| `link_resources` | `vibexp relations create --from-type <t> --from-id <id> --relation-type <r> --to-type <t> --to-id <id> --origin ai` |
| `list_resource_metadata` | `vibexp metadata` |
| `upload_attachment` / `list_attachments` | `vibexp attachment` |

`vibexp <cmd> --help` is the ground truth for flags — never guess one. For
anything unmapped, `vibexp api` calls any endpoint (last resort).

## CLI usage rules

- **Always pass `--team` and `--project` explicitly** from the resolved scope
  (`resolve-scope.md`). Never rely on the CLI context's default team/project —
  a context configured for another repo is not this repo's scope; the git URL
  decides. Unlike the MCP tools, `--team` requires the team **UUID** (slugs
  are rejected with `team_id must be a valid UUID`) — the scope cache stores
  it.
- **`--format json`** on every call an agent parses; add **`--jq`** to keep
  only the fields the step needs — that trimming is the token win, use it.
  Responses are enveloped by collection name (`{"teams": […]}`,
  `{"projects": […]}`, `{"memories": […]}`, search: `{"results": […]}`), so
  filters start there — e.g. `--jq '.projects[] | {id, git_url}'`, or for
  search triage `--jq '.results[] | {type, id, slug, title, score, excerpt}'`.
- **Bodies via `--body-file`** — write content to a scratch file (or pipe with
  `-`), never inline it as a shell argument.
- **Feed authorship**: always `--author` with the same stable assistant name
  used over MCP (e.g. `Claude Code`) — the CLI's default (`vibexp-cli`) would
  fragment the thread identity.
- **Archive over delete, still**: archiving is `update --status archived`. The
  CLI exposes `delete` subcommands; these skills never use them.
- **Excerpt-vs-get still applies**: `search`/`list` return excerpts; run the
  `get` command before relying on or editing content.

## Per-call fallback

Both transports are the same API, so capability decides, not purity: if the
chosen transport lacks a command or parameter a step needs (an MCP-only filter,
or `blueprint get`/`prompt get`, which are CLI-only), use the other transport
for that one call and continue. If a CLI call fails with an auth error
mid-session, fall back to MCP for the rest of the session and tell the user.
