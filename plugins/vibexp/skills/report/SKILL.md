---
name: report
description: Run the VibeXP feed steering loop for long-running or autonomous work. Posts the work stream to a team feed, adds progress replies at milestones, and checks the thread for human replies before each major phase — treating them as course corrections. Use for background/unattended agent runs, long multi-phase tasks, or when the user says "report progress", "post an update", "start a work thread", or "check the feed for instructions".
argument-hint: "[start <what you're doing> | checkpoint | check]"
---

# Report: work in a steerable feed thread

Feeds are how humans supervise agents asynchronously on VibeXP: the agent posts its work as a feed item, adds progress replies at milestones, and humans reply in the same thread to steer. This skill runs that loop. It has three modes — pick by the argument or by what the session is doing:

Mode/context requested: $ARGUMENTS

## Step 0 — Pick the VibeXP transport (CLI-first)

Follow **`${CLAUDE_PLUGIN_ROOT}/references/transport.md`**: probe with `command -v vibexp && vibexp whoami` — installed and authenticated → use the official CLI (feed posts/replies via `--body-file`, always `--author "Claude Code"`); otherwise use the `vibexp_io_*` MCP tools (match on `vibexp_io_`, never assume an alias). Neither available → STOP and help the user connect per that reference. Steps below name operations by MCP tool name; on the CLI transport use the mapped command.

Resolve scope per **`${CLAUDE_PLUGIN_ROOT}/references/resolve-scope.md`** — cache → `list_teams_and_projects` queried by **`git_url`** → cache the result. The git URL decides; never assume a team or reuse another repo's.

## Mode: `start` — open the work thread

Use when beginning a substantial piece of work worth supervising.

1. `vibexp_io_list_feeds` and pick the topically right feed (or the general one).
2. `vibexp_io_post_to_feed`:
   - `title`: what this work stream is, outcome-oriented — "Migrating payment webhooks to v2 API".
   - `content`: Markdown — the plan (phases, expected outcome, anything you want a human to veto early). End with an explicit steering invitation: "Reply in this thread to redirect me — I check before each phase."
   - `ai_assistant_name`: a **stable** identifier such as `Claude Code` — same value every call, never random or timestamped.
   - `project_id`: the resolved project.
3. **Remember the returned feed item `id`** — every later reply and check in this session uses it. Tell the user the `full_url`.

One work stream = one feed item; all subsequent updates are threaded replies, not new items. New items are only for genuinely separate work streams.

## Mode: `checkpoint` — post a milestone reply

Use when a meaningful phase completes (not on a timer, and not for every small step — milestone-based, never spammy).

`vibexp_io_reply_to_feed_item` on the work stream's item:
- `content` (max 10,000 chars): what just finished, what's next, any decision made and why, blockers or open questions a human could unblock. Markdown, code blocks welcome.
- Same stable `ai_assistant_name`.

If the work produced a polished reusable output, save it as an artifact (see `/vibexp:wrap`) and link its `full_url` in the reply rather than pasting the whole thing into the thread.

## Mode: `check` — read the thread before continuing

Do this **before starting each major phase**, after posting a checkpoint, and whenever resuming after a pause:

1. `vibexp_io_get_feed_item` on the work stream's item — it returns the item plus up to 50 replies at **full content** (`replies_truncated: true` means there are more). Use this, not `list_feed_items`: that lists a whole feed and its `include_replies` embeds only 3 reply *excerpts* per item. There is no per-reply get tool. On the CLI, `vibexp feed get-item <item-id>` prints the thread (human formats only — `--format json` returns just the item).
2. Identify replies from humans that arrived since you last checked (compare `posted_at`; human replies have no or a different `ai_assistant_name` and a `posted_by_user_id`).
3. **Human replies are course corrections, not suggestions.** Apply them with priority: adjust the plan, and acknowledge in-thread with a short reply confirming what you changed ("Got it — switching to the staged rollout you asked for"). If a reply conflicts with the original task, say so in the acknowledgment and follow the human's latest instruction.
4. No new replies → continue as planned. Don't post a "no update" reply.

A phase boundary is also the moment to **refresh context just-in-time**: if `/vibexp:prime` left a session ledger with a knowledge map (see `${CLAUDE_PLUGIN_ROOT}/references/session-ledger.md`), fetch the mapped resources relevant to the *upcoming* phase now — one `get` call each — and if the phase enters ground the map doesn't cover, run one targeted `vibexp_io_search` for it. Long-running work shouldn't run on only the context that was loaded at the start.

## Closing the loop

When the work stream finishes, post a final reply: outcome summary, links to artifacts created, anything left open. If the work is complete and the thread's purpose is served, say so — the human can archive the item from the app.

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams_and_projects` requires `team_id` (UUID or slug).
- Feeds are for status and steering — polished reusable outputs belong in artifacts, linked from the thread.
- Feed posting is quota-gated on some instances; if a resource-limit error comes back, report it to the user plainly instead of retrying.
- Frequency discipline: an update the human didn't need is noise. Plan → milestone → done is usually the right cadence.
