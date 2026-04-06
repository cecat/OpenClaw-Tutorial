# Slack Integration — Step-by-Step Setup Guide

*Status: stub — to be completed as part of the hands-on lab development.*

This guide covers the complete process of connecting a Slack workspace to an
OpenClaw deployment. It is referenced from Module 8 of the tutorial outline.

## Prerequisites

- OpenClaw gateway running and accessible via Tailscale (Module 3 complete)
- A Slack workspace where you have permission to install apps
- `openclaw.json` accessible for editing (via the dashboard or extract script)

## Part 1 — Slack Side: Create and Configure the App

*(Step-by-step to be written — will include screenshots of api.slack.com/apps)*

1. Go to api.slack.com/apps → Create New App → From Scratch
2. Enable Socket Mode (under Settings → Socket Mode)
3. Under OAuth & Permissions → Bot Token Scopes, add:
   `channels:history`, `chat:write`, `users:read`, `groups:history`
   **Do not add `assistant:write`**
4. Install the app to your workspace → copy the Bot Token (`xoxb-...`)
5. Under Basic Information → App-Level Tokens → Generate a token with
   `connections:write` scope → copy the App Token (`xapp-...`)
6. Note the channel IDs of any channels where the bot should respond
   (right-click a channel in Slack → View channel details → copy the ID)

## Part 2 — OpenClaw Side: Configure the Gateway

*(Step-by-step to be written)*

1. Add tokens to `openclaw.json` under `channels.slack`
2. Add channels to `config.yaml` — `apply-config.sh` automatically syncs both
   `bindings[]` (which agent handles a channel) and `channels.slack.channels`
   (the event delivery allowlist). Both must include a channel for it to work.
   Do not edit either list by hand; let `apply-config.sh` manage them.
3. Mark one agent as `"default": true` in `openclaw.json` for DMs and unrouted messages
4. Run `python3 apply-config.sh` and verify the gateway restarts cleanly

## Part 3 — Update CHANNELS.md

*(Step-by-step to be written)*

`CHANNELS.md` is a quick-reference table in the agent workspace listing each channel, its ID, and its assigned agent. The authoritative source is `config.yaml` — after adding a channel there and running `apply-config.sh`, update `CHANNELS.md` to match. The agent uses `CHANNELS.md` to know where to direct Slack messages; keeping it in sync with `config.yaml` avoids routing confusion.

## Part 4 — Agent-Side Configuration for Outbound Posting

Connecting Slack at the gateway level (Parts 1–2) lets agents *receive* messages
and *reply* in the same session. It does not enable agents to proactively post to
other channels or send DMs. That requires three additions to each agent workspace.

### The core limitation: `sessions_send` is a reply mechanism

OpenClaw's `sessions_send` tool routes a message back to whoever is currently in
the open session. It works for replies. It **cannot**:
- Post to a channel the agent is not currently in a session with
- Initiate any post when no session is active (heartbeats, scheduled tasks)
- DM another Slack app/bot

The failure mode is silent — the agent reports no error, believes the send
succeeded, and nothing appears in the target channel. An agent whose
`HEARTBEAT.md` says to handle `SLACK_POST` tasks `via sessions_send` will
silently fail on every scheduled post.

### 4.1 Add channel IDs to PATHS.md

Add a Slack section to each agent's `PATHS.md` listing every channel the agent
may ever post to. Without this, the agent has no reliable target reference.

```markdown
## Slack Workflow
- `/shared/slack/outbox/` — Slack post queue (write JSON here to send a message)
- `/shared/slack/sent/` — sent posts archive

Slack posts are delivered by `send-slack-posts.sh` (host cron, every 5 min).
Use the outbox for any proactive post — channel or DM — that is not a direct
reply in your current conversation. See `runbooks/RUNBOOK_SLACK_POST.md`.

### Channel IDs
- `C0AMBT2GD97` — #agent-luoji
- `C0AMYPF9NDN` — #agent-cecat
- `C0AQH73GEHM` — #claws (shared between agents)
- `C0AJ1EL2KJ5` — #agent-chattpc26

### User IDs (for DMs — use as `channel` value in outbox JSON)
- `U05H8JM8NFQ` — Charlie
```

### 4.2 Fix HEARTBEAT.md SLACK_POST and SLACK_DM entries

In the READY task dispatch table, replace `via sessions_send` with the outbox
reference for both `SLACK_DM` and `SLACK_POST` entries:

```markdown
| `SLACK_DM \| <user_id> \| <message>` | Write to Slack outbox — see `runbooks/RUNBOOK_SLACK_POST.md`. Use `<user_id>` as the `channel` value. |
| `SLACK_POST \| <channel_id> \| <message>` | Write to Slack outbox — see `runbooks/RUNBOOK_SLACK_POST.md`. Use `<channel_id>` as the `channel` value. |
```

`sessions_send` remains correct for one case: direct replies to the person
currently messaging the agent in the current session.

### 4.3 Add RUNBOOK_SLACK_POST.md to each agent's runbooks/

Create `runbooks/RUNBOOK_SLACK_POST.md` in each agent workspace with the exact
outbox JSON pattern. Describing the pattern in prose elsewhere is not sufficient
— agents executing a READY task follow a recipe, not a description.

```markdown
# RUNBOOK_SLACK_POST.md — Post to Slack (Channel or DM)

Use for any Slack message that is not a direct reply in your current
conversation. Channel IDs and user IDs are in PATHS.md.

Delivery: host cron sends within ~5 minutes of writing the outbox file.

## Pattern

    exec: python3 -c "
    import json, time, pathlib
    msg = {
      'channel': 'CHANNEL_OR_USER_ID',
      'text': 'Your message here',
      'requested_by': 'AGENT_ID',
      'requested_at': __import__('datetime').datetime.utcnow().isoformat() + 'Z',
      'status': 'pending'
    }
    pathlib.Path(f'/shared/slack/outbox/{int(time.time())}-AGENT_ID-post.json').write_text(json.dumps(msg))
    "

- For channel posts: use the channel ID as `channel`
- For DMs to a user: use the user's Slack ID as `channel`
- For @mentions in text: use `<@USER_ID>` syntax
```

### 4.4 Inter-agent messaging

Agents cannot DM each other via Slack's bot DM mechanism. To have one agent
reach another:

1. Post via the outbox to a channel that both agents monitor (a shared channel
   bound to both in `config.yaml`, or the target agent's dedicated channel)
2. Include the target agent's name in the text so it recognizes the message

With ACP dispatch disabled (recommended conservative setting), this Slack-mediated
channel post is the only inter-agent communication path.

## Verification

*(Verification steps to be written)*

---

*This document will be expanded into a complete step-by-step guide with
screenshots when the hands-on lab is developed.*
