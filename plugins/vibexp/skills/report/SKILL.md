---
name: report
description: Run the VibeXP feed steering loop for long-running or autonomous work. Posts the work stream to a team feed, adds progress replies at milestones, and checks the thread for human replies before each major phase — treating them as course corrections. Use for background/unattended agent runs, long multi-phase tasks, or when the user says "report progress", "post an update", "start a work thread", or "check the feed for instructions".
argument-hint: "[start <what you're doing> | checkpoint | check]"
---

# Report: work in a steerable feed thread

Feeds are how humans supervise agents asynchronously on VibeXP: the agent posts its work as a feed item, adds progress replies at milestones, and humans reply in the same thread to steer. This skill runs that loop. It has three modes — pick by the argument or by what the session is doing:

Mode/context requested: $ARGUMENTS

## Step 0 — Check the VibeXP MCP connection

VibeXP tools are named `vibexp_io_*` (prefixed with the user's MCP server alias — match on `vibexp_io_`, never assume an alias). If none are available, STOP and help the user connect:

```
claude mcp add --transport http vibexp https://<your-vibexp-host>/mcp/v1/common
```

(Hosted instance: `https://connect.vibexp.io/mcp/v1/common`; self-hosters use their own origin. OAuth in the browser, no API key. Docs: https://docs.vibexp.io)

Resolve team and project as usual: `vibexp_io_list_teams` (one → use it; several → match this repo's git remote or ask once), `vibexp_io_list_projects` matched against the git remote.

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

1. `vibexp_io_list_feed_item_replies` on the work stream's item (replies are truncated to ~300 chars by default; use `full_details: true`, or `vibexp_io_get_feed_item_reply` for a specific one).
2. Identify replies from humans that arrived since you last checked (compare `posted_at`; human replies have no or a different `ai_assistant_name` and a `posted_by_user_id`).
3. **Human replies are course corrections, not suggestions.** Apply them with priority: adjust the plan, and acknowledge in-thread with a short reply confirming what you changed ("Got it — switching to the staged rollout you asked for"). If a reply conflicts with the original task, say so in the acknowledgment and follow the human's latest instruction.
4. No new replies → continue as planned. Don't post a "no update" reply.

## Closing the loop

When the work stream finishes, post a final reply: outcome summary, links to artifacts created, anything left open. If the work is complete and the thread's purpose is served, say so — the human can archive the item from the app.

## Conventions (apply throughout)

- Every tool except `vibexp_io_get_user` and `vibexp_io_list_teams` requires `team_id` (UUID or slug).
- Feeds are for status and steering — polished reusable outputs belong in artifacts, linked from the thread.
- Feed posting is quota-gated on some instances; if a resource-limit error comes back, report it to the user plainly instead of retrying.
- Frequency discipline: an update the human didn't need is noise. Plan → milestone → done is usually the right cadence.
