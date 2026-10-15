---
name: install
description: Install the official VibeXP CLI and connect it to your organization's VibeXP instance, step by step. Detects what is already set up, installs the right binary for this machine, asks for your instance URL and verifies it, creates a CLI context, signs in via browser OAuth — and when OAuth isn't possible (headless/SSH, no OAuth server, REST not wired for browser tokens) walks you through creating an API key and gives you the exact command to run in your own terminal. Ends with a verified, authenticated CLI that every /vibexp skill can use. Use when the user says "install vibexp", "set up the vibexp cli", "connect vibexp", "log in to vibexp", "vibexp auth", or when another /vibexp skill stopped because no VibeXP connection was found.
argument-hint: "[your VibeXP URL, e.g. https://vibexp.example.com]"
---

# Install: get the VibeXP CLI installed and signed in

Take the user from "nothing installed" (or "installed but not signed in") to a working, authenticated `vibexp` CLI connected to **their organization's** VibeXP instance. VibeXP is self-hosted: every organization runs its own instance, so there is no default URL. The user tells you where theirs is, and you check it before using it.

Instance URL from the user (may be empty, in which case ask in Step 2): $ARGUMENTS

Current state of this machine:

- OS / arch: !`uname -sm 2>/dev/null || echo "(unknown)"`
- CLI: !`command -v vibexp >/dev/null 2>&1 && vibexp version 2>&1 | head -4 || echo "(vibexp not installed)"`
- Contexts: !`vibexp config get-contexts 2>/dev/null || echo "(none)"`
- Signed in: !`vibexp whoami >/dev/null 2>&1 && echo yes || echo no`

## Step 0 — Read the state and skip what is done

The preflight from `${CLAUDE_PLUGIN_ROOT}/references/transport.md` is this skill's starting point, not a gate:

- **Installed and `Signed in: yes`.** Show which context and instance are active (`vibexp config current-context`, `vibexp auth status`). Ask whether the user wants to keep it, sign in again, or add a second organization. Adding one is a new context (Steps 2–5). Nothing else to do otherwise.
- **Installed but not signed in.** Skip to Step 2.
- **Not installed.** Start at Step 1.

This skill is safe to re-run: every step checks before it changes anything.

## Step 1 — Install the CLI

Pick the method for this machine. Say which one you chose and get the user's go-ahead before installing software.

| Machine | Method | Upgrades via |
|---|---|---|
| macOS with Homebrew | `brew install vibexp/tap/vibexp` | `brew upgrade vibexp` |
| Linux, or macOS without Homebrew | release binary (below) | `vibexp update` |
| Windows | `vibexp_<ver>_windows_<arch>.zip` from https://github.com/vibexp/cli/releases/latest, extract, put `vibexp.exe` on `PATH` | `vibexp update` |
| Go toolchain present, user prefers it | `go install github.com/vibexp/cli/cmd/vibexp@latest` | re-run with `@latest` |

**Release binary (Linux / macOS)**: download, verify the checksum, and extract in a fresh temp directory:

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')                    # linux | darwin
case "$(uname -m)" in x86_64|amd64) ARCH=amd64;; aarch64|arm64) ARCH=arm64;; *) echo "unsupported arch"; exit 1;; esac
VERSION=$(curl -fsSL https://api.github.com/repos/vibexp/cli/releases/latest | sed -n 's/.*"tag_name": *"v\([^"]*\)".*/\1/p')
TMP=$(mktemp -d) && cd "$TMP"
base="https://github.com/vibexp/cli/releases/download/v${VERSION}"
curl -fsSLO "$base/vibexp_${VERSION}_${OS}_${ARCH}.tar.gz" && curl -fsSLO "$base/checksums.txt"
if command -v sha256sum >/dev/null; then sha256sum --check --ignore-missing checksums.txt; else shasum -a 256 --check --ignore-missing checksums.txt; fi
tar -xzf "vibexp_${VERSION}_${OS}_${ARCH}.tar.gz" vibexp
```

A failed checksum means STOP. Do not install the binary; report the mismatch.

Install location: prefer a user-writable directory on `PATH` so no `sudo` is needed. Use `install -m 0755 vibexp "$HOME/.local/bin/vibexp"` when `~/.local/bin` is on `PATH`. Otherwise `/usr/local/bin` needs `sudo`, which you can't answer a password prompt for. Ask the user to run it themselves: `! sudo install -m 0755 "$TMP/vibexp" /usr/local/bin/vibexp`.

Verify with `vibexp version`. It must print a version. A new shell may be needed if the install directory was only just added to `PATH`.

## Step 2 — Get and verify the instance URL

Use the URL from the arguments, or ask: *"What's the address of your organization's VibeXP? It's the URL you open in the browser to sign in."* If they don't know it, their VibeXP admin does. Don't guess, and don't reuse a URL from another context unless the user confirms it.

Normalize it to the **origin** (`https://host[:port]`): add `https://` if there's no scheme, and drop any trailing slash and path (`/mcp/v1/common`, `/api/v1`, `/login`, app routes). That origin is the CLI's base URL, the same one that serves the web app, the API (`/api/v1`) and MCP (`/mcp/v1/common`).

Then check it:

```bash
curl -fsS --max-time 10 "$URL/health"                                         # expect {"status":"healthy","sha":"<server version>"}
curl -s -o /dev/null -w '%{http_code}\n' --max-time 10 "$URL/.well-known/oauth-authorization-server"   # 200 → OAuth login available
```

- `/health` healthy → note the server version and continue.
- `/health` fails (DNS, TLS, 404, or HTML instead of JSON) → tell the user exactly what failed. Ask them to re-check the URL, or whether their deployment serves the API on a separate host (the API host is the one that answers `/health`). Don't continue with an unverified URL.
- OAuth metadata 200 → Step 4 tries browser login first. Anything else → no OAuth server; go straight to the API-key path (Step 4b).

## Step 3 — Create the CLI context

A context is a named profile (base URL + stored credential) in `~/.vibexp/config.yaml`. Name it after the instance, e.g. the hostname with dots turned into dashes (`vibexp-acme-com`), or a short org name the user prefers. If a context already points at this URL, reuse it.

```bash
vibexp config set-context <name> --base-url "$URL"
vibexp config use-context <name>       # set-context only auto-activates the very first context
```

Don't set a default `--team`/`--project` here. The /vibexp skills resolve scope per repository (`references/resolve-scope.md`) and always pass both explicitly.

## Step 4 — Sign in

### 4a · Browser OAuth (default)

Skip straight to 4b when the machine has no usable browser: an SSH session (`$SSH_CONNECTION` set), Linux with neither `$DISPLAY` nor `$WAYLAND_DISPLAY`, a container or CI. Also skip it when Step 2 found no OAuth metadata.

Otherwise tell the user a browser window is about to open and they should finish signing in there. Then run:

```bash
vibexp auth login        # OAuth 2.1 + PKCE; opens the browser, waits on a loopback callback
```

Give the call a generous timeout (about 5 minutes) since it blocks until the user finishes. If you can't run it, have the user run `! vibexp auth login`.

- Exit 0 → Step 5.
- Exit 4 with *"no browser available"*, *"this deployment has no OAuth server"*, or *"signed in, but this deployment's REST API is not wired to accept browser-login tokens"* → that is the CLI telling you to use an API key: go to 4b. Explain the reason to the user in one line.
- Scope errors on a deployment with a non-standard authorization server → retry once with `--scope` values from the user's admin (or `VIBEXP_OAUTH_SCOPE`); still failing → 4b.

### 4b · API key (fallback; works on every deployment)

API keys are created in the web app and act as the user, with exactly their team roles. Walk the user through it:

1. Open `<URL>` in the browser → **Settings** → **API Keys** → **Create New API Key**.
2. Name it after this machine (e.g. `CLI – <hostname>`), and under integrations select **VibeXP CLI** only (least privilege; the legacy *MCP Server* integration is not needed).
3. Click **Create** and copy the key right away. It's shown only once and starts with `vxk_`.

Then give them the command to run **in their own terminal, not in this chat**:

```bash
vibexp auth login --context <name> --with-api-key
```

It reads the key from a hidden prompt, so it never lands in shell history, the transcript, or a process list. Don't run this yourself: your shell has no TTY, so it would need the key on stdin, which means the key passing through the conversation. Wait for the user to say it's done, then go to Step 5.

For CI or a headless box where an interactive prompt isn't possible, there's an alternative. Set `VIBEXP_API_KEY` from the environment's secret store. It takes precedence over stored credentials, so make sure it doesn't silently shadow the login on a developer machine.

**Key safety.** Never ask the user to paste the key into the chat. Never put it in a command argument, a file in the repo, or agent memory. If they paste it anyway, don't use it: tell them it's now in the transcript and should be revoked (**Settings → API Keys → Revoke**) and replaced with a new one.

## Step 5 — Verify end to end

```bash
vibexp whoami                     # exit 0 = authenticated (this is the probe every /vibexp skill uses)
vibexp auth status                # method (oauth | api key) + identity; never prints the secret
vibexp version                    # CLI version + server version and health
vibexp team list --format json --jq '.teams[] | {id, slug, name}'
```

- `whoami` non-zero → re-check Step 4 (an invalid or expired key exits 4 with the server's error and a request id; show it).
- Zero teams → the account works but has no team yet. The user must be invited, or create a team in the web app, before the knowledge skills have anywhere to read or write.
- `VIBEXP_API_KEY` set in the environment → `auth status` reflects it, not the stored login. Point that out if it isn't what the user intended.

## Step 6 — Optional: MCP for other AI tools

The /vibexp skills work over the CLI alone. The MCP connection matters for tools that can't run a CLI, or for exposing team prompts as slash commands (`/vibexp:promptify`). Offer it, don't push it:

```bash
claude mcp add --transport http vibexp "$URL/mcp/v1/common"
```

MCP signs in with **OAuth in the browser, never an API key** (the endpoint rejects keys with 401). After adding, restart Claude Code or run `/mcp` to complete the login.

## Step 7 — Report and hand off

Summarize what was done: install method and CLI version, instance URL and server version, context name, auth method, identity, and the teams visible. Then point at what comes next:

- `/vibexp:onboard` for a repo that's new to VibeXP; `/vibexp:prime` to start a task with the team's knowledge.
- `/vibexp:help <question>` for anything about using VibeXP.
- Several organizations → one context each; switch with `vibexp config use-context <name>`. The skills use whichever context is active.

## Conventions (apply throughout)

- Never handle the API key: no pasting into chat, no command arguments, no files, no memory. The user types it into the CLI's hidden prompt.
- Get the user's go-ahead before installing software or anything needing `sudo`. Verify downloads against `checksums.txt`.
- `vibexp <cmd> --help` is the ground truth for flags; never guess one. Exit code 4 is an auth error, 2 is a usage error.
- Probe auth with `vibexp whoami`, not `auth status`: `auth status` exits 0 even when logged out.
- This skill configures the local machine only. It never writes to the VibeXP knowledge base.
- Docs: API keys https://vibexp.io/docs/user-guide/integrations/api-keys/ · MCP https://vibexp.io/docs/user-guide/mcp-server/ · CLI https://github.com/vibexp/cli
