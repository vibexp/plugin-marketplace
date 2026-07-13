# VibeXP Claude Code Plugins

Official [Claude Code](https://code.claude.com) plugin marketplace for [VibeXP](https://vibexp.io) — the shared knowledge base your AI builds on. Free, open source, self-hostable.

These plugins encode VibeXP best practices as installable skills, built around the core loop:

> **Before a task** your AI reads the relevant knowledge → **as it works** it applies it → **after the work** it writes learnings and outputs back → **every session after** starts smarter.

## Prerequisites

A VibeXP instance connected to Claude Code over MCP (OAuth in the browser, no API key):

```sh
claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common
```

Use `https://connect.vibexp.io/mcp/v1/common` for the hosted instance, or your own deployment's URL if you [self-host](https://docs.vibexp.io). New to VibeXP? It's one `docker compose up -d` away: [github.com/vibexp/vibexp](https://github.com/vibexp/vibexp).

## Install

```sh
claude plugin marketplace add vibexp/plugin-marketplace
```

Then inside Claude Code:

```
/plugin install vibexp@vibexp
```

## Plugins

### `vibexp` — the knowledge loop

| Skill | What it does |
|---|---|
| `/vibexp:prime` | Start a task with your team's knowledge: retrieves the relevant memories, blueprints, artifacts, and recent feed activity for the current project (auto-detected from your git remote) and briefs you before working. |
| `/vibexp:wrap` | End a session by writing back: durable learnings saved as memories (deduplicated against existing ones), polished outputs saved as versioned artifacts, and a status update posted to your team feed. |
| `/vibexp:consolidate` | Garden the knowledge base: merge near-duplicate active memories, correct or archive stale and contradicted ones, and promote recurring feed lessons into durable memory. |
| `/vibexp:report` | Run long or autonomous work as a steerable feed thread: post the plan, reply at milestones, and check the thread for human replies before each phase — treating them as course corrections. |
| `/vibexp:promptify` | Turn a prompt that worked into a team asset: generalize it with `{{variables}}`, factor boilerplate into `@slug` base prompts, and publish it — optionally MCP-exposed as a native slash command in every teammate's tool. |
| `/vibexp:onboard` | Bootstrap a new project's knowledge base: import the repo's AI config (CLAUDE.md, .cursorrules, AGENTS.md, …) as blueprints and seed a few high-value memories, deduplicating against anything already there. |

## Contributing

Issues and PRs welcome. Each plugin lives under `plugins/<name>/` with its skills in `skills/<skill>/SKILL.md`. Validate changes with:

```sh
claude plugin validate .
```

## License

[MIT](./LICENSE)
