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
2. Add channel IDs to the allowlist
3. Configure `bindings` to route channels to specific agents
4. Mark one agent as `"default": true` for DMs and unrouted messages
5. Restart the gateway and verify connection

## Part 3 — Update CHANNELS.md

*(Step-by-step to be written)*

Add entries to `CHANNELS.md` mapping each channel ID to its agent and purpose.

## Verification

*(Verification steps to be written)*

---

*This document will be expanded into a complete step-by-step guide with
screenshots when the hands-on lab is developed.*
