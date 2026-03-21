# Google Integration — Step-by-Step Setup Guide

*Status: stub — to be completed as part of the hands-on lab development.*

This guide covers connecting Google services (Gmail, Contacts, Drive, Sheets) to
an OpenClaw agent workspace. It is referenced from Module 8 of the tutorial outline.

We use two tools for Google integration, for different purposes. This guide covers
both; you only need the sections relevant to your use case.

| Tool | Use for | When to use |
|---|---|---|
| `gsuite-mcp` | Gmail, Google Contacts | You need to read/send email or manage contacts |
| `gog` CLI | Google Sheets, Docs, Drive | You need to read/write spreadsheets or documents |

See `GOG-Integration.md` for gog-specific setup. This guide covers gsuite-mcp and
the Gmail/Contacts workflow.

---

## Prerequisites

- OpenClaw gateway running and accessible (Module 3 complete)
- A Google account (personal or Workspace)
- Node.js installed on the host (required for gsuite-mcp)
- `openclaw.json` accessible for editing

---

## Part 1 — Google Cloud Project Setup

*(Step-by-step to be written — will include screenshots of console.cloud.google.com)*

1. Go to [console.cloud.google.com](https://console.cloud.google.com) and create
   a new project (or select an existing one)
2. Enable the APIs you need:
   - **Gmail API** — for reading and sending email
   - **People API** — for Google Contacts (read and write)
   - **Google Drive API** — if using gog for Drive access
   - **Google Sheets API** — if using gog for Sheets access
3. Configure the OAuth consent screen:
   - User type: **Internal** if you have Google Workspace; **External** otherwise
   - Add the scopes corresponding to the APIs you enabled
   - For External apps, add your agent email as a test user during development

   **⚠️ HANDS-ON TODO:** Expand with screenshots of the consent screen setup.

   **Critical — publish the app before deploying to production:** Apps left in
   "Testing" mode have refresh tokens that expire after 7 days. For a daily cron
   job this means re-authenticating weekly. Publish the app (OAuth consent screen
   → Publish App) so refresh tokens are long-lived. Gmail and Contacts are
   sensitive scopes — Google will show an "unverified app" warning during the
   one-time auth flow; click through it. Formal Google verification is only
   required if external users (outside your own account) need to authorize the app.
4. Create OAuth 2.0 credentials:
   - Application type: **Desktop app**
   - Download the `credentials.json` file
   - Save it to `~/.config/gsuite-mcp/` on the host

---

## Part 2 — gsuite-mcp Authentication

*(Step-by-step to be written)*

1. Install gsuite-mcp on the host:
   ```bash
   npm install -g @markusp/mcp-gsuite
   ```
2. Run the browser-based OAuth flow (one-time):
   ```bash
   gsuite-mcp auth login
   ```
   A browser window will open. Log in to the Google account the agent should use.
   After authorization, gsuite-mcp writes `token.json` to
   `~/.local/share/gsuite-mcp/`.
3. Verify the token file exists and is not empty:
   ```bash
   ls -la ~/.local/share/gsuite-mcp/token.json
   ```

**Note:** Use a dedicated Google account for the agent, not your personal account.
This limits the blast radius if credentials are compromised and keeps agent email
activity separate from your personal email.

---

## Part 3 — Sandbox Bind Mounts

The agent scripts run inside an ephemeral sandbox container. They need access to
the OAuth token and credentials files, which live on the host. Configure these
bind mounts in `openclaw.json` under the relevant agent's `sandbox.docker.binds`:

```json
"binds": [
  "/home/YOUR_USER/.config/gsuite-mcp:/tmp/.config/gsuite-mcp:ro",
  "/home/YOUR_USER/.local/share/gsuite-mcp:/tmp/.local/share/gsuite-mcp:rw",
  "/path/to/agents/gmail-agent/scripts:/scripts:ro"
]
```

- `credentials.json` mount: **read-only** — the container reads but never writes credentials
- `token.json` mount: **read-write** — the container must be able to refresh tokens
- `scripts/` mount: **read-only** — agent scripts are not modified at runtime

Apply the change with `./apply-config.sh` and restart the gateway.

---

## Part 4 — Verify the Integration

*(Verification steps to be written)*

1. Send a test message to the agent asking it to run `gmail_api.py` to list recent inbox messages
2. Confirm the agent returns message subjects and senders
3. Confirm the token refresh worked by checking the `token.json` modification timestamp

---

## Part 5 — Writing Style and Contacts Bootstrap (Optional)

*(To be written)*

The Gmail agent can analyze your sent mail to learn your writing style and populate
`MEMORY.md` with a style guide. It can also harvest sender/recipient addresses from
your sent mail to pre-populate Google Contacts. See the runbooks in `gmail-agent/runbooks/`
for details.

---

*This document will be expanded into a complete step-by-step guide with screenshots
when the hands-on lab is developed.*
